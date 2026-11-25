# Paper 01 — Structural and Lifetime Models

Techniques covered:

1. **Segmented Bump + Generational Dense Pool** — `SegGenPool`
2. **Last-Use Causal Lifetime** — `LastUseLifetime`
3. **Incremental Tile Compaction** — `TileCompact`
4. **Adaptive Layout Morphing & Columnar Slabs** — `LayoutMorph` / `ColSlab`

Each is specified independently first (§2–§5), then combined (§6).
Claim tags: **[A]** architectural · **[H]** hypothesis · **[M]** measured (none).
Struct definitions are illustrative C-like pseudocode, not a final ABI.

---

## 1. Common Conventions

| Item | Default | Notes |
|------|---------|-------|
| Segment size | 2 MiB, 2 MiB-aligned | Matches x86-64 huge page size so a segment can be one huge page. Configurable. |
| Tile size | 64–256 KiB, aligned, subdivides a segment | Unit of compaction. |
| Cache line | 64 B | Side-table and column alignment unit. |
| Handle | 64-bit | Layout in §2.3. |
| Side tables | Separate from payload | Segment metadata, per-slot generation, status bits. |
| Epoch scheme | EBR (epoch-based reclamation) | Global epoch; each thread publishes its local epoch at safepoints or pin-scope entry. A resource retired at epoch `e` may be freed when all threads have observed `e+2`. |

**Handle vs raw pointer.** A *handle* is stable across relocation; a *raw pointer* is valid only within a **pin scope** (§4.6). Compiled code resolves handle → pointer once per scope and hoists the result out of loops. **[A]**

---

## 2. `SegGenPool` — Segmented Bump + Generational Dense Pool

### 2.1 Purpose
Give O(1) allocation, O(1) bulk reclamation, good placement locality, and stale-reference detection, through two cooperating allocation paths sharing one segment substrate.

### 2.2 Inputs
- Allocation request: `(size, align, type_id, lifetime_hint, release_kind)`.
  - `lifetime_hint ∈ {Short, Medium, Long, Unknown}` comes from `LastUseLifetime` (site-level prediction) or a profile.
  - `release_kind` from `LastUseLifetime` (§3.4).
- Thread identity (for thread-local open segments).

### 2.3 State and metadata

**Segment metadata (side table, one entry per segment, 64 B aligned):**

```c
struct SegMeta {
    atomic<u8>  state;        // Free, Open, Sealed, Evacuating
    u8          class_;       // Bump | DensePool(size_class) | Large
    u16         flags;
    u32         owner_tid;    // thread that owns Open state (bump path)
    u64         birth_epoch;
    atomic<u32> live_bytes;   // bump: decremented on SegDec releases
    atomic<u32> live_objs;
    u32         bump_off;     // bump pointer (owner-only while Open)
    u32         dense_count;  // dense pool: number of occupied dense positions
    u32         free_slot_head; // dense pool: freelist of slot indices
    MpscQueue*  remote_free;  // frees from non-owner threads
    // padding to 64 B
};
```

**Dense pool arrays (per dense segment):**

```c
struct DensePool {
    u32  slot_to_dense[CAP];    // sparse: slot -> dense index (or INVALID)
    u32  dense_to_slot[CAP];    // dense: dense index -> slot
    u32  gen[CAP];              // generation per slot (side table)
    T    payload[CAP];          // dense, contiguous
};
```

**Handle (64 bits):**

```
 63              44 43            23 22             0
 [ seg_idx : 20  ] [ slot : 21    ] [ gen : 23      ]
```

- 20 bits → up to ~1 M segments (2 TiB of address space at 2 MiB).
- 21 bits → up to 2 M slots/segment (enough for 16 B slots in 2 MiB = 131,072).
- 23 bits of generation → 8.4 M reuses per slot before wrap (see §2.8).

(Bit allocation is tunable; the invariant is `seg_idx` locates metadata, `slot` locates the entry, `gen` validates.)

### 2.4 Allocation paths

**Bump path** (variable-size, typically short/medium-lived, no per-object free):

```
alloc_bump(size, align):
    seg = tls.open_segment[lifetime_class]
    off = align_up(seg.bump_off, align)
    if off + size <= SEG_SIZE:
        seg.bump_off = off + size
        seg.live_bytes += size        // owner-only: plain add
        return seg.base + off
    return slow_path_new_segment(size, align)
```

