# Paper 04 — JIT–GC Co-Design (`JIT-GC Co-Design`)

Techniques covered:

1. **JIT-Synthesized Destructors** — `JITDestruct` (with speculative escape guards and handoffs)
2. **Speculative Cache-Line Sweeping** — `CacheSweep` (64B-aligned side-table bitmaps, unrolled JIT tracers, `PREFETCH`)
3. **Stall-Cycle Harvesting** — `StallHarvest` (PMU telemetry + JIT control-flow prediction → bounded `gc_micro_step()`)

Claim tags: **[A]** architectural · **[H]** hypothesis · **[M]** measured (none).

---

## 1. Scope and Setting

### 1.1 What this layer is for
`LastUseLifetime` (Paper 01) frees what the compiler can prove dead. What remains are objects that **escape** the proof: stored in shared structures, passed through unknown code, or crossing threads. A runtime collector, the **residual collector**, handles these. The JIT knows types, field offsets, escape behavior, and hot control flow, which is exactly what that collector needs.

### 1.2 The residual collector (baseline assumed by this paper)
- Operates on the `SegGenPool` heap; precise (type maps from the compiler/JIT), not conservative.
- Marking uses **side-table bitmaps** (§3), not object headers.
- Concurrent marking uses a **snapshot-at-the-beginning (SATB)** deletion write barrier with per-thread buffers; the barrier is elided on stores into objects known to be non-escaped or scalar-only (§2.6). **[A]**
- Sweeping is metadata-only (bitmaps and generation arrays), never payload traversal (§3.6).
- Relocation is delegated to `TileCompact`; tiles being compacted are excluded from concurrent marking, or their mark bits move with the objects (§3.8).

### 1.3 Dependencies
| Technique | Needs | Can run without |
|-----------|-------|-----------------|
| `JITDestruct` | A JIT or AOT compiler with type/escape info, safepoint maps, a code cache or link-time specialization | `CacheSweep`, `StallHarvest` |
| `CacheSweep` | Type maps; side-table bitmaps; (optional) JIT for unrolled tracers: AOT-generated tracers work too | `StallHarvest` |
| `StallHarvest` | A collector with *bounded steps* (e.g., `CacheSweep`); a poll mechanism; PMU access **optional** (degrades to prediction-only) | PMU |

---

## 2. `JITDestruct` — JIT-Synthesized Destructors

### 2.1 Purpose
Replace generic "visit every object, decode its type, free it" machinery with **per-type, per-site compiled reclaim code**, and make non-escape *speculation* safe via cheap guards and a handoff to the residual collector.

### 2.2 Inputs
- Typed IR; field layout; ownership edges between objects (`owned` fields: those whose referents die with the parent).
- Profile: allocation-site hotness, observed escape behavior.
- Escape analysis result per site: `NoEscape`, `ArgEscape`, `GlobalEscape`, plus a *speculative* upgrade: `SpecNoEscape` (assumed non-escaping based on inlining and type feedback).

### 2.3 State and metadata

**Per-object ownership state, in the side table** (one byte per granule-start, or two bits in the start bitmap):

| Bit | Meaning |
|-----|---------|
| `OWNED` | Thread-local, compiler-managed: release is the compiled code's job |
| `TRACED` | Shared/escaped: the residual collector is responsible |

Allocation under `SpecNoEscape` sets `OWNED` in the allocating thread's bump segment (Paper 01 §2.4). **[A]**

**Per-type reclaim routine** `reclaim_T(obj)` and **per-type tracer** `trace_T(obj, wl)` (the latter belongs to `CacheSweep`), generated once per type, shared by all sites. Small types may inline.

### 2.4 Synthesized destructor: how it works

For type `T` with fields `f1..fn`:

```
reclaim_T(obj):                       // generated per type
    for each owned field f of managed type U:   // statically known offsets, no type-map interpretation
        if obj.f != null: reclaim_U(obj.f)       // or enqueue to an iterative worklist (below)
    // POD fields: nothing
    seg_dec(obj, sizeof(T))                      // SegDec: metadata only
```

