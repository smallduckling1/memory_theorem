# Paper 03 — Cognitive-Thermodynamic Memory Model (`CTMM`)

Claim tags: **[A]** architectural · **[H]** hypothesis · **[M]** measured (none).
The thermodynamic vocabulary is a *metaphor for a policy*, not a physical claim: "temperature" is a decayed access rate, "semantic weight" is a retention value.

---

## 1. Purpose

Give memory a notion of **residency class** driven by how hot an object is and how costly it is to lose. Hot objects get packed and fast; cold objects get compressed; regenerable objects can be dropped entirely, leaving only a recipe. Access to a non-resident object transparently **thaws** it.

## 2. Inputs

| Input | Source |
|-------|--------|
| Access events (object touched) | Mutator, via per-thread access log (§5.2) |
| Semantic weight hints: `priority`, `regenerable`, `regen_descriptor`, `floor_tier` | Application / type declaration / `MasterSlaveMM` policy |
| Measured regeneration or thaw cost | Runtime timing of past thaws |
| Size, compressibility estimates | Object metadata; sampled compression ratio |
| Pressure signals (resident bytes vs budget, Plasma budget) | `SegGenPool` / OS |
| Parameters: `τ` (decay time constant), thresholds, dwell times, tier budgets | Configuration |

## 3. The Four States

| State | Role | Representation | Access cost | Mutable in place? |
|-------|------|----------------|-------------|-------------------|
| **Plasma** | Hottest. Highest access rate. | Uncompressed, packed into a dense hot region (e.g., `ColSlab`/hot arena, huge-page backed), co-located with other Plasma objects for cache and TLB locality | Handle resolve + pointer; no thaw | Yes |
| **Liquid** | Default working set. | Ordinary heap layout (`SegGenPool`) | Handle resolve + pointer | Yes |
| **Crystalline** | Cold, retained. | Compressed (e.g., LZ4/zstd-class, optional dictionary) or packed/quantized; read-mostly; may live in a slower memory tier (CXL-attached, NVMe-backed mapping) | Thaw (decompress) on access | No; thaw first |
| **Sublimation** | Cold *and regenerable*; payload released. | **Descriptor only**: a recipe (function id + arguments), a content hash, or a backing-store key. No payload memory | Thaw = regenerate or refetch | No; regenerate first |

"Sublimation" here means a direct solid-to-gas jump in the metaphor: the payload skips any further compressed form and disappears, leaving only enough information to reconstruct it. It is permitted **only** when the object is declared `regenerable` and its `regen_descriptor` is valid. **[A]**

State graph:

```
         promote (T↑)              demote (T↓)               demote + regenerable
 Liquid ──────────────► Plasma     Liquid ───────► Crystalline ─────────────────► Sublimation
   ▲ ◄─────────────────── │           ▲ ◄───────── thaw ─────── │                      │
   │      demote          │           │                          │                      │
   └──────────── thaw (hydrate) ◄──────┴──────────────────────────┴──────────────────────┘
        Tiny/incompressible regenerable objects may go Liquid → Sublimation directly.
```

Default thaw target is **Liquid**, not Plasma: a single access must not catapult an object into the hottest tier (§6, hysteresis). Promotion to Plasma follows only from sustained temperature, or from an explicit prefetch/promote hint. **[A]**

## 4. Equations

### 4.1 Temperature (decayed access rate)

Continuous form with half-life `τ`:

```
T_i(t) = Σ_k  a_k · 2^{ −(t − t_k) / τ }
```

where `t_k` are access times and `a_k` is the access weight (1 for a normal access; less for scan-like access, see §8). **Lazy update**: store `(T, t_last)` and update only when touched or examined:

```
T_i ← T_i · 2^{ −(t_now − t_last)/τ } + a
t_last ← t_now
```

No periodic whole-heap decay scan is needed. **[A]**

### 4.2 Semantic weight

```
W_i = priority_i · R_i
R_i = estimated cost to lose the object  (regeneration time, refetch latency, or recompute cost;
                                          measured via EWMA of past thaw/regen times; ∞ if not regenerable)
```

A non-regenerable object has `R = ∞` and is never sublimated. `W` is a *retention value per access*; `priority` is a declared class multiplier.

### 4.3 Retention score and density

```
S_i = T_i · W_i                    (expected loss rate if the object is not resident)
D_i = S_i / size_i                 (loss rate per byte: ranks objects when memory is scarce)
```

