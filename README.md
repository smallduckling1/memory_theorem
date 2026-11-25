# Unified Memory Architecture — Documentation & Research Package

> Working title. A modular set of memory-management, layout, tiering, and JIT–GC co-design techniques, each named, specified, and explained independently before being composed.

## Origin

This was seen by me because of my project, the **Kaiba compiler** ([github.com/smallduckling1/kaiba](https://github.com/smallduckling1/kaiba)). Building a compiler made the cost of the usual memory models (manual `malloc`/`free`, tracing GC, static ownership, and a flat view of memory) concrete, and these techniques grew out of trying to remove those costs without paying for them elsewhere.

Nothing in this package should be read as a description of what Kaiba currently implements unless a section says so explicitly. It is a design and research package.

---

## 1. Claim-Status Legend (read this first)

Every non-trivial claim in these documents carries one of three tags so that **architecture** is never confused with **evidence**:

| Tag | Meaning |
|-----|---------|
| **[A]** | *Architectural.* Follows from the design and from hardware/OS mechanics (cache lines, TLB, page faults, memory ordering, safepoints). Can be argued without running code. |
| **[H]** | *Hypothesis.* A predicted performance or cost outcome that **must be validated by benchmark**. No numbers are asserted. |
| **[M]** | *Measured.* Backed by an actual experiment. **No [M] claims exist yet.** Any figure added later must name hardware, workload, and method (see `paper/05`). |

Latency figures quoted for hardware (e.g. "DRAM is roughly 50–100 ns") are typical orders of magnitude, not measurements of this system.

---

## 2. File Map

```
.
├── README.md                              ← you are here
├── THEORY.md                              ← main readable document (start here)
└── paper/
    ├── 00_abstract_and_problem.md         ← problem, cost model, full taxonomy
    ├── 01_structural_and_lifetime_models.md   ← SegGenPool, LastUseLifetime, TileCompact, LayoutMorph/ColSlab
    ├── 02_master_slave_model.md           ← MasterSlaveMM            (pending)
    ├── 03_ctmm_thermodynamic_memory.md    ← CTMM                     (pending)
    ├── 04_jit_gc_codesign.md              ← JITDestruct, CacheSweep, StallHarvest (pending)
    └── 05_synthesis_benchmarks_and_limits.md  ← compatibility, failure modes, benchmarks (pending)
```

## 3. Quick Navigation

| I want to… | Go to |
|------------|-------|
| Understand why existing models hurt, in plain engineering terms | `THEORY.md` §1 |
| See every technique in one place, one card each | `THEORY.md` §2 |
| See how techniques combine and where they apply | `THEORY.md` §3 |
| Read the formal problem statement and cost model | `paper/00` |
| Implement the allocator / lifetime / compaction / layout layer | `paper/01` |
| Implement policy-driven background reclamation | `paper/02` |
| Implement temperature-based tiering | `paper/03` |
| Implement the JIT–GC side | `paper/04` |
| Judge what can go wrong and how to test it | `paper/05` |

**Suggested reading orders**

- *Everyone:* `README` → `THEORY` §1 → §2 → §3.
- *Implementer:* `THEORY` §2 → `paper/00` taxonomy → the paper for your layer.
- *Reviewer / skeptic:* `THEORY` §1 → `paper/05` limits → `paper/00` cost model.

---

## 4. Naming Glossary of Every Technique

Every technique has exactly one canonical name. Use these names verbatim in code, issues, and discussion.

| # | Canonical name | Code name | One-line description | Layer | Paper |
|---|----------------|-----------|----------------------|-------|-------|
| 1 | Segmented Bump + Generational Dense Pool | `SegGenPool` | Segment-based bump allocation plus dense, generation-tagged slot pools with stable handles. | Allocation / structure | 01 |
| 2 | Last-Use Causal Lifetime | `LastUseLifetime` | Compiler-derived release at the *last causal use* of a value rather than at scope end. | Lifetime (static) | 01 |
| 3 | Incremental Tile Compaction | `TileCompact` | Budgeted, resumable evacuation of sparse fixed-size tiles to defragment without long pauses. | Relocation | 01 |
| 4a | Adaptive Layout Morphing | `LayoutMorph` | Profile-driven switching among AoS, SoA, AoSoA, and hot/cold-split layouts. | Layout | 01 |
| 4b | Columnar Slabs | `ColSlab` | Slab container storing records field-by-field with stable handles. | Layout | 01 |
| 5 | Master / Slave Memory Management | `MasterSlaveMM` | Policy-bearing Masters own scoped Slaves; background workers reclaim; explicit `Alive`/`Deleted`/`Evicted`/`Expired` statuses. | Lifetime (policy) | 02 |
| 6 | Cognitive-Thermodynamic Memory Model | `CTMM` | Temperature- and weight-driven tiering through **Plasma**, **Liquid**, **Crystalline**, **Sublimation** with hysteresis. | Tiering | 03 |
| 7 | JIT–GC Co-Design | `JIT-GC Co-Design` | Umbrella for the three runtime/compiler collection techniques below. | Collection | 04 |
| 7a | JIT-Synthesized Destructors | `JITDestruct` | JIT emits type-specific reclaim code plus speculative escape guards and handoffs. | Collection | 04 |
| 7b | Speculative Cache-Line Sweeping | `CacheSweep` | 64B-aligned side-table bitmaps, unrolled JIT-generated tracers, and `PREFETCH`-driven sweeping. | Collection | 04 |
| 7c | Stall-Cycle Harvesting | `StallHarvest` | PMU telemetry + JIT control-flow prediction schedule bounded `gc_micro_step()` into stall windows. | Collection | 04 |

**Supporting terms**

| Term | Meaning |
|------|---------|
| Handle | Stable, validated reference (not a raw address). Variants: `(seg, slot, gen)` in `SegGenPool`; `(slot_index, epoch_gen)` in `CTMM`. |
| Generation / epoch | Counter bumped when a slot is reused or a tier changes, so stale handles are detected rather than dereferenced. |
| Tile | Fixed-size, aligned region (default 64–256 KiB) that is the unit of compaction. |
| Segment | Larger aligned region (default 2 MiB) that is the unit of bump allocation and bulk reclaim. |
| Pin scope | A bounded region of code during which a resolved raw pointer is guaranteed not to be invalidated. |
| EBR | Epoch-based reclamation; grace-period scheme that delays freeing until no thread can hold an old reference. |

---

## 5. How to Read and Iterate on These Documents

1. **One technique, one card.** Each technique is specified with the same fields: *Purpose → Inputs → State/metadata → Read path → Write path → Reclaim/evict path → Failure and edge cases → Trade-offs*. When you add or change a technique, update its card in `THEORY.md` §2 **and** its paper.
2. **Names are stable.** Renaming a technique requires updating the glossary above first.
3. **Tag every claim** with **[A]**, **[H]**, or **[M]**. Promote **[H]** → **[M]** only by adding the method and raw result to `paper/05`.
4. **Conflicts go in one place.** Cross-technique interactions and conflicts belong in `paper/05` and `THEORY.md` §3, not scattered in individual papers (individual papers may link to them).
5. **Open questions** are marked `OPEN:` inline so they can be grepped: `grep -rn "OPEN:" .`
6. **Iteration loop:** change a card → check the glossary → check `paper/05` compatibility matrix → add or update a benchmark hypothesis.
7. **Status:** `paper/00` and `paper/01` are complete drafts. `paper/02`–`paper/05` are pending.

## 6. Scope and Non-Goals

- Not a drop-in replacement for any existing runtime.
- Not a claim that every technique should be used together; `paper/05` documents which combinations conflict.
- Not a set of benchmark results. Performance statements are **[A]** or **[H]** until measured.
