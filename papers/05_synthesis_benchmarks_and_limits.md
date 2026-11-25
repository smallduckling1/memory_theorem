# Paper 05 — Synthesis, Benchmarks, and Limits

Claim tags: **[A]** architectural · **[H]** hypothesis · **[M]** measured (none).
This paper is where the package is held accountable: what combines, what conflicts, what each technique honestly costs, how to test it, and when it would be refuted.

---

## 1. Technique Roster (canonical names)

| ID | Code name | Paper |
|----|-----------|-------|
| T1 | `SegGenPool` | 01 |
| T2 | `LastUseLifetime` | 01 |
| T3 | `TileCompact` | 01 |
| T4 | `LayoutMorph` / `ColSlab` | 01 |
| T5 | `MasterSlaveMM` | 02 |
| T6 | `CTMM` | 03 |
| T7a | `JITDestruct` | 04 |
| T7b | `CacheSweep` | 04 |
| T7c | `StallHarvest` | 04 |

---

## 2. Compatibility Matrix

Legend: **R** row *requires* column · **S** synergy (works better together) · **N** neutral/independent · **C** conflict or overlap needing a stated rule (see §3) · **—** same technique.
Read as: "row technique with column technique."

| ↓ row \ col → | SegGenPool | LastUse | TileCompact | LayoutMorph/ColSlab | MasterSlave | CTMM | JITDestruct | CacheSweep | StallHarvest |
|---|---|---|---|---|---|---|---|---|---|
| **SegGenPool** | — | S | S | S | S | S | S | S | N |
| **LastUseLifetime** | S | — | N | S | S (`Defer`) | N | **C** (static vs speculative overlap) | N | N |
| **TileCompact** | **R** (tiles/handles) | N | — | S (free morph) | **C** (move protocol) | **C** (move/tier protocol) | N | **C** (mark vs move) | S (copy slices) |
| **LayoutMorph/ColSlab** | **R** (handles, segments) | S | S | — | N | **C** (Crystalline columns) | N | N | N |
| **MasterSlaveMM** | S | S | **C** | N | — | **C** (Evicted vs Sublimation) | N | N | **C** (shared idle budget) |
| **CTMM** | S | N | **C** | **C** | **C** | — | N | N | **C** (shared idle budget) |
| **JITDestruct** | S | **C** | N | N | N | N | — | S (`OWNED` elision) | S |
| **CacheSweep** | S | N | **C** | N | N | N | S | — | **R** for micro-steps (bounded steps) |
| **StallHarvest** | N | N | S | N | **C** | **C** | S | **R** (needs bounded collector) | — |

**Hard requirements (dependency order):**
1. `TileCompact` ⇒ `SegGenPool` handles/tiles (or an equivalent handle indirection).
2. `ColSlab` ⇒ `SegGenPool` handles; `LayoutMorph` ⇒ `ColSlab`.
3. `StallHarvest` ⇒ a collector (or other background work) with **bounded, resumable steps**.
4. `CacheSweep` ⇒ per-type pointer maps and side-table bitmaps.
5. `JITDestruct` ⇒ a compiler with escape information and safepoint maps; the `OWNED`/`TRACED` bits live in `CacheSweep`-compatible side tables.
Everything else is independently adoptable. **[A]**

**Minimal viable subsets:**

| Goal | Subset |
|------|--------|
| Fast, safe allocation with stale-handle detection | `SegGenPool` |
| + earlier reclamation, no GC for provable cases | + `LastUseLifetime` |
| + long-run RSS control | + `TileCompact` |
| + data-intensive scans | + `ColSlab` → `LayoutMorph` |
| + cache/policy-driven services | + `MasterSlaveMM` and/or `CTMM` |
| + managed runtime with residual GC | + `JITDestruct`, `CacheSweep`, `StallHarvest` |

---

## 3. Conflicts and Overlaps (each with a decided rule)

### C1 — Three techniques "move objects" (`SegGenPool` swap-remove, `TileCompact` evacuation, `LayoutMorph` morph, plus `CTMM` promotion/demotion)
**Problem.** Concurrent movers could race on one object.
**Rule.** One shared **move protocol**: a `moving` bit in the object's side-table status (CAS-claimed), the pin-scope contract, EBR for old-copy lifetime. **Only the claimant may move**; others skip. Pinned objects are never moved. `resolve_mut` cooperates (help-move or abort, Paper 01 §4.6). **[A]**