The fast path is a compare, an add, and a store; there is no lock and no atomic because the open segment is thread-owned. **[A]**

**Dense pool path** (fixed-size, long-lived or frequently iterated, needs stable handles):

```
alloc_dense(type):
    seg = tls.dense_segment[type.size_class]
    slot = pop_free_slot(seg) or seg.next_fresh_slot++
    d = seg.dense_count++
    seg.slot_to_dense[slot] = d
    seg.dense_to_slot[d]    = slot
    return make_handle(seg.idx, slot, seg.gen[slot])
```

### 2.5 Read / resolve path

```
resolve(h):
    seg = segs[h.seg_idx]
    if seg.gen[h.slot] != h.gen:  return STALE      // detected, never dereferenced
    d = seg.slot_to_dense[h.slot]
    return &seg.payload[d]
```

Cost: two dependent loads (generation + slot_to_dense), both from compact side arrays that stay cache-resident under hot use. **[A]** Inside a pin scope the result is cached, so loops pay this once. Raw pointers into bump memory skip resolution entirely (no handle) when `LastUseLifetime` proved the pointer does not outlive relocation (§3, §4.6).

### 2.6 Write path
Writes go to `payload[d]` through the resolved pointer. Handles are immutable; generations change only on free.

### 2.7 Release / reclaim paths

**Bump release (`SegDec`):**

```
release_bump(ptr, size):
    seg = seg_of(ptr)                 // address mask: ptr & ~(SEG_SIZE-1)
    if current_thread == seg.owner:   seg.live_bytes -= size
    else:                             seg.remote_free.push({ptr,size})   // owner drains in batches
    if seg.state == Sealed and seg.live_bytes == 0:
        reclaim_segment(seg)          // O(1): reset bump_off, state=Free
```

Individual bump objects are never reused; only the **whole segment** is recycled when `live_bytes == 0` (or compacted by `TileCompact` when sparse). **[A]**

**Dense release:**

```
free_dense(h):
    seg = segs[h.seg_idx]
    if seg.gen[h.slot] != h.gen: return DOUBLE_FREE_DETECTED
    d = seg.slot_to_dense[h.slot]
    last = --seg.dense_count
    // swap-remove keeps payload contiguous
    move payload[last] -> payload[d]
    s_last = seg.dense_to_slot[last]
    seg.slot_to_dense[s_last] = d
    seg.dense_to_slot[d]      = s_last
    seg.slot_to_dense[h.slot] = INVALID
    seg.gen[h.slot]++                 // invalidates all outstanding handles
    push_free_slot(seg, h.slot)
```

Swap-remove **moves an object** within the segment; this is legal because outside pin scopes mutators hold handles, not pointers. Pinned or in-scope readers: the swap defers (queue) until the epoch advances. **[A]**

**Cross-thread free.** Non-owner threads push to the segment's MPSC `remote_free` queue; the owner drains it on its next allocation slow path or safepoint. This avoids cross-core writes to hot allocator state. The cost is delayed reuse. **[A]**

### 2.8 Failure and edge cases

| Case | Behavior |
|------|----------|
| **Generation wrap** | On `gen` reaching max, the slot is **retired** (never reused) until its segment is rebuilt by `TileCompact`. Leaks at most one slot per 8.4 M reuses. |
| **Large object** (> SEG_SIZE/4) | Allocated in a Large Object Space with its own mmap'd region and a one-entry handle. Never bump-allocated. |
| **Single survivor pins a segment** | A long-lived object in a bump segment keeps `live_bytes > 0`. Mitigation: route by `lifetime_hint`, so Long objects get their own segment class; residual cases go to `TileCompact`. |
| **Thread exits with Open segment** | Segment sealed; remaining live bytes tracked; drained by remote-free queue. |
| **Underfilled thread-local segments** | Bounded waste: ≤ `threads × classes × SEG_SIZE`. Optionally use smaller segments for low-activity threads. |
| **Stale handle** | `resolve` returns `STALE`; policy per build: trap (debug), return null (release), or invoke a fallback (e.g., `MasterSlaveMM` status check). |
| **Free of pinned slot** | Deferred until pin released (epoch). |
| **NUMA** | Segments are bound to the allocating thread's node (first-touch). Cross-node frees go through the remote queue. |

