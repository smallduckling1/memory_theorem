# THEORY — Why Memory Management Needs Rethinking, and What We Propose

This is the main readable document. It has three parts:

1. **Why today's models fall short and what they really cost**
2. **The Named Technique Catalog** (each technique explained separately)
3. **How the pieces fit together and where they apply**

Claim tags: **[A]** architectural, **[H]** hypothesis needing benchmarks, **[M]** measured (none yet). See `README.md`.

Hardware numbers below are typical orders of magnitude for modern x86/ARM server and desktop parts. Measure your own machine before relying on them.

---

# Part 1 — Why Today's Models Fall Short

## 1.0 The hardware reality every model has to live with

Memory is not flat. A program that treats it as flat pays for the difference on every access. **[A]**

| Level | Typical latency | Granularity |
|-------|-----------------|-------------|
| L1 data cache | ~1 ns (4–5 cycles) | 64 B line |
| L2 | ~3–5 ns | 64 B line |
| L3 (shared) | ~10–20 ns | 64 B line |
| DRAM | ~60–100 ns (more under NUMA / contention) | 64 B line, 4 KiB page |
| TLB miss (page walk) | tens to hundreds of cycles, partly cached | 4 KiB page (or 2 MiB / 1 GiB huge) |
| Minor page fault | ~0.5–several µs | 4 KiB page |
| Major page fault (SSD) | ~100 µs+ | 4 KiB page |

Three consequences drive everything that follows:

- **Dependent loads serialize.** Following `a->next->next->next` cannot be overlapped by out-of-order execution because each address depends on the previous load. A linked structure scattered across the heap costs roughly *N × miss latency*, while the same data in a contiguous array lets the hardware prefetcher stream it.
- **Bytes fetched ≠ bytes used.** A 64-byte line is fetched whole. If a loop needs one 4-byte field from a 96-byte record, ~94% of the bandwidth is wasted, and the cache holds less useful data.
- **Latency tails are caused by *events*, not averages.** A page fault, a GC pause, a lock convoy, a compaction, a cross-core cache-line bounce: each is rare, each is large, and together they define p99/p99.9.

## 1.1 Manual `malloc` / `free`

### How it works
The allocator keeps free lists or size-class bins, often per-thread arenas. `malloc` finds a block; `free` returns it. Block headers or side metadata record size and state. The program holds raw pointers.

### What it actually costs

**Latency and p99 jitter.** Typical `malloc`/`free` is fast, but the tail is not. Slow paths include: refilling a thread cache from a central arena (lock or atomic contention), coalescing neighbors, splitting large blocks, `mmap`/`brk` growth, `madvise`/`munmap` returning memory to the OS, and the page faults that follow first touch of fresh pages. A request handler that allocates inside its critical path inherits all of these in its tail. **[A]**

**Pointer chasing and cache misses.** `malloc` returns memory by size class and availability, not by *who will use it together*. Nodes of one tree or list end up interleaved with unrelated allocations, so traversal touches many lines and many pages (TLB pressure). Allocation order after a long uptime is effectively random. **[A]**

**Fragmentation.** *External* fragmentation: free memory exists but in pieces too small to satisfy a request. *Internal* fragmentation: size-class rounding wastes the tail of each block (often 10–25% for general-purpose size-class schemes **[H]**). Because the program holds raw pointers, the allocator **cannot move** live objects to repair this. Long-running services therefore ratchet up RSS and rarely come back down. **[A]**

**Memory bloat.** Per-allocation headers (8–16 B), alignment padding, per-thread caches holding free blocks that other threads cannot use, and freed pages not returned to the OS together yield resident memory well above live bytes. Returning memory (`madvise(DONTNEED)`) has its own cost (page faults when reused). **[A]**

**Developer and engineering cost.** Use-after-free, double free, leaks, and buffer overflows are memory-safety bugs. Large vendors (notably Microsoft and the Chromium project) have publicly reported that roughly 70% of their severe security vulnerabilities were memory-safety issues; treat the exact figure as a widely cited industry estimate, not a precise constant. Beyond security: debugging time, sanitizer slowdowns (ASan commonly doubles runtime and multiplies memory), code-review burden for ownership conventions that exist only in comments, and the defensive coding patterns (copying "to be safe") that waste runtime. **[A]**