### C2 — `MasterSlaveMM` `Evicted` vs `CTMM` `Sublimation`
**Overlap.** Both drop a payload and keep a descriptor for later rehydration.
**Rule.** *Ownership of the decision decides.* If an object belongs to a Master with an explicit policy, the Master decides `Evicted`; CTMM manages only tiers *above* (Plasma/Liquid/Crystalline) and **does not sublimate** that object unless the Master delegates. For objects with no Master, CTMM may sublimate if `regenerable`. A shared descriptor format is an `OPEN` item (Papers 02 §12, 03 §12). **[A]**

### C3 — `LastUseLifetime` (static) vs `JITDestruct` (speculative)
**Overlap.** Both release non-escaping objects early; one proves, the other speculates.
**Rule.** Static proof wins: a proven `Free`/`SegDec` is emitted unguarded. Only objects `LastUseLifetime` marks `Escaped` or "uncertain" are candidates for `SpecNoEscape` with guards/handoff. In AOT-only builds `JITDestruct` collapses to its destructor-synthesis half (no speculation). **[A]**

### C4 — `TileCompact` vs `CacheSweep` marking
**Problem.** Mark bits are keyed by address; relocation changes addresses.
**Rule.** Do not evacuate a tile while it is being concurrently marked, **or** copy `start`/`mark` bits with each relocated object. Default: phase exclusion (marking tiles are `NoEvacuate` until the cycle ends). Handle-addressed objects are unaffected (table update only). **[A]**

### C5 — `LayoutMorph` vs `CTMM` Crystalline
**Problem.** A morph to SoA changes the unit that can be compressed (columns vs records); Crystalline compresses payloads.
**Rule.** Tier at *column* granularity for `ColSlab` when layout is SoA/AoSoA (cold columns → Crystalline, hot columns stay Liquid/Plasma); tier at *record* granularity for AoS. A morph requires all affected parts to be Liquid or higher (thaw first); morphing a partially Crystalline slab is deferred. **[A]**

### C6 — `StallHarvest` vs background workers (`MasterSlaveMM`, `CTMM`, `TileCompact`)
**Problem.** Several actors claim the same idle time and the same cores.
**Rule.** One **idle-budget coordinator** divides harvested time and worker CPU among actors by debt-weighted priority, with a guaranteed floor for the collector (so allocation never stalls) and a ceiling on total background CPU (e.g., ≤ X% of cores). Reclaim/evict beats tier-demotion under memory pressure; tier-demotion beats compaction when pressure is low. **[A]**