### 2.9 Trade-offs

| Benefit | Cost |
|---------|------|
| O(1) alloc, no per-object header on bump path **[A]** | No individual reuse on bump path; footprint depends on segment-level death |
| O(1) bulk free of dead segments **[A]** | Mixed-lifetime pollution (mitigated, not eliminated) |
| Dense iteration over pools **[A]** | Extra load per handle resolve; swap-remove moves objects |
| Stale-handle detection **[A]** | 23-bit generation (wrap policy), side-table memory |
| Hugepage-friendly (reduces TLB pressure) **[A]** | Huge pages can increase internal fragmentation and make `madvise` returning coarser |

---

## 3. `LastUseLifetime` — Last-Use Causal Lifetime

### 3.1 Purpose
Release memory at the earliest point the compiler can *prove* no further use is causally possible, rather than at scope end (RAII/ownership) or at some later collection (tracing). No lifetime annotations required.

### 3.2 Inputs
- Typed IR in SSA/CFG form.
- Alias information: points-to or region analysis (flow-sensitive within function; summaries across functions).
- **Function effect summaries** for each parameter and return value (§3.6).
- Type attributes: `PureMemory` (no observable destructor effect) vs `Effectful` (locks, files, sockets, user-visible destructors).

### 3.3 State: the Causal Lifetime Graph (CLG)

Nodes:
- `Alloc(v)` — allocation of value `v`.
- `Use(v, site)` — read, write, call-argument use, or field projection.
- `Derive(v → w)` — `w` is an alias, borrow, field/interior reference, or copy of a reference to `v`.
- `Escape(v, sink)` — `v` stored into the heap, passed to unknown code, returned, captured by a closure, or sent across a thread boundary.

Edges encode causal order (CFG order, and derive relationships). Define the **alias closure** `A(v)` = `v` plus all values reachable by `Derive` edges.

### 3.4 Computation

**Step 1 — Backward liveness over alias closures.** For each program point `p` and each allocation `v`, compute `Live(v, p)` = there exists a path from `p` to a `Use` of any member of `A(v)`.

**Step 2 — Release edges.** A *release edge* for `v` is a CFG edge `(p → q)` where `Live(v,p)` is true and `Live(v,q)` is false. If `v` is dead at entry to a block with several predecessors where it was live on some and dead on others, split the critical edge so the release executes only on the path where `v` just died. No runtime drop flags are needed. **[A]**

**Step 3 — Classification (the "release kind" tag, attached to the allocation site):**

| Kind | Condition | Action at release edge |
|------|-----------|------------------------|
| `Free` | Dense-pool object, no escape | `free_dense(h)` |
| `SegDec` | Bump object, no escape | `live_bytes -= size` (batched, see §3.5) |
| `Defer` | Owned by a policy object (see `MasterSlaveMM`) or release must not run on this thread | Enqueue to Master/worker |
| `Escaped` | Escapes to heap structure/unknown code/another thread | No compiler release; demoted to a runtime-managed path (`MasterSlaveMM`, `CTMM`, or the collector of `JIT-GC Co-Design`) |
| `Scoped` | `Effectful` type | Release at scope end (RAII semantics preserved) |

**Step 4 — Lifetime hint synthesis.** The distance in IR between alloc and last use (and whether it crosses a loop/call boundary or a request boundary) yields `Short/Medium/Long` hints that route allocations to the right segment class in `SegGenPool`. **[A]**

### 3.5 Batching releases
Releasing at every last use can add instruction overhead. Coalescing rules:
- Adjacent `SegDec`s on the same segment within one basic block are summed into one atomic/plain add.
- Releases in a loop body whose objects were allocated in the loop are replaced by **one segment-region reset at loop exit** if the loop allocates into a dedicated region (an "iteration arena"). **[A]**
- Release code is placed on cold edges where possible, to avoid disturbing hot instruction layout.

### 3.6 Interprocedural summaries

Each function exports, per parameter `p`, an effect from the lattice:

| Effect | Meaning |
|--------|---------|
| `Consume` | Callee releases `p` (caller must not use afterwards) |
| `Borrow` | Callee uses `p` only during the call; caller keeps ownership |
| `Capture(sink)` | Callee stores `p` into `sink` (heap object, global, other thread) → at call site, `p` becomes `Escaped` unless `sink` is proven local |
| `Unknown` | Opaque/dynamic/FFI; conservative: `Escaped` |

