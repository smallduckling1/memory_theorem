# Paper 00 — Abstract, Problem Statement, and Technique Taxonomy

Claim tags: **[A]** architectural · **[H]** hypothesis, needs benchmark · **[M]** measured (none yet).

## Abstract

Mainstream memory-management models each optimize one axis and externalize the rest. Manual `malloc`/`free` gives control but no relocation and no safety. Tracing garbage collection gives safety and relocation but spends memory-latency-bound traversal, barriers, and headroom. Static ownership gives safety and determinism but constrains program shape and says nothing about placement, density, or residency. All three assume a flat memory in which an object is either allocated or freed.

We present a modular set of named techniques that give the compiler, runtime, and hardware-aware scheduler the information and the means to act on it: **`SegGenPool`** (segmented bump + generational dense pool), **`LastUseLifetime`** (release at last causal use), **`TileCompact`** (incremental, budgeted compaction), **`LayoutMorph`/`ColSlab`** (adaptive layout and columnar slabs), **`MasterSlaveMM`** (policy-bearing masters with background reclamation), **`CTMM`** (temperature- and weight-driven tiering: Plasma, Liquid, Crystalline, Sublimation), and the **`JIT-GC Co-Design`** trio **`JITDestruct`**, **`CacheSweep`**, **`StallHarvest`**.

Each technique is specified independently (inputs, metadata, read/write/reclaim paths, failure modes). The composition is optional and layered. This paper establishes the problem, a cost model, design principles, and the taxonomy. Performance claims are architectural or hypotheses until validated per `paper/05`.

---

## 1. Problem Statement

**Goal.** Reduce, simultaneously and with explicit trade-offs, the following costs of memory management in long-running, latency-sensitive, and data-intensive programs:

1. **Tail latency** (p99/p99.9 jitter from allocator slow paths, collector pauses, compaction, page faults, drop cascades).
2. **Cache and TLB inefficiency** (pointer chasing, wasted bytes per line, scattered placement).
3. **Fragmentation** (external and internal; inability to relocate).
4. **Memory bloat** (RSS ≫ live bytes due to headroom, headers, late release, no tiering).
5. **Engineering cost** (safety bugs, tuning, ownership friction, hand-rolled pools/caches).

**Constraint.** Preserve what existing models do well: low average allocation cost (manual), safety and cyclic-structure ergonomics (GC), and determinism with zero tracing for provable cases (ownership).

**Non-goals.** A universal allocator; a replacement for OS virtual memory; hard-real-time guarantees; benchmark claims in this paper.

---

## 2. Cost Model

We use a deliberately simple model so that each technique's *mechanism* can be mapped to the term it reduces. Numbers are symbolic.

### 2.1 Access cost
For an access stream, expected cost per access:

```
C_access = Σ_l  h_l · L_l   +  m_TLB · L_walk  +  p_fault · L_fault
```

where `h_l` is the fraction of accesses served at cache level `l`, `L_l` its latency, `m_TLB` the TLB miss rate, and `p_fault` the page-fault rate. **[A]**

For a dependent chain of `n` loads (pointer chasing) with miss probability `q` per hop:

```
C_chain ≈ n · ( (1−q)·L_hit + q·L_miss )          (no overlap possible)
```

For an independent stream of `n` accesses with memory-level parallelism `P`:

```
C_stream ≈ n · L_miss / P                         (overlap possible)
```

**Implication.** Converting dependent chains into independent streams (or into contiguous layout) changes the divisor from 1 to `P`. This is the mechanism behind `LayoutMorph`/`ColSlab` (contiguity) and `CacheSweep` (prefetch-driven overlap). **[A]**

### 2.2 Bytes-useful ratio
For a scan reading field set `F` of records with stride `S`:

```
U = (Σ_{f∈F} size(f)) / S          (fraction of fetched bytes actually used; ≤ 1, per line granularity)
```

AoS has `U` small for narrow scans; columnar approaches `U → 1`. **[A]**

### 2.3 Fragmentation and bloat

```
F_ext   = 1 − (largest_free_block / total_free)           (external)
F_int   = wasted_bytes_in_allocated_blocks / allocated     (internal)
B       = RSS / live_bytes                                 (bloat factor, ≥ 1)
```

`B` decomposes into headroom (GC), headers and alignment, retained free pages, late release (scope-end), and unevictable cold data. Each technique targets a named component. **[A]**

### 2.4 Tail latency
Let a request's latency be `base + Σ events`. Event classes: allocator slow path, collector pause or barrier slow path, compaction step, page fault, thaw/hydration, cross-core cache-line transfer, lock wait. The p99 is dominated by the *union of rare events*, so reducing average cost does not reduce the tail unless each rare event is **bounded**. Hence the design principle of **bounded steps** (§4). **[A]**