Under pressure, objects with the **lowest `D`** are demoted first. **[A]**

### 4.4 Break-even gate for demotion

Demote an object only if the expected benefit outweighs the expected thaw cost over a planning horizon `H`:

```
bytes_saved_i · ρ  >  P_reaccess_i(H) · C_thaw_i

P_reaccess_i(H) ≈ 1 − exp( − λ_i · H ),   λ_i ≈ T_i · ln2 / τ
ρ  = current memory pressure price (value of a freed byte, set by the pressure controller)
C_thaw_i = decompression time (Crystalline) or regeneration time (Sublimation), measured
```

Under low pressure `ρ → 0` and nothing is demoted even if it is cold. Under high pressure `ρ` rises and colder objects tip over first. **[A]**

### 4.5 Promotion to Plasma gate

```
promote if  T_i ≥ θ_P↑  AND  dwell_in_Liquid ≥ d_up  AND  plasma_bytes + size ≤ plasma_budget
            OR   (explicit promote hint)
```

If the budget is full, the candidate replaces the Plasma resident with the lowest `D` only if its `D` exceeds that resident's by a margin `δ`. This prevents swaps at near-equal scores.

## 5. Handles, Metadata, and Concurrency

### 5.1 Two-tier handle

```
 63                    32 31                      0
 [ slot_index : 32     ] [ epoch_gen : 32          ]
```

- **`slot_index`** is stable for the life of the object. It indexes the **slot table**; it never changes on tier transitions.
- **`epoch_gen`** is the slot's reuse generation: incremented when the object is freed and the slot reused. A handle whose `epoch_gen` mismatches is `STALE`.

A separate **per-slot `tier_epoch`** (in the slot entry, not in the handle) is incremented on every tier transition. It exists so *cached resolved pointers* and inline caches can detect that a payload has moved or changed form, without invalidating handles that applications hold. **[A]**

### 5.2 Slot table entry (16 B; four per cache line)

```c
struct CTMMEntry {
    u64 payload;      // [ptr or offset:48 | tier:2 | flags:6 | fmt:8]  atomically updated
    u32 epoch_gen;    // identity generation
    u16 tier_epoch;   // bumped on tier transitions (wraps; compared for equality only)
    u16 temp_q;       // quantized T (log-scale 16-bit) — updated by workers, not mutators
    // coarse last-update timestamp lives in a parallel side array (workers only)
};
```

Cold metadata (descriptor for Sublimation, compression dictionary id, size) lives in a separate side structure indexed by `slot_index`, so the hot entry stays at 16 B. **[A]**

### 5.3 Resolve (read) path

```
resolve(h):
    e = &table[h.slot_index]
    p = load_acquire(e.payload); g = e.epoch_gen
    if g != h.epoch_gen: return STALE
    switch tier(p):
        Plasma | Liquid: log_access(h.slot_index); return ptr(p)
        Crystalline | Sublimated: return thaw(h)          // §7
```

**The mutator does not write the shared entry on access.** `log_access` appends the slot index to a **per-thread SPSC access ring** (8 B per entry, batch-flushed or polled by a worker). Workers fold ring contents into `T` using the lazy-decay formula. This prevents a coherence write on every read and keeps `CTMMEntry` lines in Shared state across cores. **[A]**

Sampling option: log only 1 in `N` accesses (weighted `a = N`) to bound ring traffic on very hot objects.

### 5.4 Lock-free concurrency model

| Element | Technique |
|---------|-----------|
| Tier transitions | CAS on `payload` word (pointer + tier + flags together) |
| Old payload reclamation | EBR: retired to epoch `e`, freed after all threads pass `e+2` |
| Access logging | Per-thread SPSC rings; no shared writes on the read path |
| Worker coordination | Workers own disjoint slot-table ranges (sharded by `slot_index` range); no locks |
| Thaw dedup | `THAWING` flag claimed by CAS in `payload.flags` |
| Concurrent mutation during demotion | Per-object version (in the side array); writers bump it via `resolve_mut`; demoter re-checks before publishing (seqlock pattern) |

### 5.5 E-core / background execution

A pool of workers, pinned to efficiency cores or a dedicated cpuset, runs a **clock-hand scan** over the slot table in fixed-size chunks (e.g., 4096 entries per pass, rate-limited):