- **No generic traversal**: field offsets and types are compile-time constants; branches on null only.
- **Cascade bounding.** A naive recursive drop of a long list/deep tree causes stack depth and a latency spike. The generated code for self-referential types uses an **iterative worklist with a budget**: process up to `N` nodes, and enqueue the remainder to the `MasterSlaveMM`/background queue (`Defer`). This replaces the "drop cascade" tail of scope-based RAII (THEORY §1.3). **[A]**
- **POD fast path.** For types with no managed fields, the routine is a single `SegDec`, or nothing at all if the whole segment dies together.
- **Effectful types** (observable destructor effects) are excluded and use scope semantics (Paper 01 §3.8).

### 2.5 Speculative escape guards and handoffs

The JIT compiles a method assuming `SpecNoEscape` for some allocations (e.g., the callee that would let the object escape was not observed in profile, or was devirtualized by type feedback). Two layers of safety:

**Layer 1 — Compilation-time dependencies (class-hierarchy / type-feedback assumptions).**
When the assumption depends on global facts ("only `ImplA` implements `foo`"), the compiled code registers a **dependency**. If class loading invalidates the fact, the dependent code is deoptimized at the next safepoint (standard deopt machinery: frames are rewritten to interpreter state; scalar-replaced or stack-assigned objects are **materialized** as heap objects with `TRACED` set). **[A]**

**Layer 2 — Local escape guards and handoff.**
At each program point where the object *might* escape under a path the compiler did not prove away (store into a heap object field, call through an unprofiled target, publish to another thread):

```
guard_escape(obj):
    // cold path; the common case does not reach it
    handoff(obj)

handoff(obj):                       // owner thread only; obj not yet visible to any other thread
    side_table[obj].state: OWNED → TRACED        // plain byte store; owner-only
    counters[site].handoffs++                    // feedback for recompilation
    // then the escaping store/publish proceeds, with release ordering
```

The object **does not move and is not copied**: only its responsibility changes. The residual collector will find it through the normal graph (it is reachable from wherever it escaped to) and consider it in marking. **[A]**

**Release site under speculation:**
```
release(obj):                        // at last use (from LastUseLifetime)
    if side_table[obj].state == OWNED:  seg_dec(obj)     // compiler-managed free
    else:                               /* TRACED: collector owns it; do nothing */
```

One side-table byte read; the line is cached (the owner just touched the object). **[A]**

### 2.6 Why handoff is race-free (ordering argument)

1. `OWNED` objects are reachable only from the owner thread's locals and from other `OWNED` objects (otherwise they would have escaped).
2. `handoff` executes **before** the instruction that makes the pointer visible elsewhere, on the owner thread: the side-table store is sequenced-before the publishing release-store. Any thread that obtains the pointer through that publishing store has acquire semantics and therefore sees `TRACED`. **[A]**
3. The residual collector only considers `TRACED` objects as live-by-reachability; `OWNED` objects are never swept by it (their bits are excluded from its "allocated but unmarked" set), so it cannot free something the owner still uses. When the owner `seg_dec`s an `OWNED` object, it also clears that object's alloc bit.
4. The SATB barrier is **not** needed for stores into `OWNED` objects, because they are invisible to the collector; this is the barrier-elision of §1.2. After handoff the barrier applies as usual. A conservative compiler rule: any store into an object that *may* be `TRACED` keeps the barrier. **[A]**

### 2.7 Feedback and recompilation
`counters[site].handoffs / counters[site].allocs` above a threshold (e.g., > 5–10%) means the speculation is bad: recompile that site as `TRACED`-from-birth (allocate normally, no guards, no `OWNED`). This bounds the guard overhead and avoids a "speculation storm". Hysteresis prevents flip-flopping between modes. **[A]**

### 2.8 Failure and edge cases