### 2.5 Engineering cost (qualitative)
Counted as: classes of bug the model permits (use-after-free, double free, leak, data race on lifetime), tuning knobs exposed, and constraints on program shape imposed. Not quantified here.

---

## 3. Cost Analysis of Existing Paradigms

| Axis | Manual `malloc`/`free` | Tracing GC | Static borrow/ownership |
|------|------------------------|------------|--------------------------|
| **Tail latency** | Arena contention, coalescing, `mmap`/`madvise`, first-touch faults | STW pauses; or barrier + allocation-stall cost for concurrent; post-collection cache cold-start | Allocator slow paths; scope-end drop cascades |
| **Pointer chasing** | Scattered nodes; order of allocation decoupled from use | Marking is a chase; layout depends on collector | Same as malloc unless arenas/indices used (leaving the model) |
| **Fragmentation** | Cannot relocate (raw pointers) | Non-moving: as malloc; moving: copy reserve or barriers | Cannot relocate |
| **Bloat** | Headers, thread caches, unreturned pages | Headroom (commonly multiples of live size), per-object header bits | As malloc, plus scope-held values |
| **Safety / dev cost** | UAF, double free, leaks | Tuning, finalizers, pooling hacks, native pinning | Annotation spread, graph/shared-state friction, `unsafe` |
| **Placement awareness** | None | Generational at best | None |
| **Residency awareness** | None (OS paging only) | None | None |

**Observation 1.** No existing paradigm reasons about *residency* (what should be hot, compressed, regenerable, or dropped). That decision is delegated to application-level caches or the OS pager, both blind to object semantics. **[A]**

**Observation 2.** Placement and layout are decided at allocation time and never revisited, even though access patterns change by phase. **[A]**

**Observation 3.** The compiler computes liveness already, but the information is discarded at the allocation boundary; GC then rediscovers it dynamically by traversal. **[A]**

**Observation 4.** Collectors run either at fixed trigger points or on separate threads, uncoordinated with the mutator's *microarchitectural* state, though stall cycles and idle cores are free capacity. **[A]**

---

## 4. Design Principles

| # | Principle | Consequence |
|---|-----------|-------------|
| P1 | **Use compiler knowledge before runtime discovery.** | `LastUseLifetime` before any tracing. |
| P2 | **Every maintenance action is bounded and resumable.** | `TileCompact` steps, `gc_micro_step()`, background batch sizes have hard caps. |
| P3 | **References are validated, not trusted.** | Generation-tagged handles; stale access is detected. |
| P4 | **Metadata lives beside, not inside, hot data when feasible.** | Side tables for status, marks, generation; avoids dirtying object lines. |
| P5 | **Policy is separated from mechanism.** | `MasterSlaveMM` and `CTMM` decide; workers execute. |
| P6 | **Hysteresis on every adaptive decision.** | Layout morph, tier transitions, compaction selection never flap. |
| P7 | **Each technique is independently adoptable.** | Compatibility matrix in `paper/05`; no technique requires all others. |
| P8 | **Fail toward the conservative path.** | Unprovable → escape to runtime-managed path; unsafe to move → pin; guard fails → deopt/handoff. |
| P9 | **Name the cost.** | Every technique declares its overhead and tail sources. |

---

## 5. Full Taxonomy of Named Techniques

### 5.1 By layer

```
Compile-time
 └─ LastUseLifetime ........................ release at last causal use

Allocation & Structure
 └─ SegGenPool ............................. segmented bump + generational dense pool

Relocation & Layout
 ├─ TileCompact ............................ incremental, budgeted compaction
 └─ LayoutMorph / ColSlab .................. adaptive layout + columnar slabs

Policy & Residency
 ├─ MasterSlaveMM .......................... policy masters, scoped slaves, background workers
 └─ CTMM ................................... Plasma / Liquid / Crystalline / Sublimation tiering

Collection (runtime + JIT)
 └─ JIT-GC Co-Design
     ├─ JITDestruct ........................ synthesized destructors, escape guards, handoff
     ├─ CacheSweep ......................... 64B side-table bitmaps, unrolled tracers, PREFETCH
     └─ StallHarvest ....................... PMU + control-flow prediction → bounded micro-steps
```

### 5.2 By cost axis targeted