1. Drain access rings; update `T` for touched slots.
2. For each examined slot: compute decayed `T`, `D`; evaluate gates §4.4/§4.5.
3. Execute transitions (§6) within a per-pass byte and time budget.

Caveats stated plainly: E-cores are slower per core, share the LLC and memory controller with P-cores, and heavy compression traffic can pollute shared cache; workers therefore use non-temporal loads for cold payload reads where available and a bandwidth cap. On homogeneous CPUs the same design runs on a reserved core set. **[A]**

## 6. Transitions and Hysteresis

### 6.1 Thresholds with a dead band

| Transition | Condition | Dwell requirement |
|------------|-----------|-------------------|
| Liquid → Plasma | `T ≥ θ_P↑` | in Liquid ≥ `d_up` |
| Plasma → Liquid | `T ≤ θ_P↓` where `θ_P↓ < θ_P↑` | in Plasma ≥ `d_P_min` |
| Liquid → Crystalline | break-even §4.4 holds **and** `T ≤ θ_C↓` | in Liquid ≥ `d_down` |
| Crystalline → Liquid | thaw by access; stays Liquid for ≥ `d_post_thaw` before re-demotion | — |
| Crystalline → Sublimation | `regenerable` **and** `T ≤ θ_S↓` **and** break-even §4.4 holds with `C_thaw = regen cost` | in Crystalline ≥ `d_S` |
| Sublimation → Liquid | thaw by access (regenerate) | — |

Design rules:
1. **Dead band**: up-threshold strictly above down-threshold (e.g., ratio ≥ 2×) so oscillating `T` near a boundary causes no transitions.
2. **Dwell times** are asymmetric: cooling is slow (long `d_down`), thaw is immediate (access is fact), and post-thaw residency is guaranteed.
3. **Transition budget**: at most `K` transitions per object per window; a repeatedly transitioning object is marked `STICKY` in its current tier for a period.
4. **Cost gate** (§4.4) ensures a transition is worth its price. **[A]**

### 6.2 Transition mechanics

**Liquid → Crystalline (demotion):**
```
v0 = version(slot)
buf = compress(payload)                     // worker, scratch memory
if size(buf) ≥ size(payload) · κ: mark INCOMPRESSIBLE; stay Liquid
alloc crystalline storage; copy buf
if version(slot) != v0: discard; abort       // write raced: retry later
CAS payload: (Liquid, p_old) → (Crystalline, p_new); tier_epoch++
retire(p_old)                                // EBR
```

**Plasma packing (promotion):** copy payload into the Plasma region (bump/dense in a hot segment), CAS pointer and tier, retire the old location. Because the object moves, promotion obeys the `moving` protocol of `TileCompact` (Paper 01 §4.6): pinned objects are skipped. **[A]**

**Crystalline → Sublimation:** write descriptor (recipe/key) into cold metadata, CAS to `Sublimated`, retire crystalline storage. Requires that the descriptor was validated at declaration time.

## 7. Transparent Thawing / Hydration

```
thaw(h):
    e = entry(h); p = load(e.payload)
    if CAS(e.payload, p, p | THAWING):          // claim
        switch tier(p):
            Crystalline: buf = alloc_liquid(size); decompress(storage(p) → buf)
            Sublimated:  buf = regenerate(descriptor(h.slot_index))     // bounded time; idempotent
        publish: store_release(e.payload, make(buf, Liquid)); e.tier_epoch++
        retire(storage(p))
        log_access(...)                          // counts as an access (weight a)
        return buf
    else:
        spin-yield briefly, or return PENDING (async API), or wait on a per-slot parker
```

API variants:
- `get(h)`: blocking; thaw on the caller's thread (simple, adds the thaw latency to the caller's tail).
- `get_async(h)`: returns `PENDING` and schedules thaw on a worker; caller polls or awaits.
- `prefetch(h)`: hint that a thaw will be needed soon (from `JIT-GC` control-flow prediction or application knowledge).

Thaw is **a new tail-latency source**. It is bounded (decompression ≈ size / throughput; regeneration bounded by a per-class timeout) and can be avoided for latency-critical classes with a `floor_tier` (e.g., "never below Liquid"). **[A]**

## 8. Failure and Edge Cases