| Case | Behavior |
|------|----------|
| **Reflection, serialization, debuggers, JVMTI-like agents reading an `OWNED` object** | Treated as escape: such access paths call `handoff` first |
| **Exceptions carrying the object** | Landing-pad release respects state; throw is an escape if the payload is `OWNED` and passed in the exception object |
| **Deopt with scalar-replaced objects** | Materialize + set `TRACED` (or `OWNED` if the deoptimized frame remains the sole owner) |
| **Thread migration of coroutines** | Frame move is an escape for objects it holds |
| **Handoff storm** | §2.7 recompilation |
| **Code-cache growth** | One routine per type (not per site); inline only below size threshold; LRU eviction of cold routines |
| **Type redefinition / schema change** | Invalidate routines; deopt dependents |
| **Finalizers/destructors with side effects** | Excluded from synthesis (use scope/`Scoped` semantics) |
| **Missing side-table update on a path (compiler bug)** | Debug mode: poison and verify `OWNED` at safepoints; `TRACED` objects never freed by compiled code |

### 2.9 Trade-offs

| Benefit | Cost |
|---------|------|
| No generic traversal for the common case; no header bits **[A]** | Code cache growth; per-type code |
| Barrier elision for `OWNED` stores **[A]** | Guard/handoff cost at cold escape points; side-table byte test on release |
| Bounded cascades (iterative + budget) **[A]** | Deferred remainder holds memory briefly |
| Handoff without copying **[A]** | Mixed-ownership segments: an escaped object keeps its (bump) segment alive until the collector or `TileCompact` handles it |

---

## 3. `CacheSweep` — Speculative Cache-Line Sweeping

### 3.1 Purpose
Make the residual collector's mark and sweep phases **cache-line-efficient** and **latency-tolerant**: decide many objects per cache line of metadata, never dirty object lines, and overlap pointer-chase misses with prefetch.

### 3.2 Inputs
- Heap layout from `SegGenPool` (segments, granule size).
- Per-type pointer maps (offsets of reference fields) from the compiler/JIT.
- Roots: stacks (via safepoint maps), registers, globals, handle tables.
- Work budget per step.

### 3.3 State: 64B-aligned side-table bitmaps

Define **granule** `g` = minimum object alignment (default 16 B). Bitmaps:

| Bitmap | Bit `i` means | Density |
|--------|---------------|---------|
| `start` | An object starts at granule `i` | 1 bit per 16 B = 1/128 of heap (~0.78%) |
| `mark` | Object starting at `i` is marked live | same |
| (optional) `pin`, `owned`/`traced` bits | per §2.3 | same |

Alignment and coverage arithmetic: **one 64-byte cache line = 512 bits = 512 granules = 8 KiB of heap** (with `g = 16`). Bitmaps for each segment are allocated 64 B-aligned, so a bitmap "line" corresponds to an 8 KiB *block* and loads in one line fill. Bitmaps of adjacent blocks are adjacent in memory, so a streaming sweep is a sequential read that the hardware prefetcher handles. **[A]**

Per-block summary (kept in a separate small array): `live_count`, `free_runs`, `needs_sweep`.

### 3.4 Marking

```
mark_step(budget):
    while budget and ring.not_empty():
        obj = ring.pop_oldest()                    // prefetch queue (below)
        trace_T(obj)                               // unrolled, per-type, JIT/AOT-generated
```

**Unrolled JIT tracer for type `T`** with reference fields at offsets `o1, o2, o3`:

```
trace_T(obj):
    c1 = load(obj + o1); c2 = load(obj + o2); c3 = load(obj + o3)    // independent loads, issued together
    visit(c1); visit(c2); visit(c3)                                   // unrolled; no loop, no type-map decode

visit(c):
    if c == null: return
    bm = bitmap_line_for(c)                                           // 1/128 density → usually cache-resident
    if test_bit(bm, idx(c)): return                                   // already marked: no store, no object touch
    set_bit_atomic(bm, idx(c))                                        // fetch_or on a 64-bit word
    PREFETCH(c)                                                       // object line
    ring.push(c)                                                      // enqueue for later tracing
```