Summaries are inferred bottom-up over the call graph; recursion handled by fixpoint. Dynamic dispatch uses the join over possible targets (or a speculative devirtualization guard in a JIT).

### 3.7 Read, write, reclaim paths
`LastUseLifetime` has no runtime read/write path of its own; it **emits** release actions into the other techniques' reclaim paths (`SegGenPool` `Free`/`SegDec`; `MasterSlaveMM` `Defer`). Its runtime cost is the emitted release instructions. Its "failure" path is `Escaped`.

### 3.8 Failure and edge cases

| Case | Behavior |
|------|----------|
| **Imprecise alias info** | Alias set larger than reality → release later than optimal (still safe). Never earlier. Soundness rule: *when uncertain, assume alive*. |
| **Exceptions / unwinding** | Each potentially-throwing call has a landing pad; releases for values live at that call are emitted in the pad. |
| **Async/coroutines** | Values live across suspension points are fields of the coroutine frame; last use is computed on the state machine; frame release follows the same analysis. |
| **Threads / sharing** | Sending a value to another thread is `Escape`. Either ownership transfers (`Consume` + remote release) or it is `Escaped`. |
| **Unsafe/FFI raw pointer** | `Unknown`; object is pinned or `Escaped`. |
| **Observable destructors** | `Effectful` types are `Scoped`; otherwise release-at-last-use would reorder visible side effects (unlock timing, file close). |
| **Debuggability** | Values may be freed before the source-level scope ends, so debug builds can keep scope-end semantics (a `Scoped` mode), poisoning freed memory to catch analysis bugs. |
| **Interior references** | `&v.field` is a `Derive`; `v` stays live while the derived reference does. With `LayoutMorph`, interior references into morphable slabs must be pin-scoped (§5.8). |

### 3.9 Trade-offs

| Benefit | Cost |
|---------|------|
| Smaller live ranges → lower footprint **[A]** | Analysis cost; more release sites |
| No tracing needed for provable values **[A]** | Precision limited by alias analysis; conservative fallback |
| No lifetime annotations **[A]** | Release order differs from scope order (observable only for `Effectful`, which are excluded) |
| Produces lifetime hints for placement **[A]** | Hints are predictions; wrong hints cost locality, not correctness |
| Accepts shapes a borrow checker rejects (shared/cyclic data fall to `Escaped` runtime paths instead of being compile errors) | Those shapes lose compile-time guarantees; safety relies on handles/generations/runtime policy |

**Relationship to ownership/borrow checking.** `LastUseLifetime` is a *dataflow optimization with sound fallback*, not a type discipline. Programs that fit single-ownership get zero-runtime-overhead release; programs that don't still compile and fall back to runtime-managed handling. The cost of that flexibility is that "compiles" no longer implies "no runtime reclamation machinery." **[A]**

---

## 4. `TileCompact` — Incremental Tile Compaction

### 4.1 Purpose
Repair fragmentation and shrink RSS without long pauses, by evacuating sparse fixed-size tiles in small, budgeted, resumable steps.

### 4.2 Inputs
- Tile metadata: `live_bytes`, `live_objs`, `pinned_count`, `age`, `hotness`.
- Budget per step: `B_bytes` (e.g., 64–256 KiB copied) and `B_us` (e.g., 20–100 µs).
- Triggers: fragmentation `F_ext` above threshold, free-segment pool below watermark, RSS above target, or idle-core availability.

### 4.3 State and metadata

```c
struct TileMeta {
    atomic<u8>  state;     // Active, Sealed, Selected, Evacuating, Quiescing, Free
    u32         live_bytes;
    u32         live_objs;
    atomic<u16> pinned;    // pin count; >0 blocks full evacuation
    u32         src_cursor;// resumable copy cursor (object index/offset)
    u32         dst_tile;  // current destination
    u64         retire_epoch;
};
// Object relocation record (handle-based objects): slot_to_ptr entry updated atomically.
// Raw-pointer objects (no handle): forwarding word in object header, or excluded.
```

### 4.4 Selection policy