### C7 — Handle indirection vs raw-pointer speed
**Problem.** Every access pays a resolve step.
**Rule.** Compiled code hoists resolves into pin scopes; the compiler may use raw pointers for `LastUseLifetime`-proven bump objects that are never relocated (e.g., in segments flagged `NoRelocate` for the object's proven lifetime). **[A]**

### C8 — Release-at-last-use vs in-flight relocation
Covered in Paper 01 §6.3: releases are side-table updates, valid across relocation; the `moving` bit makes release wait or cancel the move.

### C9 — `LastUseLifetime` reordering vs observable destructors
**Rule.** `Effectful` types keep scope semantics (`Scoped`). **[A]**

### C10 — Complexity budget
Every technique adds invariants. **Rule:** adopt along the subset ladder (§2); each addition must pass its own hypotheses (§7) before the next is enabled by default.

---

## 4. Honest Overheads and Failure Modes

### 4.1 Per-technique summary

| Technique | Steady-state overhead | New tail-latency source | Memory overhead | Worst case |
|-----------|-----------------------|--------------------------|-----------------|------------|
| `SegGenPool` | Handle resolve (2 loads) on pool objects; remote-free queues | Segment acquisition slow path; page faults on fresh segments | Side tables; underfilled thread-local segments (≤ threads × classes × segment) | One survivor pins a segment (needs `TileCompact`); generation wrap retiring slots |
| `LastUseLifetime` | Extra release instructions; compile-time analysis | None at runtime beyond release code | Code size | Imprecise aliasing → little gain (not a correctness issue); analysis time explosion on huge functions |
| `TileCompact` | Copy bandwidth; handle indirection; epoch pinning | Mutator help-move on write conflict (bounded by one object copy) | Destination tiles during evacuation | Pinned residue; compaction can't keep up with churn → heap growth |
| `LayoutMorph`/`ColSlab` | Sampled profiling; accessor guards; k-line gather for whole-record access in SoA | Morph quiesce for write-heavy slabs | Per-type profile, duplicated accessor code | Misprediction → worse layout; flapping if hysteresis mis-set |
| `MasterSlaveMM` | Status check + epoch guard on every access; queue traffic | Deferred-reclaim backlog; foreground help; hydrate on `Evicted` | Deferred payload memory; timer wheel/index | Backlog growth; worker starvation; expiry storms |
| `CTMM` | Slot-table load per access; access ring appends | **Thaw latency** on cold access | Slot table (16 B/object), descriptors, compression buffers | Thaw storm; wrong `regenerable` contract; thrash if dead band mis-tuned |
| `JITDestruct` | Side-table byte test on release; guards at cold escape points | Deopt on invalidated assumptions; handoff storms | Code cache; per-type routines | Speculation storm → recompilation churn |
| `CacheSweep` | Barrier on non-`OWNED` stores; marking CPU | Mark/sweep steps (bounded) | ~0.8% per bitmap (×2) | Prefetch pollution; mark-ring overflow on pathological graphs |
| `StallHarvest` | Poll sites; PMU reads | GC extending a critical path on misprediction | Per-site state | PMU unavailable → degraded; harvest supply below debt → forced slices dominate |

All entries are **[A]** (mechanism-derived). None are measured.

### 4.2 Cross-cutting failure modes

| Failure mode | Cause | Detection | Mitigation |
|--------------|-------|-----------|------------|
| **Epoch stall** | A thread never passes a safepoint (long loop, blocked in native code); EBR can't advance | Epoch lag metric | Mark blocked threads quiescent; safepoint polls in loops; watchdog; bound retire-list size |
| **Stale-handle bugs masked by `STALE`** | App ignores `STALE` results | Counter of `STALE` returns | Debug trap mode; lint `STALE` handling |
| **Metadata blow-up for tiny objects** | 16 B slot entry + side-table bits on 16-byte payloads | Metadata/payload ratio metric | Use bump/region for tiny short-lived objects; aggregate into `ColSlab` |
| **Cross-core traffic amplification** | Status words, queues, bitmaps, access rings | Coherence-miss PMU events | Sharding, batching, per-thread buffers |
| **Background CPU hidden cost** | Workers on E-cores or idle cores are "free" only if truly idle | Total CPU cycles and energy accounting | Charge worker CPU in all comparisons |
| **Feedback-loop oscillation** | `LayoutMorph`, `CTMM`, speculation recompiles reacting to each other's effects | Transition-rate metrics | Hysteresis everywhere; global rate limiters; freeze periods |
| **Correctness holes at technique boundaries** | Interactions (C1–C9) | Differential testing, stress with fault injection (random delays at safepoints, forced moves) | Shared protocols; verifier mode |
| **Operational complexity** | Many knobs | Config count | Defaults per workload class; autotune with caps |

### 4.3 Where each technique is expected to be a poor idea

| Technique | Poor fit |
|-----------|----------|
| `SegGenPool` | Programs allocating mostly huge objects; extremely memory-tight embedded targets |
| `LastUseLifetime` | Code dominated by dynamic dispatch/FFI where nearly everything is `Escaped` |
| `TileCompact` | Workloads with very low churn (nothing to compact); heaps dominated by pinned objects |
| `LayoutMorph`/`ColSlab` | Whole-record, point-lookup-dominated workloads; tiny collections |
| `MasterSlaveMM` | Latency-insensitive batch jobs where inline `free` is already cheap |
| `CTMM` | Uniformly hot working sets; non-compressible, non-regenerable data; strict p99 targets without a `floor_tier` |
| `JIT-GC Co-Design` | Short-lived processes (warmup dominates); hard real-time; environments without PMU/JIT |

---

## 5. What Would Falsify the Design

Pre-registered criteria. A technique is **refuted for a workload class** if, on the benchmark of §6 under the stated protocol, any of its criteria holds.

| Technique | Falsification criterion (examples) |
|-----------|-------------------------------------|
| `SegGenPool` | p99 allocation+free latency not better than the best of {mimalloc, jemalloc, glibc} on request-scoped workload at equal throughput and equal RSS |
| `LastUseLifetime` | Peak live bytes not lower than scope-end release on ≥ 3 workloads with long scopes; or compile-time overhead > agreed budget |
| `TileCompact` | RSS after soak (churn) not lower than non-compacting baseline by an agreed margin, or any step exceeds `B_us` at p99.9 |
| `ColSlab`/`LayoutMorph` | Narrow-scan throughput gain < expected from `U` by a wide margin (indicating the model is wrong), or phase-change workload shows no gain over the best static layout |
| `MasterSlaveMM` | Mutator p99 not better than inline free; or total CPU exceeds budget; or backlog bound violated |
| `CTMM` | Under a binding memory budget, hit-rate/latency not better than tuned LRU/TinyLFU on Zipf and shifting-Zipf workloads; or transition rate shows thrash |
| `JITDestruct` | No reduction in collector+barrier cost vs escape-analysis-only baseline; or handoff rate > threshold on representative code |
| `CacheSweep` | Mark throughput/LLC-miss counts no better than header-bit marking; prefetch ring gain absent |
| `StallHarvest` | p99 not lower than periodic-pause or fixed-background-thread baseline; or GC-induced mutator slowdown exceeds budget |
| **Whole package** | Any combination's measured benefit is smaller than the sum of subset benefits by more than interaction cost, i.e., the combination is net-negative vs the best proper subset |

---

## 6. Benchmark and Validation Methodology

### 6.1 Principles
1. **Separate design from evidence.** Nothing is "measured" until the protocol below produces a result recorded with its metadata.
2. **Ablate.** Each technique is on/off independently; test every step of the subset ladder (§2), not only the full stack.
3. **Equal-resource comparisons.** Compare at equal RSS budget, and separately at equal throughput; report both.
4. **Charge everything.** Include background-worker and GC thread CPU time and (where measurable) energy.
5. **Report tails, not means.** p50, p99, p99.9, max; with proper handling of coordinated omission.
6. **Report negative results.**

### 6.2 Hardware and environment reporting (mandatory)
CPU model, microarchitecture, core counts (P/E), SMT on/off, cache sizes, memory type/speed/channels, NUMA topology, kernel version, THP/huge-page settings, governor/frequency policy (pin frequency or report turbo behavior), isolation (`isolcpus`/cpuset), OS version, compiler/flags, PMU access mode, VM or bare metal.

### 6.3 Baselines

| Category | Baselines |
|----------|-----------|
| Manual allocators | glibc `malloc`, jemalloc, mimalloc, tcmalloc (tuned and default) |
| Tracing GC | A generational collector (e.g., G1-class), a low-pause concurrent collector (ZGC/Shenandoah-class), Go's collector, with heap-size sweeps |
| Ownership languages | Rust with default allocator, Rust with arena/index-based design |
| Tiering/caching | LRU, CLOCK, TinyLFU-based caches; OS swap/zswap for cold data; zstd/LZ4 standalone compression |
| Layout | Hand-written AoS and SoA (best static) |

### 6.4 Workloads

| ID | Workload | Targets | Key metrics |
|----|----------|---------|-------------|
| W1 | **Allocation churn microbench**: mixed sizes, controllable lifetime distributions, multi-threaded, remote frees | `SegGenPool`, `LastUseLifetime` | ns/op distribution, p99, RSS |
| W2 | **Pointer-chasing**: linked list / tree / graph traversal after long churn (heap aged) | `SegGenPool`, `TileCompact`, `CacheSweep` | Traversal time, LLC/dTLB misses |
| W3 | **Request/response simulator**: per-request temporaries + long-lived shared state; open-loop load generator | Whole stack; tail latency | p50/p99/p99.9, throughput, RSS |
| W4 | **Long-run fragmentation soak** (hours to days) with phase shifts | `TileCompact`, `SegGenPool` | RSS/live ratio `B` over time, step-latency histogram |
| W5 | **Entity/record scans**: narrow-field scans, whole-record access, mixed; phase-changing mix | `ColSlab`, `LayoutMorph` | Throughput, bytes-useful ratio `U`, cache-line counts |
| W6 | **Skewed cache**: Zipf and shifting-Zipf over a dataset larger than the memory budget, with regenerable and non-regenerable items | `CTMM`, `MasterSlaveMM` | Hit rate, latency incl. thaw, transition rate, bytes saved |
| W7 | **TTL-heavy store**: large expiries with bursts | `MasterSlaveMM` | Backlog, p99, worker CPU |
| W8 | **Managed-runtime allocation benchmark**: object-graph-heavy workloads with controllable escape fraction | `JITDestruct`, `CacheSweep`, `StallHarvest` | GC overhead, barrier cost, pause distribution, handoff rate |
| W9 | **Latency probe under GC load**: critical section with and without waits (I/O, locks) | `StallHarvest` | Mutator p99 with harvest on/off; harvested/total GC cycles |
| W10 | **Adversarial**: pinned-heavy, alias-heavy, scan-pollution, thaw storm, expiry storm, epoch stall | All | Failure mode behaviors vs §4.2 |

### 6.5 Metrics

| Category | Metrics |
|----------|---------|
| Latency | p50/p99/p99.9/max via HDR histogram; open-loop load to avoid coordinated omission |
| Throughput | ops/s at fixed RSS; at fixed latency target |
| Memory | RSS, PSS, live bytes, `B = RSS/live`, `F_ext`, side-table overhead, pinned residue, reclaim backlog bytes |
| Microarchitecture | L1/L2/LLC misses, dTLB misses, page faults (minor/major), cache-coherence events, backend-stall cycles, retired instructions, `PREFETCH` usefulness counters |
| CPU accounting | Mutator CPU, worker CPU, GC CPU, total CPU, (optionally) energy via RAPL or platform counters |
| Technique-internal | Handle `STALE` counts, tier transition rate, thaw count/latency, handoff rate, compaction step latency, epoch lag, harvest hit/miss rates |

### 6.6 Protocol
1. Fixed seeds and recorded configurations; configuration committed alongside results.
2. Warmup until steady state is established by a stated criterion (e.g., coefficient of variation of throughput < 2% over a window); JIT workloads warm to tier-up completion.
3. ≥ 10 independent runs for microbenchmarks (≥ 5 for soaks); report median and a 95% confidence interval (bootstrap); non-parametric tests for tail comparisons.
4. Pin threads; fix frequency or report variability; disable unrelated background load; for NUMA, test local and remote.
5. Heap-size sweeps for GC baselines (do not compare against a mistuned baseline).
6. **Ablation matrix**: each subset of §2's ladder.
7. **Fault injection / stress**: randomized delays at safepoints, forced moves, forced evictions, forced handoffs; differential testing against a slow reference implementation for correctness.
8. **Validation of the cost model**: for each hypothesis predicting a counter change (e.g., fewer cache lines), verify the counter moved, not just wall-clock time. A speedup without the predicted counter change is flagged as unexplained.

### 6.7 Threats to validity
- Synthetic workloads may not represent production allocation behavior.
- Microarchitecture-specific results (prefetch depth, PMU events, fill-buffer counts).
- Baselines may be better tuned than the prototype, or vice versa.
- Hidden costs in background threads.
- Implementation quality vs design merit are confounded; report implementation maturity.

### 6.8 Reporting template (per result)

```
Result ID:        R-<technique>-<n>
Hypothesis:       H<paper>.<n>
Hardware/OS:      <section 6.2>
Workload:         W<n>, params
Baselines:        <list, versions, tunings>
Configs:          <technique flags>
Metrics:          <median, 95% CI, p99, p99.9>
Counters:         <predicted vs observed>
Outcome:          supported | refuted | inconclusive
Notes/threats:    ...
```

Only entries in this format may promote a claim from **[H]** to **[M]** in the other documents.

---

## 7. Consolidated Hypothesis Register

| ID | Statement (short) | Workloads |
|----|-------------------|-----------|
| H1.1 | Bump + segment death lowers allocator p99 vs general-purpose `malloc` | W1, W3 |
| H1.2 | `LastUseLifetime` lowers peak live bytes vs scope-end release | W1, W3 |
| H1.3 | `TileCompact` limits RSS growth with bounded steps | W4 |
| H1.4 | `ColSlab` speeds narrow scans by ~`1/U` up to bandwidth limit | W5 |
| H1.5 | `LayoutMorph` beats best static layout on phase-changing mixes | W5 |
| H2.1 | `MasterSlaveMM` offload lowers mutator p99 vs inline free | W3, W7 |
| H2.2 | Per-thread touch rings reduce coherence misses | W6, W7 |
| H2.3 | Foreground-help bounds backlog | W7, W10 |
| H2.4 | Total CPU within budget | W3, W7 |
| H3.1 | `CTMM` beats flat LRU under binding budget on skewed access | W6 |
| H3.2 | Access rings reduce coherence traffic | W6 |
| H3.3 | Hysteresis cuts transition rate ≥ 10× at equal effectiveness | W6 |
| H3.4 | Plasma packing reduces dTLB/L2 misses | W6 |
| H3.5 | Break-even gating avoids net-negative demotions | W6 |
| H4.1 | `JITDestruct` + barrier elision reduces GC-related mutator cost | W8 |
| H4.2 | Handoff rate < 5–10% on typical code | W8 |
| H4.3 | Side-table marking beats header marking on misses/coherence | W2, W8 |
| H4.4 | Prefetch ring depth 8–16 improves mark throughput | W2, W8 |
| H4.5 | Bit-parallel sweep ≫ per-object sweep on sparse-dead heaps | W8 |
| H4.6 | `StallHarvest` lowers mutator p99 vs periodic pauses | W9 |
| H4.7 | PMU-informed prediction beats JIT-only prediction | W9 |
| H5.1 | The full stack is not net-negative vs the best proper subset on W3 | W3, W8 |

---

## 8. Target Use Cases (ordered by expected fit)

| Rank | Use case | Why | Minimal subset | Main risk |
|------|----------|-----|----------------|-----------|
| 1 | **Request/response servers** with per-request temporaries and shared long-lived state | Per-request bulk free; tail latency; cache TTLs | `SegGenPool` + `LastUseLifetime` + `MasterSlaveMM` | Escape-heavy frameworks reduce proof coverage |
| 2 | **In-memory caches / indexes / CDNs** | Skewed access, compressible/regenerable entries, TTL | `MasterSlaveMM` + `CTMM` | Thaw tail; contract correctness |
| 3 | **Entity/record-heavy compute** (games, simulation, analytics, columnar engines) | Narrow scans; stable handles; SIMD | `ColSlab` → `LayoutMorph` + `SegGenPool` | Whole-record access pattern mismatch |
| 4 | **Long-running daemons** (databases, brokers, agents) | Drift; RSS recovery | `TileCompact` + `SegGenPool` (+ `CTMM`) | Pinned residue; copy bandwidth |
| 5 | **Language runtimes / compilers** (this project's origin) | Compiler knowledge feeds the collector | `LastUseLifetime`, then `JIT-GC Co-Design` | Complexity; JIT/PMU dependence |
| 6 | **Latency-critical loops** (trading, telemetry, audio) | Avoid slow paths | `SegGenPool` + `LastUseLifetime`; `StallHarvest` only if validated | Misprediction on critical path; hard `no_gc` regions required |

**Anti-targets:** short-lived CLI tools; hard-real-time control loops; FFI-dominated code; platforms without huge pages/PMU/E-core affinity where the techniques that depend on them degrade.

---

## 9. Staged Adoption Roadmap

| Stage | Deliver | Exit criterion (hypotheses to pass) |
|-------|---------|--------------------------------------|
| 0 | Benchmark harness, baselines, reporting template (§6) | Reproducible baselines on W1–W4 |
| 1 | `SegGenPool` | H1.1 |
| 2 | `LastUseLifetime` (with `Scoped`/`Escaped` fallbacks) | H1.2 |
| 3 | `TileCompact` (handles-or-pin contract) | H1.3 |
| 4 | `ColSlab`, then `LayoutMorph` | H1.4, H1.5 |
| 5 | `MasterSlaveMM` | H2.1–H2.4 |
| 6 | `CTMM` | H3.1–H3.5 |
| 7 | `JITDestruct` → `CacheSweep` → `StallHarvest` | H4.1–H4.7 |
| 8 | Whole-stack evaluation, conflict rules C1–C10 verified under fault injection | H5.1 |

---

## 10. Honest Summary

- The designs are grounded in standard systems mechanics: cache-line granularity, dependent-load latency, TLB reach, coherence traffic, epoch reclamation, safepoints, deopt guards, and bounded incremental work. Those are **architectural** claims **[A]**.
- Every performance claim is currently a **hypothesis [H]**; nothing has been **measured [M]**.
- The largest risks are not in any single technique but in their **interactions** (C1–C10), the **hidden cost of background work**, and **speculation/feedback oscillation**.
- The right unit of adoption is the **subset ladder**, not the whole stack. If an individual rung fails its hypotheses, it is dropped without invalidating the others.

## 11. Open Questions (consolidated)

- `OPEN:` Unified descriptor for `Evicted` and Sublimation (C2).
- `OPEN:` Global idle-budget coordinator design and its fairness policy (C6).
- `OPEN:` Whether handle-based references can be made cheap enough that raw-pointer fast paths are rarely needed (C7).
- `OPEN:` Automatic tuning with safe bounds for `θ`, `τ`, `D`, `B_us`, dwell times.
- `OPEN:` Verification strategy for the move protocol (model checking of the `moving`-bit/EBR interaction).
- `OPEN:` Which parts of the stack Kaiba's compiler and runtime can host first (a scoping decision for the project owner).