Key points:
1. **Test before set**: checking the mark bit avoids a write (and the associated coherence traffic) for already-marked objects. **[A]**
2. **The object itself is not written.** Marking sets a bit in the side table; the object's cache line stays clean/shared, so other cores' cached copies are not invalidated. **[A]**
3. **Prefetch-on-grey with a FIFO ring**: when an object is discovered ("greyed"), issue `PREFETCH` for its lines and push to a small FIFO of depth `D`. Tracing pops the **oldest** entry, whose line has had `D` other discoveries' worth of time to arrive. This turns a dependent chain into `D` overlapped misses, so the cost per object tends from `L_miss` toward `L_miss / min(D, MLP)` (Paper 00 §2.1). **[A]**
4. **Choosing `D`**: `D ≈ L_miss / t_trace` where `t_trace` is the time to trace one object. Typically 8–16; bounded by the core's line-fill buffers (≈10–16 outstanding L1 misses on common x86 cores). Beyond that, extra prefetches are dropped or queue-limited. Tune per microarchitecture. `OPEN:` adaptive `D` from PMU feedback.

Prior art for prefetch-on-grey in mark-sweep GC exists (e.g., work by Boehm; Cher, Hosking, and Vijaykumar; Garner, Blackburn, and Frampton, to the best of our recollection; verify citations before publication). The contribution here is the **combination** with per-type unrolled tracers, side-table-only marking, and the micro-step scheduler (§4). **[A]**

### 3.5 "Speculative"
Speculation occurs at two points:
- **Speculative object prefetch:** the object is prefetched when greyed, before it is known to need processing in this step; if the step budget expires, the prefetch is wasted (bounded waste ≤ `D` lines).
- **Speculative bitmap prefetch in sweep:** the next bitmap line(s) are prefetched while the current line is processed.

Wasted prefetches cost bandwidth and may pollute cache; the ring depth caps them. **[A]**

### 3.6 Sweeping: bit-parallel, payload-free

For each 512-bit bitmap line (one 8 KiB block):

```
dead = start & ~mark                       // objects allocated but not marked → dead (8 × 64-bit word ops; SIMD-able)
live = start & mark
block.live_count = popcount(live)
for each set bit b in dead (tzcnt loop or SIMD compress):
    free granule b                          // dense pool: gen[slot]++, push free slot; bump: nothing individually
clear mark bits for next cycle (or flip mark polarity per cycle, avoiding the clear)
```

- One line fill classifies up to 512 candidate objects, with no payload touch (assuming `reclaim_T` has no side effects, since `Effectful` types are handled elsewhere). **[A]**
- Dense-pool slots: the dead set updates the generation array (side table, 4 B per slot) and free lists; again no payload access.
- Bump segments: no per-object free. Per-block live counts roll up into the segment's `live_bytes`; a fully dead segment is recycled whole; a sparse segment becomes a `TileCompact` candidate. **[A]**
- **Polarity flip**: alternate which bit value means "marked" each cycle, avoiding a bulk clear pass. `OPEN:` interaction with concurrent allocation-as-marked.