```
score(tile) = reclaimable_bytes(tile) / (copy_cost(tile) + penalty(tile))
reclaimable_bytes = TILE_SIZE − live_bytes
copy_cost         = live_bytes + k · live_objs      (per-object overhead)
penalty           = ∞ if pinned > 0 for all-objects-required reclaim
                    + w · hotness   (avoid moving hot data; prefer cold, sparse tiles)
```

Candidates are bucketed by live ratio (e.g., 0–10%, 10–25%, 25–50%) so selection is O(1). Tiles with live ratio above a cut-off (e.g., > 70%) are never selected: copy cost exceeds benefit. **[A]**

### 4.5 Phases

```
Select → Reserve → Copy → Publish → Quiesce → Release
```

1. **Select:** pick best-scoring Sealed tile; CAS state `Sealed → Selected`.
2. **Reserve:** obtain a destination tile (or the open tail of one) with enough free space. New allocations never go into a tile being evacuated.
3. **Copy (budgeted):** for each live object (found via the tile's live bitmap in the side table, not by scanning the heap):
   - Acquire the object's *move lock*: set `moving` bit in side-table status via CAS (fails if pinned).
   - `memcpy` to destination (bump in destination tile).
   - Record `(old → new)` in a per-step relocation buffer.
   Stop when `B_bytes` or `B_us` is exhausted; save `src_cursor` (resumable).
4. **Publish:** for each moved object, atomically update the **handle indirection** (`slot_to_ptr` or `slot_to_dense` target) with a release store/CAS: `old → new`. After publish, a resolve sees the new location.
5. **Quiesce:** the source tile enters `Quiescing` with `retire_epoch = global_epoch`. Readers who resolved the *old* pointer before publish may still be reading it. The tile cannot be freed until every thread has passed a safepoint/epoch boundary beyond `retire_epoch` (EBR grace period). **[A]**
6. **Release:** when grace elapses, the tile returns to the free pool (optionally `madvise(DONTNEED)` or kept hot to avoid refault, depending on policy).

### 4.6 Mutator interaction (the protocol)

**Pin scope.** Code that needs a raw pointer calls `pin(h)`: resolves the handle, increments the object's (or tile's) pin count, and returns a pointer valid until `unpin`. While `pinned > 0` the object is not moved. For short scopes the compiler uses an **epoch guard** instead: enter epoch, resolve, use, exit epoch; the old copy stays readable until grace elapses, but **writes** must go through the `resolve_mut` path (below).

**Read during evacuation.** Always safe: either old location (still intact until Release) or new location (after Publish). **[A]**

**Write during evacuation.** The danger is a write to the *old* copy after it was copied but before Publish, which would be lost.
- `resolve_mut(h)` checks the object's `moving` bit. If set, the writer either (a) waits briefly for Publish, (b) *helps* by completing that object's move and resolving to the new location, or (c) aborts the move (clears `moving`, the copy is discarded). Policy (b) bounds mutator wait to one object copy. **[A]**
- Alternative: **copy-then-validate** with a per-object version counter: copier reads version `v0`, copies, re-reads version; if changed, retry or skip this object this round. Suited to rarely written objects.

**Allocation during evacuation.** New objects go to non-selected tiles; the evacuating tile is closed to allocation.

### 4.7 Pinned and non-relocatable objects
- Objects with live pins, FFI-exposed addresses, or without handle indirection are marked non-relocatable. A tile with such objects can be *partially* evacuated (move the rest) but not freed; it is tagged `PinnedResidue` and its live bytes counted in a **pinned-residue** statistic (a visible fragmentation floor).
- Raw-pointer objects without handles require a header **forwarding word** plus a read barrier: this is the expensive variant and is **not** the default. The default contract is handles-or-pin. **[A]**

### 4.8 Failure and edge cases

| Case | Behavior |
|------|----------|
| **Mutator write conflict** | Help-move or abort that object's move; never block unboundedly. |
| **Destination tile exhausted mid-step** | Reserve another destination; cursor resumable. |
| **Evacuation can't keep up with fragmentation** | Raise priority: more background workers; as last resort, grow heap (reported as `compaction_deficit`). |
| **Pinned residue** | Tile stays; counted; may become eligible when pins drop. |
| **Epoch stall (thread blocked, never reaches safepoint)** | Quiesce is delayed; tile held. Mitigation: blocked threads are marked quiescent while in blocking calls. |
| **Hot object moved** | Penalty term and hotness metadata discourage; if moved, it incurs one copy plus cache misses on next access. |
| **Crash consistency** | Not a goal (in-memory only). |