| Technique | Tail latency | Chasing / cache | Fragmentation | Bloat | Dev cost |
|-----------|:-----------:|:---------------:|:-------------:|:-----:|:--------:|
| `SegGenPool` | ● | ● | ◐ | ◐ | ◐ (stale-handle detection) |
| `LastUseLifetime` | ◐ | ○ | ○ | ● | ● (no annotations needed) |
| `TileCompact` | ● (bounded steps) | ◐ | ● | ● | ○ |
| `LayoutMorph`/`ColSlab` | ○ | ● | ○ | ◐ | ◐ |
| `MasterSlaveMM` | ● (off-path reclaim) | ○ | ○ | ◐ | ● (central policy) |
| `CTMM` | ◐ (adds thaw tail) | ◐ | ○ | ● | ◐ |
| `JITDestruct` | ● | ◐ | ○ | ◐ | ○ |
| `CacheSweep` | ● | ● | ○ | ○ | ○ |
| `StallHarvest` | ◐ | ○ | ○ | ○ | ○ |

● primary target · ◐ secondary effect · ○ little or no effect. **[A]** (mechanism-based; magnitude is **[H]**)

### 5.3 By information source

| Source of knowledge | Techniques |
|---------------------|------------|
| Static analysis (types, liveness, aliasing) | `LastUseLifetime`, `JITDestruct` (via JIT), `LayoutMorph` (field sets) |
| Runtime access profile | `LayoutMorph`, `CTMM`, `TileCompact` (live ratios) |
| Application-declared policy | `MasterSlaveMM`, `CTMM` (weights, never-below-tier) |
| Hardware telemetry | `StallHarvest`, `CacheSweep` (prefetch tuning) |

### 5.4 Technique index (one line each, canonical names)

| ID | Name | Code name | Core idea |
|----|------|-----------|-----------|
| T1 | Segmented Bump + Generational Dense Pool | `SegGenPool` | Bump segments with O(1) bulk death; dense slots with generation-tagged handles |
| T2 | Last-Use Causal Lifetime | `LastUseLifetime` | Release at the last causal use across aliases |
| T3 | Incremental Tile Compaction | `TileCompact` | Bounded evacuation of sparse tiles |
| T4a | Adaptive Layout Morphing | `LayoutMorph` | Profile → choose AoS/SoA/AoSoA/hot-cold; morph during evacuation |
| T4b | Columnar Slabs | `ColSlab` | Per-field columns with stable handles |
| T5 | Master / Slave Memory Management | `MasterSlaveMM` | Policy in Masters; statuses `Alive`/`Deleted`/`Evicted`/`Expired`; background reclaim |
| T6 | Cognitive-Thermodynamic Memory Model | `CTMM` | Temperature × weight tiers with hysteresis; two-tier handles; transparent thaw |
| T7a | JIT-Synthesized Destructors | `JITDestruct` | Type-specialized reclaim, speculative escape guards, handoff |
| T7b | Speculative Cache-Line Sweeping | `CacheSweep` | Side-table bitmaps per cache line; unrolled prefetching tracers |
| T7c | Stall-Cycle Harvesting | `StallHarvest` | PMU-guided bounded `gc_micro_step()` |

---

## 6. Assumptions and Threat Model

- **Hardware:** cache-coherent multicore, 64-byte lines, virtual memory with 4 KiB pages and optional 2 MiB huge pages, optionally heterogeneous cores (P/E). PMU access available only for `StallHarvest`.
- **Runtime:** safepoints exist (or can be inserted); a code cache with the ability to patch or deoptimize exists for JIT-dependent techniques. Techniques `SegGenPool`, `LastUseLifetime`, `TileCompact`, `LayoutMorph`/`ColSlab`, `MasterSlaveMM`, `CTMM` do **not** require a JIT; AOT compilation with monomorphized variants is sufficient where noted.
- **Safety model:** the goal is detection of stale references (via generations) and avoidance of invalid moves (via pinning), **not** defense against a malicious in-process attacker.
- **Concurrency:** multiple mutator threads; background workers; memory ordering is acquire/release unless stated.

---

## 7. Roadmap of the Paper Suite

| Paper | Contents |
|-------|----------|
| `01` | `SegGenPool`, `LastUseLifetime`, `TileCompact`, `LayoutMorph`/`ColSlab` |
| `02` | `MasterSlaveMM` |
| `03` | `CTMM` |
| `04` | `JITDestruct`, `CacheSweep`, `StallHarvest` |
| `05` | Compatibility matrix, conflicts, overhead/failure modes, benchmark methodology, use cases |

---

## 8. Validation Stance

No result here is measured. Section 5.2's markings state which mechanism targets which cost; **magnitudes are hypotheses**. A technique is considered *supported* only when `paper/05`'s protocol yields a result with stated hardware, workload, baseline, and confidence interval, and *refuted* if it fails the pre-registered falsification criteria listed there.