### 3.7 Concurrency
- **Concurrent marking** with SATB: before overwriting a reference field, the barrier pushes the *old* value to a per-thread buffer (unless the holder is `OWNED`, §2.6). Buffers are drained by markers. Allocation during marking is *allocate-marked* (the new object's mark bit set).
- **Multiple markers:** partition by heap region; work stealing of grey rings. `fetch_or` on a shared bitmap line from different markers causes line bouncing; mitigation: region-affine marking (a marker prefers objects in its own region), test-before-set.
- **Sweeping** is per-block and independent, so it can run lazily (on allocation demand), concurrently on workers, or incrementally in micro-steps (§4). **[A]**

### 3.8 Interaction with `TileCompact`
- A tile under evacuation is **not marked concurrently**, or its bitmap range is moved with the objects during copy (the copy routine copies `start`/`mark` bits for the object granules it relocates).
- Handle-addressed objects are marked through the handle table (handle → slot → object), so relocation changes only the table entry, not references.
- Raw-address references from other objects to moved objects require either handle indirection (default) or an update pass (not the default). **[A]**

### 3.9 Failure and edge cases

| Case | Behavior |
|------|----------|
| **Mark-ring overflow / deep structures** | Ring is bounded; overflow spills to a larger work stack; long arrays traced in chunks (e.g., 128 elements per step) |
| **Large arrays of references** | Unrolled tracer replaced by a chunked loop with software prefetch ahead by `D` elements |
| **Side-table false sharing** | Region-affine markers; test-before-set |
| **Prefetch pollution / useless prefetch** | Ring depth cap; PMU-based back-off (`StallHarvest` §4.6) |
| **Tracer code size** | Per-type routines; shape sharing for structurally identical types |
| **Weak references / finalizers** | Processed in a separate phase after marking; finalizable objects resurrected for one cycle; excluded from payload-free sweep |
| **Interior pointers** | Map to object start via `start` bitmap (search backward for the nearest set bit within the block, bounded by block size) |
| **Granule waste** | Small objects round up to 16 B; documented internal fragmentation; 8 B granule option doubles bitmap density |
| **Precise maps missing (FFI-held objects)** | Pinned + treated as roots with explicit root registration |
| **Mark/relocation mismatch bugs** | Debug verification pass: recompute reachability with a slow collector and compare |

### 3.10 Trade-offs

| Benefit | Cost |
|---------|------|
| Sweep reads ~1/128 of the heap's bytes as metadata; one line = 512 objects' status **[A]** | Side-table memory (~0.8% for `mark` + same for `start`) |
| Object lines never dirtied by marking **[A]** | Extra indirection to compute the bitmap address (shift/mask), usually L1/L2-resident |
| Prefetch ring overlaps misses **[A]** | Prefetch accuracy is workload-dependent **[H]**; wasted bandwidth |
| Unrolled tracers: no type-map decode **[A]** | Code size; JIT/AOT dependence |

---

## 4. `StallHarvest` — Stall-Cycle Harvesting

### 4.1 Purpose
Run collector work in **time that would otherwise be wasted**, in strictly bounded pieces, so that GC progress does not require long pauses or dedicated cores, and degrades gracefully when idle time is scarce.

### 4.2 An honest model of what can be harvested

A *stall* occurs at different time scales, and a single thread can only execute one instruction stream, so the harvesting mechanism differs by scale:

| Scale | Cause | Harvestable how | Granularity |
|-------|-------|-----------------|-------------|
| **~50–100 ns** | A single DRAM miss | Only by **interleaving independent work around the miss** (software coroutine/prefetch interleaving), or via an **SMT sibling** hardware thread. Out-of-order execution already hides part of this; PMU "stall" cycles are an *upper bound* on what is harvestable. | **Nano-step** (≈ 50–200 ns, 1–4 grey objects whose lines are already prefetched) |
| **~1–100 µs** | Lock wait, futex, queue-empty poll, allocator slow path, blocked-on-network/disk-completion spin, RPC await | Run a **micro-step** at a software-visible wait point | **Micro-step** (≈ 1–10 µs) |
| **> 100 µs** | Idle cores, blocked threads | Hand work to background workers | **Slice** (bounded, ≈ 100 µs–1 ms) |

Claims about harvesting must name the scale. This paper's `gc_micro_step()` targets the middle tier and the nano-step interleaving at the bottom tier; the top tier is the ordinary background collector. **[A]**

### 4.3 Inputs
- **PMU telemetry** (optional): backend-stall cycles, memory-stall cycles, LLC misses, retired instructions, read with userspace counter reads (e.g., `rdpmc` on x86 after `perf_event_open` setup; equivalent mechanisms on ARM where permitted). Event names vary by microarchitecture.
- **JIT control-flow prediction:** the JIT annotates code with **poll sites** and predicts stall length class at each:
  - before a loop whose profile shows cold pointer dereferences,
  - before blocking calls, lock slow paths, `await` points, and allocation slow paths,
  - at function entry of request handlers awaiting I/O.
- **GC debt:** `debt = pending_mark_work + pending_sweep_blocks`, scaled by allocation rate (how much work must be done per allocated byte to finish before the heap fills).
- **Latency policy:** `no_gc` regions and deadlines (e.g., mark a request handler's critical section as non-harvestable).

### 4.4 State and metadata

```c
struct PollSite {            // one per poll site, in a side table
    u32 site_id;
    u16 predicted_stall_cyc_q;   // EWMA of observed gap, log-quantized
    u8  class_;                  // Nano | Micro | None
    u8  disable_until_epoch;     // back-off after bad outcomes
    u16 wasted_count, useful_count;
};
struct HarvestState {        // per thread
    u64 last_pmu_snapshot[K];    // stalls, misses, instructions
    u32 tokens;                  // harvested-cycle budget available to GC (token bucket)
    u32 debt_share;
};
```

### 4.5 Mechanism

**Poll.** At a poll site, compiled code executes a short inline sequence: check the site's `class_` and `tokens`, and (for Micro sites) compare predicted stall against the cost of one micro-step.

```
gc_poll(site):
    if site.class_ == None or tls.no_gc: return
    pred = site.predicted_stall_cyc
    if debt == 0 or pred < MIN_STEP_COST + SAFETY_MARGIN: return
    budget = min(pred - SAFETY_MARGIN, B_MAX)         // B_MAX: hard cap, e.g. ~10 µs of cycles
    t0 = rdtsc()
    gc_micro_step(budget)                              // §4.5.1
    record_outcome(site, elapsed = rdtsc() - t0)       // update EWMA; check overrun
```

**4.5.1 `gc_micro_step(budget)`** performs, in priority order, within `budget`:
1. Drain a few SATB buffer entries;
2. `mark_step` with the prefetch ring (`CacheSweep` §3.4), stopping at budget or when the ring is empty;
3. `sweep` blocks (`CacheSweep` §3.6), whole lines at a time;
4. Optionally a `TileCompact` copy slice (Paper 01 §4.5) or `MasterSlaveMM` reclaim batch (Paper 02 §4.4) if debt in those subsystems is high.
It must be **resumable at any line boundary** and must check the cycle counter between units of work. **[A]**

**Nano-step interleaving (software):** in a hot loop with a known miss, the JIT emits: `PREFETCH(next); gc_nano_step(≈100 ns budget); use(next)`. Only the work that is already cache-ready (grey objects whose prefetch was issued earlier) runs. **[A]**

**Learning the stall map from the PMU.** Between two consecutive polls, the thread reads `stall_cycles` and `cycles`. If the stall fraction in a region is persistently high, the region's `predicted_stall` is raised; otherwise lowered (EWMA). Sites with low realized stall are disabled (back-off) so polls don't add overhead. **[A]**

### 4.6 Abort and safety conditions

| Condition | Action |
|-----------|--------|
| Budget exhausted | Return immediately (hard cap) |
| `elapsed > 1.5 × budget` (overrun) | Penalize site (`wasted_count++`), shrink prediction |
| LLC-miss rate after steps exceeds pre-step baseline by `ε` (cache pollution) | Back off that site for `N` epochs; reduce prefetch depth |
| Mutator signals `no_gc` or deadline slack < threshold | Skip |
| Allocation stall imminent (heap near full, debt high) | **Forced slice**: guarantees minimum mutator-utilization floor of GC progress regardless of stalls (so harvesting can't starve the collector) |
| PMU read fails / not permitted | Fall back to JIT prediction only; coarser classes |

### 4.7 Scheduling and fairness

- **Token bucket:** harvested stall time refills `tokens`; GC may spend tokens; if the supply over a window is below required debt rate, the scheduler increases the proportion of forced slices (minimum mutator utilization MMU-style guarantee) and wakes background workers. **[A]**
- **Shared budget with other background actors** (`MasterSlaveMM` reclaim, `CTMM` workers, `TileCompact`): one coordinator divides idle time to avoid oversubscription (Paper 05 §3).

### 4.8 Failure and edge cases

| Case | Behavior |
|------|----------|
| **PMU unavailable** (VMs, containers, `perf_event_paranoid`, cloud instances) | Prediction-only mode; documented loss of accuracy |
| **Thread migration** | Per-thread counters follow the thread (per-thread perf contexts); snapshot re-baselined on migration; cross-core PMU differences on hybrid CPUs (P vs E events differ) are handled by per-core-type event tables |
| **Misprediction → GC extends the mutator's critical path** | Bounded by `B_MAX`; sites penalized; hard `no_gc` regions; tail impact measured in Paper 05 |
| **Cache pollution from GC data** | Non-temporal prefetch hints (e.g., `PREFETCHNTA`) for bitmap/object lines that won't be reused; miss-rate feedback |
| **Measurement overhead** | `rdpmc` costs on the order of tens of cycles; sample only at poll sites with non-trivial predicted stall |
| **Sync with SATB/handoff** | Micro-steps run at safepoint-consistent states; a step never holds a lock across its boundary |
| **SMT-based harvesting** | Optional; sibling-thread GC contends for shared core resources; may slow the mutator; evaluate per platform `OPEN:` |
| **Security** | Userspace counter access can aid side-channels; deployments with strict isolation may disable PMU use |
| **Priority inversion** | Micro-steps are non-blocking; no locks; abort if any step would wait |

### 4.9 Trade-offs

| Benefit | Cost |
|---------|------|
| Converts wasted waits into GC progress, avoiding dedicated GC cores or long pauses **[A]** | Complexity; per-platform tuning |
| Bounded steps keep p99 impact capped **[A]** | Misprediction costs fall on the mutator's tail **[A]**; real benefit **[H]** |
| Degrades gracefully (forced slices, background workers) **[A]** | PMU access and permissions; hybrid-core handling |
| Learns stall map automatically **[A]** | Warmup period; instability if workload phase changes frequently (needs hysteresis) |

---

## 5. How the Three Fit Together

```
Alloc site (JIT) ──► SpecNoEscape? ──yes──► OWNED object, compiled release at last use      ┐
                        │                      │  escape point → guard → handoff (TRACED)    │ JITDestruct
                        no                     ▼                                             │
                        └──────────────────► TRACED object ──► residual collector            ┘
                                                           │
            ┌──────────────────────────────────────────────┴───────────────────────────────┐
            │ Marking: SATB barrier (elided for OWNED), prefetch ring, unrolled tracers,     │
            │          side-table mark bits (no header writes)                               │ CacheSweep
            │ Sweeping: bit-parallel over 64B lines, payload-free, segment/tile bookkeeping │
            └──────────────────────────────────────────────┬───────────────────────────────┘
                                                           │ "when to run"
            gc_micro_step(budget) ◄── poll sites + PMU stall map + GC debt + MMU floor        │ StallHarvest
```

Data the techniques share: the **side tables** (ownership bits from `JITDestruct`, mark/start bits for `CacheSweep`, per-site stall maps for `StallHarvest`), the **per-type routines** (`reclaim_T` and `trace_T` are generated together from the same type information), and the **poll/safepoint infrastructure**.

## 6. Hypotheses

| ID | Hypothesis | Falsified if |
|----|------------|--------------|
| H4.1 | `JITDestruct` with `OWNED` bits and barrier elision reduces GC-related mutator overhead (barrier + collector work) on allocation-heavy code with mostly non-escaping temporaries | No reduction vs a baseline with escape analysis but no ownership bits |
| H4.2 | Handoff rate stays below 5–10% for typical request-processing code; recompilation fallback handles outliers | Handoff storms common and fallback thrashes |
| H4.3 | Side-table marking has fewer cache misses and lower coherence traffic than header-bit marking at equal heap | No difference in LLC/coherence counters or mark throughput |
| H4.4 | Prefetch ring with depth 8–16 improves mark throughput on pointer-heavy heaps vs no prefetch | No throughput gain, or gain vanishes on target hardware |
| H4.5 | Bit-parallel sweep is faster than per-object sweep by a large factor on sparse-dead heaps | Not faster, or dominated by other costs |
| H4.6 | `StallHarvest` achieves a given GC throughput with lower mutator p99 than a periodic-pause baseline | p99 not lower, or wasted work dominates |
| H4.7 | PMU-informed prediction outperforms JIT-prediction-only | No measurable difference |

## 7. Open Questions

- `OPEN:` Adaptive prefetch ring depth `D` from PMU feedback (fill-buffer occupancy).
- `OPEN:` Mark-polarity flipping under allocate-marked concurrency.
- `OPEN:` SMT sibling harvesting vs software interleaving: which wins on which cores.
- `OPEN:` Representation of `OWNED`/`TRACED` in 1 vs 2 bits and the cost of segment pinning by escaped objects in bump segments.
- `OPEN:` Whether to generate tracers at AOT time for startup vs JIT time for profile-driven specialization.