### 4.9 Trade-offs

| Benefit | Cost |
|---------|------|
| Pause per step bounded by `B_us` **[A]** | Total copy bandwidth consumed; competes with mutator memory bandwidth **[A]** |
| RSS can return to the OS **[A]** | Handle indirection required; pinned residue limits gains |
| Survivors packed densely → locality **[A]** | Copying evicts mutator's lines from cache (pollution) **[A]**; mitigated by non-temporal stores for cold data **[H]** |
| Can piggyback layout change (§5) | Concurrency protocol complexity (move lock, help-move, EBR) |

---

## 5. `LayoutMorph` / `ColSlab` — Adaptive Layout Morphing & Columnar Slabs

### 5.1 Purpose
Choose, and change at runtime, the physical layout of record collections to match how they are actually accessed, maximizing the bytes-useful ratio `U` (Paper 00 §2.2) and exposing SIMD-friendly contiguity.

### 5.2 `ColSlab` — the container

A `ColSlab<T>` stores up to `CAP` records of struct type `T` with fields `f0..fk`. Handles are stable `(slab, index, gen)`.

```c
struct ColSlab {
    u32   type_id;
    u8    layout_tag;       // AoS | SoA | AoSoA(B) | HotColdSplit
    u32   layout_epoch;     // bumped on every morph
    u32   count, cap;
    u64*  valid_bitmap;     // 1 bit per slot
    u32*  gen;              // per-slot generation
    void* col[K];           // per-field column base pointers (SoA / AoSoA)
    // varlen fields: (offset,len) in a side arena; columns remain fixed-width
};
```

Rules:
- Each column base is **64 B aligned**, and column length is padded to a multiple of the SIMD width so loops need no scalar tail for full blocks. **[A]**
- Columns written by *different threads* are separated by ≥ 64 B (or 128 B on prefetcher-pair platforms) to avoid false sharing. **[A]**
- Variable-length fields (strings, blobs) are stored as `(offset, len)` pairs into a per-slab varlen arena; the arena is managed by `SegGenPool` bump segments.
- Deletion: clear valid bit, `gen++`, push slot onto freelist. (Dense compaction of a slab, if wanted, is performed as swap-remove like §2.7, updating the handle index.)

**Access (SoA).** Field read: `col[f][idx]`. Whole-record gather (needed for AoS-like access): `k` loads from `k` columns (k cache lines), which is the cost the profile decision weighs.

### 5.3 Layout alternatives

| Layout | Structure | Best for | Weak for |
|--------|-----------|----------|----------|
| **AoS** | Records contiguous | Point lookups, whole-record access, few fields | Narrow field scans (low `U`) |
| **SoA** | One array per field | Narrow scans, SIMD, compression | Whole-record access (k lines), insertion |
| **AoSoA(B)** | Blocks of `B` records; inside, SoA | SIMD with block locality; mixed workloads | Complexity; index arithmetic |
| **HotColdSplit** | Hot fields in a dense array, cold in a separate array (or lower tier) | Few hot fields, many cold ones; works with `CTMM` | Two lookups for full record |

### 5.4 `LayoutMorph` — the profiler and decision procedure

**Inputs.** Access-site descriptors emitted by the compiler: for each site `s`, the field set `F_s`, access shape (`Scan` over many records vs `Point` at one record), and (via sampled counters) frequency `f_s`.

**Profile state (per type):**
- Co-access matrix `M[i][j]` = weighted count of sites touching fields `i` and `j` together.
- Per-site `(F_s, shape_s, f_s)` table.
- Sampling: counters are updated every 1/N executions of an access-site (e.g., N = 64–1024) to bound overhead. **[A]** Sites in hot compiled loops use per-loop counters at loop exit rather than per-iteration counters.

**Cost estimate (cache lines touched per record), per layout:**

```
Scan site s, record stride S, fields F_s:
    AoS   : ⌈S/64⌉                    (record lines streamed; for S<64, S/64 amortized)
    SoA   : Σ_{f∈F_s} size(f)/64      (only touched columns)
    AoSoA : ≈ SoA cost + small block overhead
Point site s:
    AoS   : ⌈ span(F_s)/64 ⌉ (+1 if straddling)      (usually 1–2 lines)
    SoA   : |F_s| lines (one per touched column)
    Split : (hot in F_s ? 1 : 0) + (cold in F_s ? 1 : 0) lines

TotalCost(layout) = Σ_s f_s · cost_s(layout) · n_records_touched_s
```