| Case | Behavior |
|------|----------|
| **Thrash at a boundary** | Dead band + dwell + transition budget + `STICKY` |
| **Scan pollution** (one pass over cold data heats everything) | Scan-resistant weighting: accesses tagged `SCAN` use `a ≈ 0` (or admit-on-second-touch); optional doorkeeper filter |
| **Thaw storm** (sudden shift to cold data) | Rate-limit async thaws; per-class concurrency cap; temporarily raise `θ` for demotion; pressure controller lowers `ρ` |
| **Incompressible data** | `INCOMPRESSIBLE` flag; skip Crystalline; go Sublimation only if regenerable |
| **Wrong `regenerable` declaration / non-idempotent regen** | Correctness bug by contract violation; debug mode verifies by hashing regenerated payload against a stored content hash (`OPEN:` hashing cost) |
| **Regeneration failure or timeout** | `get` returns error/`MISS`; slot stays Sublimated; optional negative cache |
| **Memory pressure while thawing** | Thaw needs allocation; maintain a thaw reserve; else fail soft |
| **Decay parameter mis-tuned** (`τ` too short/long) | `τ` chosen per class; adaptive `τ` from observed reuse distances `OPEN:` |
| **Temperature quantization error** | Log-scale 16-bit; relative error bounded; documented |
| **Stale handles** | `epoch_gen` mismatch → `STALE` |
| **Slot-table growth** | Chunked table; indices stable; no relocation of entries (so mutators can hold `&entry` within an epoch guard) |
| **Worker lag** | Temperatures stale; demotion delayed; no correctness impact |
| **PMU/hardware heterogeneity** | CTMM needs no PMU; only core affinity (optional) |

## 9. Trade-offs

| Benefit | Cost |
|---------|------|
| Larger effective capacity: cold data compressed or dropped **[A]** | CPU for compress/decompress/regenerate; background cores |
| Hot data packed → fewer cache lines/TLB entries **[A]** | Promotion copies; budget management |
| Pressure-adaptive: nothing demoted when memory is plentiful (§4.4) **[A]** | Needs a pressure controller (`ρ`) |
| No write on read path (access rings) **[A]** | Temperature is approximate and delayed |
| Graceful degradation under memory pressure instead of OOM **[A]** | Thaw adds a tail-latency source; correct regen descriptors are programmer-supplied |
| Two-tier handles keep application references stable across all transitions **[A]** | One extra load per access (slot table) |

## 10. Relationship to Other Techniques

| Technique | Relationship |
|-----------|--------------|
| `SegGenPool` | Provides Liquid storage and Plasma hot segments; CTMM's slot table can *be* the handle table; Plasma uses dense segments |
| `TileCompact` | Plasma/Liquid moves obey the same `moving` protocol; compaction treats tier as placement hint (cold tiles evacuated into cold regions) |
| `LayoutMorph`/`ColSlab` | `HotColdSplit` layout aligns with tiers: hot columns Plasma/Liquid, cold columns Crystalline; Plasma region can be a `ColSlab` |
| `MasterSlaveMM` | Master policy supplies `priority`, `floor_tier`, TTL; Master may force `Evicted`; overlap with Sublimation (Paper 05) |
| `LastUseLifetime` | Dead objects bypass tiers entirely; `Escaped` long-lived objects are CTMM candidates |
| `StallHarvest` | Thaw prefetch may use predicted control flow; workers share idle-time budget |

## 11. Hypotheses

| ID | Hypothesis | Falsified if |
|----|------------|--------------|
| H3.1 | Under a memory budget smaller than the working set, CTMM keeps higher hit-rate / lower latency than a flat LRU cache of the same budget on skewed (Zipf-like) access | No improvement over tuned LRU/TinyLFU |
| H3.2 | Access rings reduce coherence traffic relative to per-access timestamp writes | No difference in LLC/coherence counters or throughput |
| H3.3 | Hysteresis (dead band + dwell) cuts transition rate by an order of magnitude at equal effectiveness | Transition rate not reduced or effectiveness drops |
| H3.4 | Plasma packing reduces dTLB/L2 misses for the hot set vs unpacked Liquid | No counter improvement |
| H3.5 | Break-even gating avoids net-negative demotions under low pressure | Demotions occur with negative net benefit |

## 12. Open Questions

- `OPEN:` Per-class adaptive `τ` from observed reuse distances.
- `OPEN:` Dictionary management for compression across many small objects.
- `OPEN:` Verification cost of `regenerable` contracts in production.
- `OPEN:` Whether Plasma should use hardware-assisted cache partitioning (e.g., cache allocation technology) where available.