**What it gets right.** Predictable average cost, no runtime collector, direct control, no barriers on pointer writes. Any replacement has to beat this on the hot path or justify itself elsewhere.

## 1.2 Tracing Garbage Collection

### How it works
The runtime periodically finds reachable objects starting from roots (stacks, globals, registers) by traversing pointers, and reclaims the rest. Variants: stop-the-world mark-sweep, copying/semispace, generational, concurrent mark (with write/read barriers), region-based incremental compaction.

### What it actually costs

**Latency and p99 jitter.**
- *Stop-the-world phases* scale with live set or root set; even "short" pauses (milliseconds) are enormous against a µs-scale SLO.
- *Concurrent collectors* (e.g., ZGC- or Shenandoah-style designs) cut pauses sharply, but shift cost: load/store barriers on mutator paths, extra CPU for GC threads competing with application threads, and allocation stalls when the collector falls behind the allocation rate. **[A]**
- The pause is not the only symptom. A collection also **evicts the application's working set from cache** because marking sweeps through the heap. The mutator resumes with a cold cache, so the cost shows up as slowness *after* the pause. **[A]**

**Pointer chasing and cache misses.** Marking *is* pointer chasing: visit object, load its reference fields, visit those. Mark throughput is bound by memory latency and parallelism of the frontier, not by ALU. Mark bitmaps and mark-bit-in-header schemes touch extra lines (header write dirties the line and invalidates other cores' copies). **[A]**

**Fragmentation.** Non-moving collectors (mark-sweep) fragment like `malloc`. Moving collectors fix this but need either a copy reserve (semispace wastes up to half), or complex incremental compaction plus barriers to keep pointers consistent. **[A]**

**Memory bloat.** To keep collection frequency and cost tolerable, tracing collectors typically run with significant heap headroom. A well-known study (Hertz & Berger, OOPSLA 2005) reported that GC could match explicit management in speed only with a heap several times the live size, and that tighter heaps cost noticeably more time. The ratio is workload-dependent, but "2–3× live size" as a working budget is a common experience. **[A]/[H]** Per-object headers (type word, mark/forward bits, lock/hash bits) add overhead to every object, which matters for small objects.

**Developer and engineering cost.** Tuning (heap size, region size, pause targets, generation ratios), diagnosing allocation-rate-driven regressions, finalizer/ordering pitfalls, the temptation to build object pools to dodge the collector (which reintroduces manual lifetime bugs), and unpredictable interaction with native code (pinning, handles).

**What it gets right.** Memory safety by construction for the managed heap, cyclic structures "just work," and developer productivity. Any alternative must keep these ergonomics or be honest about where it gives them up.

## 1.3 Static Ownership / Borrow Checking

### How it works
The type system tracks a unique owner and bounded borrows. Destruction is inserted at scope end (RAII/drop). Lifetime annotations tie references to owners. No runtime collector.

### What it actually costs

**Latency and jitter.** Excellent: no collector. The tail still comes from the underlying `malloc`/`free` plus drop cascades (dropping a large `Vec<Box<T>>` or a deep tree frees all of it at once at scope end, producing a latency spike at a predictable but often unexpected point). **[A]**

**Pointer chasing and layout.** Ownership says nothing about *where* objects live. `Box<T>` per node is the same scattered heap as in C. Programmers who care about cache behavior reach for arenas or index-based structures, **leaving the ownership model** to do so. **[A]**

**Fragmentation and bloat.** Identical to manual management; the borrow checker proves *safety*, not *density*. Scope-based drop also keeps values alive until end of scope even when last used much earlier (non-lexical lifetimes shorten *borrows*, but drop timing of owned values still follows scope), which holds memory longer than necessary. **[A]**

**Developer and engineering cost.** This is where the cost concentrates:
- Graphs, doubly linked structures, observers, caches with back-references, and shared mutable state don't fit single-owner trees. Escape hatches (`Rc<RefCell<T>>`, `Arc<Mutex<T>>`, indices into arenas, `unsafe`) reintroduce runtime checks or unchecked code.
- Lifetime annotations spread virally through APIs.
- Refactoring friction: a changed ownership shape propagates through call graphs.
- Learning curve and compile-time cost.
- `unsafe` blocks must uphold invariants the compiler can no longer check.

**What it gets right.** Safety and deterministic reclamation with zero runtime tracing cost for code that fits the model. The goal here is to *keep* this guarantee where it applies, not discard it.

## 1.4 Flat Memory vs. Real Workloads

All three models above assume memory is one big undifferentiated address space where an object is "in memory" or "freed." Real workloads violate this:

| Workload pattern | What flat memory gets wrong |
|------------------|------------------------------|
| **Request/response servers** | Most allocations die with the request; a few (caches, sessions) live for hours. Mixing them pins pages and fragments. |
| **Hot/cold data** | 5–10% of objects take most accesses **[H]**. Flat memory gives hot and cold the same placement, same page, same eviction policy. |
| **Entity/record workloads (games, simulations, databases, analytics)** | Loops touch two fields of a 15-field record across a million records; AoS layout wastes most fetched bytes. |
| **Large caches and indexes** | Cold entries consume RAM that a compressed or spilled representation could free; reconstruction on demand is cheaper than holding. |
| **Long-lived services** | Heap shape drifts over days; without relocation, RSS never recovers. |
| **Mixed-priority tenants/subsystems** | "Who may be evicted first" is a policy decision, but flat memory has no vocabulary for it. |

The cost summary:

| Cost axis | malloc/free | Tracing GC | Borrow/ownership | Flat-memory assumption |
|-----------|-------------|------------|------------------|------------------------|
| p99 jitter | Slow-path events, drop cascades | Pauses, barriers, cache eviction by marking | Drop cascades, allocator slow paths | Page faults, tail from cold data |
| Cache misses / chasing | Scattered nodes | Marking is chasing; scattered heap | Same as malloc | Layout-blind |
| Fragmentation | Cannot move | Depends on collector; moving needs barriers | Cannot move | No compaction vocabulary |
| Bloat | Headers, caches, no return | Headroom + headers | Same as malloc + late drops | No tiering or compression |
| Developer cost | Memory-safety bugs | Tuning, pool hacks | Borrow friction, `unsafe` | Manual caching/pooling code |

**Thesis.** The four costs share a root cause: *the memory system is not told what the program knows* (lifetimes, access patterns, who may be evicted, what is regenerable), *and it cannot act on it* (move, re-layout, compress, evict, reclaim in idle cycles). The techniques below put that knowledge in the compiler and the runtime and give them the means to act.

---

# Part 2 — Named Technique Catalog (How Each Works Separately)

Each card: **Name → Problem → Core mechanism → Lifecycle / dataflow → Trade-offs.** Full mechanics are in the paper listed on each card.

---

## T1. Segmented Bump + Generational Dense Pool — `SegGenPool`
*Paper: `paper/01` §2*

**Problem.** Allocation slow paths, scattered placement, lack of cheap bulk reclaim, and stale-pointer bugs.

**Core mechanism.** Memory is carved into aligned **segments** (default 2 MiB). Two allocation paths share them:
- **Bump path:** a thread-local open segment hands out memory by pointer increment. Individual objects are not freed; a segment is reclaimed wholesale when its live counter reaches zero.
- **Dense generational pool path:** fixed-size slots stored contiguously, addressed by a **handle** `(segment, slot, generation)`. Freeing bumps the slot's generation and swaps the last element into the hole, keeping the array dense.

**Lifecycle.** `Free → Open → Sealed → (Evacuating) → Free`. Allocate: bump or pop a slot. Use: resolve handle (check generation) or use pointer. Release: decrement segment live count (bump) or retire slot (pool). Reclaim: segment-wide when empty; otherwise handled by `TileCompact`.

**Trade-offs.** O(1) allocation and bulk free; locality by allocation time; stale handles caught. Costs: one extra indirection for pool handles, memory held by a single survivor in a segment, large objects need a separate path, generation counters can wrap (handled by retiring slots).

---

## T2. Last-Use Causal Lifetime — `LastUseLifetime`
*Paper: `paper/01` §3*

**Problem.** Scope-end release holds memory too long; GC does the work of discovering what the compiler could already know; borrow checking is too rigid for non-tree shapes.

**Core mechanism.** The compiler builds a **causal lifetime graph**: values, their aliases/derived references, and use events. For each allocation it computes the **last causal use** across all aliases on each control-flow path and emits a *release action* immediately after, on the CFG edge where the value becomes dead.

**Lifecycle.** Allocate (tag with release kind) → uses → release action at last use. Release kinds: `Free` (return slot), `SegDec` (decrement segment live count), `Defer` (hand to a Master, see T5), `Escaped` (cannot prove; demote to a runtime-managed policy).

**Trade-offs.** Tightens live ranges, shrinks footprint, removes tracing for provable cases, no lifetime annotations required. Costs: analysis precision depends on alias information; opaque calls and dynamic dispatch force fallbacks; release order differs from scope order, so **types with observable destructor effects (locks, files, I/O) keep scope semantics** unless annotated.

---

## T3. Incremental Tile Compaction — `TileCompact`
*Paper: `paper/01` §4*

**Problem.** Fragmentation that raw-pointer models cannot repair, and compaction that causes long pauses.

**Core mechanism.** Heap is partitioned into **tiles** (64–256 KiB). Sparse sealed tiles are scored (reclaimable bytes ÷ copy cost) and **evacuated in small, budgeted steps**: copy live objects to a destination tile, publish new locations by atomic handle-table update (or forwarding word), wait out an epoch grace period, then free the tile.

**Lifecycle.** `Select → Reserve → Copy → Publish → Quiesce → Release`. Each step is bounded (bytes and microseconds) and resumable, so it can run at safepoints or on background cores.

**Trade-offs.** Bounded pause per step, RSS can shrink, locality improves as survivors are packed. Costs: copying bandwidth, handle/forwarding indirection, pinned objects block full reclaim, write races with mutators require a protocol.

---

## T4. Adaptive Layout Morphing & Columnar Slabs — `LayoutMorph` / `ColSlab`
*Paper: `paper/01` §5*

**Problem.** Layout is chosen once at design time but access patterns vary by phase; AoS wastes bandwidth on field scans, SoA hurts whole-record access.

**Core mechanism.** **`ColSlab`** stores N records of one type as per-field columns (64B-aligned) with a validity bitmap and generation array; records are addressed by stable handles. **`LayoutMorph`** samples access sites to build a field co-access profile, estimates cache lines touched per layout, and chooses among AoS, SoA, AoSoA (SIMD-width blocks), or hot/cold split. Morphing is performed opportunistically during `TileCompact` evacuation, when data is being copied anyway.

**Lifecycle.** Profile → decide (with hysteresis) → morph during evacuation → bump layout epoch → guarded accessors re-link.

**Trade-offs.** Field scans become sequential and SIMD-friendly; hot data packs densely. Costs: profiling overhead (sampled), morph copy cost, interior pointers (`&rec.field`) are invalid across a morph so borrows must use handles or pin scopes, and accessor code must be specialized per layout.

---

## T5. Master / Slave Memory Management — `MasterSlaveMM`
*Paper: `paper/02`*

**Problem.** Reclamation policy (TTL, capacity, priority) is scattered through application code, and reclamation work runs on the latency-critical path.

**Core mechanism.** A **Master** owns policy (capacity, TTL, eviction order, quota) for a set of **Slaves**, which are scoped objects whose lifetime is bounded by the Master. Slave state is explicit: **`Alive`, `Deleted`, `Evicted`, `Expired`**. Hot-path code only flips a status word; **background workers** perform the physical reclaim, eviction, and rehydration.

**Lifecycle.** `Alive → Deleted` (logical delete; physical free deferred), `Alive → Expired` (TTL), `Alive → Evicted` (payload dropped, handle valid, can be rehydrated). Generation/epoch tags prevent a worker from acting on a slot that has been reused.

**Trade-offs.** Takes reclaim off the critical path; policy is centralized and testable. Costs: cross-core cache-line transfers for status words and queues, memory held until workers run, requires discipline about which code may hold a Slave beyond its Master's scope.

---

## T6. Cognitive-Thermodynamic Memory Model — `CTMM`
*Paper: `paper/03`*

**Problem.** All objects get the same residency treatment, so cold data consumes hot resources and reconstructible data is never dropped.

**Core mechanism.** Each object has a **temperature** `T` (exponentially decayed access frequency) and a **semantic weight** `W` (cost to regenerate / importance). States:
- **Plasma:** hottest; densely packed, fast-path access, no indirection beyond the handle.
- **Liquid:** normal residency in standard layout.
- **Crystalline:** cold; compressed or packed, read-mostly.
- **Sublimation:** payload released entirely, leaving only a descriptor (a recipe or backing-store key) so the object can be rebuilt or refetched.

Transitions use **hysteresis** (different up/down thresholds and minimum dwell time) to prevent thrashing. Handles are **two-tier** `(slot_index, epoch_gen)`: the slot index is stable, the epoch changes on tier transitions so stale references are detected, and access to a non-resident object triggers **transparent thawing/hydration**. Background demotion/promotion runs on E-cores with lock-free queues.

**Trade-offs.** Larger effective capacity and better cache residency for hot data; graceful degradation under memory pressure. Costs: decay bookkeeping, thaw latency on cold access (a new tail source), compression CPU cost, and a need for correct regeneration descriptors.

---

## T7. JIT–GC Co-Design — umbrella

The JIT knows types, escape behavior, and control flow; the collector needs exactly that. Three techniques use it.

### T7a. JIT-Synthesized Destructors — `JITDestruct`
*Paper: `paper/04` §2*

**Mechanism.** For each hot allocation type, the JIT emits a specialized reclaim routine (field-wise release, no generic traversal), plus **speculative escape guards**: a value assumed non-escaping is released directly; if a guard sees it escape, control **hands off** to the general collector path (deopt-style transition).

**Trade-offs.** Eliminates generic tracing for the common case; guard cost on slow edges; code-cache growth; handoff must be race-free.

### T7b. Speculative Cache-Line Sweeping — `CacheSweep`
*Paper: `paper/04` §3*

**Mechanism.** Mark/liveness metadata lives in **64B-aligned side-table bitmaps** (one cache line = 512 bits covering a fixed span of objects) so a sweep step reads one line and decides many objects. JIT-generated **unrolled tracers** per type issue **`PREFETCH`** for upcoming children before visiting them, converting dependent-load latency into overlapped misses.

**Trade-offs.** Fewer lines touched, no header writes (no cache-line invalidation of object data), better memory-level parallelism. Costs: side-table memory, tracer code size, prefetch accuracy depends on type shape.

### T7c. Stall-Cycle Harvesting — `StallHarvest`
*Paper: `paper/04` §4*

**Mechanism.** PMU counters (e.g., cycles stalled on memory, LLC misses) plus JIT control-flow prediction identify windows where the mutator is about to stall or is stalled; `gc_micro_step()` (bounded to a few µs of work) is scheduled into them. A hard budget and abort conditions prevent GC work from becoming the stall.

**Trade-offs.** Uses otherwise wasted cycles. Costs: PMU access/permissions and sampling overhead, prediction errors cause wasted or harmful work (cache pollution), hardware-specific tuning.

---

# Part 3 — How the Pieces Fit Together and Where They Apply

## 3.1 Layered composition

```
 ┌───────────────────────────────────────────────────────────────┐
 │ Compiler layer:  LastUseLifetime  (release kind per allocation) │
 └───────────────┬───────────────────────────────┬───────────────┘
        proven   │                     Defer /   │  Escaped
        release  ▼                     policy     ▼
 ┌───────────────────────────┐   ┌──────────────────────────────┐
 │ Structure: SegGenPool      │   │ Policy: MasterSlaveMM         │
 │  (bump + dense pool)       │◄──┤  (statuses, background reclaim)│
 └───────┬───────────────────┘   └──────────────┬───────────────┘
         │ fragmentation                         │ residency
         ▼                                       ▼
 ┌───────────────────────────┐   ┌──────────────────────────────┐
 │ TileCompact  ──morphs──►   │   │ CTMM: Plasma/Liquid/          │
 │ LayoutMorph / ColSlab      │   │       Crystalline/Sublimation │
 └───────────────────────────┘   └──────────────────────────────┘
                 ▲
                 │ runtime collector for what the compiler can't prove
 ┌───────────────────────────────────────────────────────────────┐
 │ JIT-GC: JITDestruct (fast path) · CacheSweep (trace) ·         │
 │         StallHarvest (when to run)                             │
 └───────────────────────────────────────────────────────────────┘
```

**Decision flow for one allocation:**
1. Compiler (`LastUseLifetime`) proves a last use → emits `Free`/`SegDec`. Done; no runtime tracing involved. **[A]**
2. Not provable but owned by a policy object → `Defer` to a Master (`MasterSlaveMM`). Reclamation is policy-driven and off-path.
3. Long-lived and variably hot → `CTMM` tiers it.
4. Anything left escapes to the collector: `JITDestruct` handles the common typed case, `CacheSweep` traces the rest, `StallHarvest` decides when.
5. Throughout, `SegGenPool` provides placement and handles, `TileCompact` repairs fragmentation, and `LayoutMorph`/`ColSlab` reshape record data during evacuation.

## 3.2 Shared primitives (what makes composition feasible)

| Primitive | Used by |
|-----------|---------|
| Stable generation-tagged handles | SegGenPool, TileCompact, MasterSlaveMM, CTMM, ColSlab |
| Epoch-based reclamation (grace periods) | TileCompact, MasterSlaveMM, CTMM, JITDestruct handoff |
| Side-table metadata (outside object payload) | SegGenPool, CacheSweep, CTMM, MasterSlaveMM status words |
| Lock-free MPSC/SPSC queues | MasterSlaveMM, CTMM, remote frees in SegGenPool |
| Safepoints / deopt guards | JITDestruct, LayoutMorph accessor re-linking, TileCompact steps |
| Background (E-core) workers | MasterSlaveMM, CTMM, TileCompact, StallHarvest fallback |

## 3.3 Conflicts and tensions (summary; detail in `paper/05`)

| Tension | Explanation | Mitigation |
|---------|-------------|------------|
| `LastUseLifetime` vs destructor side effects | Release at last use reorders observable destructors | Restrict to pure-memory types; scope semantics for effectful types |
| `TileCompact` vs raw pointers / FFI | Moving objects invalidates addresses | Handles by default; pin scopes; pinned objects excluded, tile reported as "pinned" |
| `LayoutMorph` vs interior borrows | `&rec.field` is invalid after morph | Borrows through handles or pin scopes; morph only unpinned slabs |
| `CTMM` thaw latency vs p99 goals | Cold access now pays decompress/refetch | Prefetch on predicted access; per-class "never below Liquid" |
| `MasterSlaveMM` background reclaim vs memory pressure | Deferred frees hold memory | Reclaim watermark triggers foreground help |
| `StallHarvest` vs cache pollution | GC work in stall windows may evict the mutator's lines | Budget caps; non-temporal hints; abort on rising miss rate |
| Handle indirection vs raw-pointer speed | Every access has a resolve step | Pin scopes hoist resolve out of loops; JIT specializes |
| Many techniques at once vs complexity | Each adds metadata and invariants | Adopt incrementally; `paper/05` compatibility matrix |

## 3.4 Where this applies

| Domain | Techniques most relevant | Why |
|--------|--------------------------|-----|
| **Request/response servers (RPC, web, API gateways)** | `LastUseLifetime`, `SegGenPool` (request segments), `MasterSlaveMM` (sessions/caches), `StallHarvest` | Per-request bulk free, predictable tails, cache TTLs off-path |
| **Games / simulations / ECS** | `ColSlab`, `LayoutMorph`, `SegGenPool` dense pools | Field scans over large entity sets, stable entity handles |
| **In-memory databases / analytics** | `ColSlab`, `CTMM`, `TileCompact` | Columnar scans, cold-data compression, spill/regenerate |
| **Large caches / CDNs / indexes** | `CTMM`, `MasterSlaveMM` | Weighted eviction, regenerable entries, TTL |
| **Language runtimes (the compiler/JIT itself)** | `JIT-GC Co-Design`, `LastUseLifetime` | Compiler knowledge feeds collector |
| **Long-running daemons** | `TileCompact`, `CTMM` | RSS recovery, drift control |
| **Latency-critical trading/telemetry loops** | `LastUseLifetime`, `SegGenPool`, `StallHarvest` (careful) | Avoid slow paths; harvest idle cycles |

**Poor fits:** tiny, short-lived programs where startup and metadata outweigh benefits; hard-real-time systems that cannot tolerate any speculative runtime machinery; code dominated by FFI-held raw pointers (pinning defeats relocation).

## 3.5 What this package does not claim

Every performance benefit above is **[A]** (follows from mechanics) or **[H]** (needs benchmarks). `paper/05` defines how to test them: workloads, baselines, hardware reporting, and which results would falsify a technique.