**Decision.** Select `argmin TotalCost`. Morph only if

```
TotalCost(current) / TotalCost(candidate) ≥ 1 + θ      (θ ≈ 0.3, hysteresis margin)
AND  time_since_last_morph ≥ T_dwell
AND  estimated_morph_cost  <  expected_gain · horizon
```

with `horizon` the profile's recent window. The hysteresis margin and dwell time prevent flapping at a workload phase boundary. **[A]**

### 5.5 Morph execution

Morphing is expensive, so it is **piggybacked on `TileCompact` evacuation**: when a slab's tile is being copied anyway, the copy routine writes records in the new layout.

```
morph_during_evacuation(slab_tile_src → dst, new_layout):
    for each live record r in src:            // from valid bitmap
        write fields of r into dst per new_layout
    publish: update slab index to dst; slab.layout_epoch++
```

For small slabs (or when no compaction is pending), a **standalone morph** is run as a background job with the same publish/quiesce protocol as §4.5.

### 5.6 Accessor specialization and re-linking

Compiled code that accesses `ColSlab` fields is specialized per layout:
- **JIT:** emit accessors guarded on `slab.layout_epoch == cached_epoch`; on mismatch, deoptimize/relink to the accessor for the new layout.
- **AOT:** monomorphize the loop per layout variant (e.g., AoS and SoA versions) and select via a dispatch at loop entry; check `layout_epoch` at safepoints (or loop entry only).
- Hot loops hoist `col[f]` base pointers into registers inside a pin scope, so the per-iteration cost is the same as hand-written columnar code. **[A]**

### 5.7 Read / write / reclaim summary

| Path | Behavior |
|------|----------|
| **Read** | `h → (slab, idx, gen check) → col[f][idx]` (SoA) or `rec[idx].f` (AoS) |
| **Write** | Same; `resolve_mut` cooperates with morph/evacuation (§4.6) |
| **Reclaim** | Per-slot free (valid bit, `gen++`); slab or tile freed via `TileCompact` when sparse |
| **Evict** | Cold column or cold part of a `HotColdSplit` may be handed to `CTMM` (Crystalline tier) |

### 5.8 Failure and edge cases

| Case | Behavior |
|------|----------|
| **Interior pointer `&rec.field`** | Invalid after morph. Rule: interior borrows must be taken inside a pin scope; a slab with live pins is **not morphable** (morph is deferred). In a SoA layout `&rec.field` points into a column, so the pointer also depends on `idx`; swap-remove invalidates it. |
| **Variable-length fields** | Stay in the varlen arena; morph copies `(offset,len)`, not payloads. |
| **Profile noise / phase changes** | Hysteresis + dwell; a decision requires a window of consistent samples. |
| **Mixed workloads (both scans and point lookups)** | AoSoA or `HotColdSplit` is the compromise; cost model picks if it beats both. |
| **Many types, little data** | Per-type metadata overhead dominates; types with < `N_min` records never morph. |
| **Multi-writer slabs** | Column separation to avoid false sharing; morph requires quiescence (EBR) for writers, so write-heavy slabs morph rarely. |
| **Precision of the cost model** | It ignores prefetcher effectiveness and TLB effects; treat as a ranking heuristic, validate per workload (**[H]**). |
| **Stale accessor code** | Epoch guard prevents use of old-layout accessors on a new-layout slab. |

### 5.9 Trade-offs

| Benefit | Cost |
|---------|------|
| `U → 1` on narrow scans; SIMD-friendly **[A]** | Whole-record access costs `k` lines in SoA **[A]** |
| Layout adapts to phase **[A]** | Profiling overhead (sampled); morph copy cost; code duplication |
| Free morph during compaction **[A]** | Morph is delayed until a compaction opportunity or a dedicated job |
| Stable handles keep external references valid | Interior pointers must use pin scopes; more programmer/compiler discipline |

---

## 6. Combining the Four (without yet involving Papers 02–04)

### 6.1 Dataflow of one allocation through the structural/lifetime stack

```
Source: new Particle{pos, vel, mass, tag, ...}
   │
   ▼  LastUseLifetime: escape? last use? ─────────── Short, no escape → bump, release = SegDec
   │                                              │
   │                              Long, collection of many records → ColSlab<Particle>
   ▼
SegGenPool: segment class by lifetime_hint
   ├─ bump segment (Short)         ── segment dies when live_bytes→0
   └─ dense/ColSlab pool (Long)    ── handle (seg,slot,gen); iteration is dense
   ▼
Steady state: tile sparse? ── TileCompact evacuates ── piggyback LayoutMorph if profile says so
```

### 6.2 What each technique contributes to the others

| Provider → Consumer | Contribution |
|---------------------|--------------|
| `LastUseLifetime` → `SegGenPool` | Release kind (`Free`/`SegDec`) and `lifetime_hint` for segment-class routing |
| `LastUseLifetime` → `LayoutMorph` | Static field-set per access site (`F_s`, `shape_s`) for the profile |
| `SegGenPool` → `TileCompact` | Tile/segment metadata, live bitmaps, handle indirection to update |
| `TileCompact` → `LayoutMorph` | The copy opportunity for free morphs |
| `LayoutMorph`/`ColSlab` → `SegGenPool` | Dense, handle-addressed, columnar container as a pool variant |
| `SegGenPool`/`TileCompact` → `LastUseLifetime` | Runtime counters on mispredicted hints feed back into site hints |

### 6.3 Interaction hazards (full treatment in Paper 05)

| Hazard | Techniques | Resolution |
|--------|-----------|------------|
| Release at last use while a tile is mid-evacuation | `LastUseLifetime` × `TileCompact` | Release is a status/live-count update in side tables, which remain valid across relocation; moving object's `moving` bit makes release wait for Publish or cancel the move |
| Swap-remove vs relocation vs morph all "move objects" | `SegGenPool` × `TileCompact` × `LayoutMorph` | One shared move protocol (`moving` bit + EBR); only one mover per object |
| Interior borrows | `LastUseLifetime` × `LayoutMorph` | Pin scopes; unpinned handle-only access in morphable slabs |
| Hint misprediction pins segments | `LastUseLifetime` × `SegGenPool` | `TileCompact` repairs; feedback adjusts site hints |

### 6.4 Minimal adoption path
1. `SegGenPool` alone (bump + dense pool, handles).
2. Add `LastUseLifetime` release kinds.
3. Add `TileCompact` (requires handle indirection from step 1).
4. Add `ColSlab` (static columnar layout), then `LayoutMorph` (adaptive).

Each step is independently valuable; the combination is not required. **[A]**

---

## 7. Validation Hypotheses (to be tested per Paper 05)

| ID | Hypothesis | Falsified if |
|----|------------|--------------|
| H1.1 | Bump + segment-death reclaim lowers allocator p99 vs a general-purpose `malloc` on request-scoped workloads | p99 not improved at equal throughput |
| H1.2 | `LastUseLifetime` reduces peak live bytes vs scope-end release on workloads with long scopes and early last use | Peak footprint not reduced beyond noise |
| H1.3 | `TileCompact` reduces RSS growth over long runs with churn while keeping per-step pause ≤ `B_us` | RSS not reduced, or step latency exceeds budget |
| H1.4 | `ColSlab` improves narrow-field scan throughput vs AoS by ~`1/U` up to memory-bandwidth limits | No speedup on narrow scans, or slowdown beyond whole-record penalty on mixed loads |
| H1.5 | `LayoutMorph` with hysteresis outperforms the best *static* layout on phase-changing workloads, with < X% profiling overhead | No gain vs best static, or flapping observed |

No numbers are claimed.

---

## 8. Open Questions

- `OPEN:` Best default for `θ` (hysteresis margin) and `T_dwell`; likely workload-dependent.
- `OPEN:` Handle width: 64-bit handle vs 32-bit compressed handles for smaller structures.
- `OPEN:` Whether `LastUseLifetime` alias analysis should be whole-program (best precision) or summary-based only (better compile time).
- `OPEN:` Non-temporal stores during evacuation: benefit vs losing cache warmth for subsequently-hot objects.
- `OPEN:` Interaction of generation wrap policy with very high churn pools (retired-slot accumulation).
