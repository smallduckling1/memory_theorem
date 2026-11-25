# Paper 02 — Master / Slave Memory Management (`MasterSlaveMM`)

Claim tags: **[A]** architectural · **[H]** hypothesis · **[M]** measured (none).
Pseudocode is illustrative. Shared conventions (handles, EBR, side tables, pin scopes) are defined in Paper 01 §1.

---

## 1. Purpose

Separate **policy** (how long an object lives, how much of a collection may be resident, who is evicted first) from **mechanism** (freeing, evicting, rehydrating), and move the mechanism **off the latency-critical path** onto background workers.

- **Master:** owns policy for a group of objects.
- **Slave:** an object whose lifetime is *scoped by* its Master.
- **Workers:** background threads that perform physical reclamation, eviction, and rehydration.

The hot path only reads or flips a status word. **[A]**

## 2. Inputs

| Input | Source |
|-------|--------|
| Master policy: `capacity_bytes`, `max_count`, `ttl`, `idle_ttl`, `eviction_order` (LRU / CLOCK / weighted / FIFO), `priority`, optional `loader` (rehydration function) | Application or compiler-emitted defaults |
| Slave insertion: payload, size, optional per-object TTL override, weight | Application |
| Release requests of kind `Defer` | `LastUseLifetime` (Paper 01 §3.4) |
| Memory-pressure signals (RSS vs. budget, free-segment watermark) | `SegGenPool` / OS |
| Weight and temperature hints | `CTMM` (optional, Paper 03) |

## 3. State and Metadata

### 3.1 Slave status word (side table, 64 bits)

```
 63        61 60      56 55            32 31                    0
 [ state:3 ] [ flags:5 ] [ aux:24       ] [ gen:32             ]
```

| Field | Meaning |
|-------|---------|
| `state` | `Alive`, `Deleted`, `Evicted`, `Expired` (below) |
| `flags` | `HYDRATING`, `PINNED`, `RETIRED_QUEUED`, `RENEWABLE`, `DIRTY` |
| `aux` | Coarse timestamp bucket (expiry / last-touch), or loader token |
| `gen` | Slot generation; incremented on physical free. Same role as `gen` in the `SegGenPool` handle |

All transitions are performed by CAS on this one word, so a status change and a generation check are a single atomic operation. **[A]**

### 3.2 The four explicit statuses

| Status | Meaning | Payload resident? | Handle valid? |
|--------|---------|-------------------|---------------|
| **`Alive`** | Normal. Readable and writable. | Yes | Yes |
| **`Deleted`** | Logically deleted by its owner; physical reclaim pending. | Yes until reclaimed (never read) | No (reads return `STALE`) |
| **`Evicted`** | Payload dropped under capacity policy; metadata and any loader token remain. | No | Yes (rehydratable) |
| **`Expired`** | TTL elapsed. Subject to policy: reap, or lazily renew on access. | Yes until reaped | Yes until reaped; reads follow `on_expired` policy |

### 3.3 State machine

```
                 insert
   (free slot) ────────────► Alive
                              │  ▲
              owner delete    │  │ hydrate / renew
        ┌─────────────────────┤  │
        ▼                     │  │
     Deleted ◄────── reap ── Expired ◄── ttl elapsed ── Alive
        │                     ▲                          │
        │ worker reclaim      │ ttl elapsed              │ capacity pressure
        ▼                     │                          ▼
   (slot free, gen++)      Evicted ◄────────────────── Alive
                              │ ▲
                              │ └─ hydrate (loader)  → Alive
                              └─── owner delete → Deleted
```

Allowed transitions:

| From → To | Trigger | Actor |
|-----------|---------|-------|
| (free) → `Alive` | `insert` | Mutator |
| `Alive` → `Deleted` | `delete` | Mutator (owner) |
| `Alive` → `Expired` | TTL elapsed | TTL worker |
| `Alive` → `Evicted` | Capacity/pressure policy | Eviction worker |
| `Expired` → `Alive` | `renew` or access under lazy-renew policy | Mutator or worker |
| `Expired` → `Deleted` | Reap | TTL worker |
| `Evicted` → `Alive` | Hydrate via `loader` | Mutator (sync) or worker (async) |
| `Evicted` → `Expired` | TTL elapsed while evicted | TTL worker |
| `Evicted` → `Deleted` | `delete` or reap | Mutator or worker |
| `Deleted` → (free) | Physical reclaim after grace | Reclaim worker |

Any other transition is rejected (CAS fails or invariant check). `Deleted` is absorbing until the slot is freed. **[A]**

### 3.4 Master record

```c
struct Master {
    Policy   policy;          // capacity, ttl, order, priority, on_expired, loader
    atomic<u64> resident_bytes;   // sharded per core, summed lazily (see §7.3)
    atomic<u32> resident_count;
    EvictionIndex idx;        // CLOCK ring / weighted heap / LRU approximation
    TimerWheel    ttl;        // hashed hierarchical timer wheel keyed by aux time bucket
    Region*       region;     // bump region (SegGenPool) owning Slaves allocated "in" this Master
    u32           master_gen; // invalidated when Master is destroyed
};
```

### 3.5 Slave scope rule
A Slave's lifetime is a subset of its Master's:
- Master destroyed → every Slave transitions to `Deleted` in bulk. If Slaves were bump-allocated in the Master's `region`, the region is dropped as a unit (`SegGenPool` segment death), with no per-object work for POD payloads. **[A]**
- A Slave handle used after its Master's destruction resolves to `STALE` through `master_gen` / slot `gen` mismatch.

## 4. Read, Write, Reclaim, Evict Paths

### 4.1 Read path (mutator)

```
get(h):
    guard = epoch_enter()                     // cheap: thread-local store
    w = load_acquire(status[h.slot])
    if gen(w) != h.gen: return STALE
    switch state(w):
        Alive:    touch_hint(h); return ptr(h)       // see §7.2: touch is batched
        Expired:  switch master.on_expired:
                      ReturnStale: return ptr(h) flagged STALE_OK
                      Miss:        return MISS
                      Renew:       if CAS(Expired→Alive, aux=now+ttl): return ptr(h)
        Evicted:  return hydrate(h)   // sync: loader() under HYDRATING flag; or MISS/async
        Deleted:  return STALE
    // guard dropped by caller when done with ptr
```

The epoch guard ensures that even if a worker transitions the slot to `Evicted` or `Deleted` right after the status load, the payload is not freed while the reader uses it. **[A]**

### 4.2 Write / insert path

```
insert(master, payload, size, opts):
    if master.resident_bytes + size > capacity:  admission_control()   // may trigger sync eviction help
    slot = alloc_slot()                          // SegGenPool dense pool or region bump
    write payload
    store_release(status[slot], make(Alive, gen, aux=now+ttl))
    master.idx.insert(slot); master.ttl.schedule(slot, aux)
    return handle
```

Mutation of an existing Slave goes through `get_mut(h)`, which additionally sets `DIRTY` (if the Master uses write-back) and bumps a per-slot version used by workers that snapshot (§5.3).

### 4.3 Delete path (the hot-path cost is one CAS + one enqueue)

```
delete(h):
    w = load(status[h.slot])
    if gen(w) != h.gen or state(w) == Deleted: return NOOP
    if CAS(status, w, with(Deleted, RETIRED_QUEUED)):
        reclaim_q[this_core].push({slot, gen})        // SPSC ring, batch-flushed
        master.resident_bytes -= size  (sharded)
```

The mutator never touches the payload or the allocator's free lists. **[A]**

### 4.4 Reclaim path (worker)

```
reclaim_batch(entries):
    for {slot, gen} in entries:
        w = load(status[slot])
        if gen(w) != gen or state(w) != Deleted: continue        // stale entry: reused or raced
        retire_to_epoch(payload(slot), current_epoch)             // EBR: freed after grace
        later (after grace): run destructor if any; status = make(Free, gen+1)
                             free_slot(slot)                      // push to freelist
```

### 4.5 Evict path (worker)

```
evict_batch(master, need_bytes):
    victims = master.idx.pick(need_bytes)       // CLOCK/weighted; skips PINNED, HYDRATING
    for slot in victims:
        w = load(status[slot])
        if state(w) != Alive or flags(w) & (PINNED|HYDRATING): continue
        if master.policy.write_back and (flags(w) & DIRTY): spill(slot) // write to backing store first
        if CAS(status, w, with(Evicted)):
            retire_to_epoch(payload(slot))                    // payload freed after grace
```

### 4.6 Hydrate path

```
hydrate(h):
    w = load(status[h.slot])
    if CAS(status, w, w | HYDRATING):              // claim: exactly one hydrator
        p = master.loader(token(w))                // may block: I/O, recompute
        install p; store_release(status, make(Alive, gen, aux=now+ttl))
        return ptr
    else:
        wait / yield / return PENDING               // another thread is hydrating
```

### 4.7 TTL path
A hashed hierarchical timer wheel (granularity ≈ policy resolution, e.g., 1 ms–1 s) holds `{slot, gen, deadline}`. A TTL worker advances the wheel and CAS-transitions due slots `Alive → Expired` (and then reaps or leaves them per policy). Deadlines are jittered by ±δ to avoid synchronized expiry storms (§8). **[A]**

## 5. Background Worker Offloading

### 5.1 Worker roles
| Role | Work | Priority |
|------|------|----------|
| Reclaim worker | Drain delete queues, retire payloads, free slots | Medium |
| Eviction worker | Keep `resident_bytes ≤ capacity` with headroom (`low/high watermark`) | High under pressure |
| TTL worker | Advance wheel, expire, reap | Low |
| Hydration worker | Async prefetch/hydration for predicted accesses | Low–medium |

Workers are pinned preferably to efficiency cores (E-cores) or a dedicated cpuset, so that their cache and bandwidth footprint stays away from latency-critical cores; see §6 for what this does and doesn't buy. **[A]**

### 5.2 Queues
- **Per-producer SPSC rings** from each mutator core to each worker shard, so no producer ever contends for a queue tail. Workers poll rings round-robin.
- Mutators push in **batches** (e.g., 16–64 entries) to amortize cache-line transfers.
- Backpressure: if a ring exceeds a high-water mark, the mutator performs **foreground help** (processes a small batch itself) so memory held by deferred work stays bounded. **[A]**

### 5.3 Snapshot-based spill
For write-back Masters, a worker snapshots a payload while mutators may be writing. It uses a per-slot version: read `v0`, serialize, re-read; if changed, retry or skip. The same seqlock pattern as in `TileCompact` (Paper 01 §4.6).

## 6. Cache and Cross-Core Costs (explicit accounting)

Offloading moves work, it does not make it free. Costs, with where they appear:

| Cost | Mechanism | Magnitude (order) | Mitigation |
|------|-----------|-------------------|------------|
| **Status-word line bounce** | `delete` CAS makes the line Modified on the mutator core; the worker's read pulls it across cores. Adjacent slots share a 64 B line (8 words), so unrelated slots cause false sharing. | One cross-core transfer ≈ tens of ns to ~100 ns, plus invalidation of other sharers **[A]** | Group status words per owner thread/segment; keep read-mostly state separate from frequently CASed state; batch work so one transfer serves many entries |
| **Queue transfer** | Producer writes ring entries; consumer reads them. | One line per 8 entries (8 B entries) | Batch pushes; SPSC; consumer prefetches next ring line |
| **Payload touch by worker** | Destructors / spill read payload lines on the worker's core, evicting its cache and possibly pulling lines away from the producer. | Per-line miss ≈ DRAM or cross-core latency | For POD payloads the worker touches **metadata only**; payload memory is released at segment/tile granularity |
| **Memory held until worker runs** | Deferred reclamation delays reuse. | Bytes × deferral time | Watermarks; foreground help |
| **Shared LLC / memory-bandwidth contention** | E-cores share LLC and memory controllers with P-cores. | Workload-dependent | Non-temporal loads for spill; rate-limit workers; budgets |
| **E-core throughput** | Lower frequency and narrower core: worker is slower per unit of work. | Lower throughput per core | Size worker pool for the backlog; accept longer deferral |

**Accounting rule.** Benchmark reports must charge worker CPU time to the technique (Paper 05 §6). Taking reclamation off the *mutator's* critical path reduces its tail, but total CPU is not necessarily lower. **[A]**

## 7. Race Prevention: Generation and Epoch

### 7.1 The races and their guards

| Race | Guard |
|------|-------|
| **ABA on slot reuse**: queue entry `{slot, gen=5}` outlives the slot's reuse as `gen=6` | Workers CAS on the full status word, which embeds `gen`; entries carry `gen`; mismatch → skip. |
| **Use-after-evict**: reader loaded `Alive`, evictor transitions + frees payload | Payload retired via EBR; freed only after all threads pass the epoch. Reader holds an epoch guard. |
| **Lost write during spill/evict** | Version re-check before publishing `Evicted`; write sets `DIRTY`/bumps version; evictor aborts or retries. |
| **Double hydrate** | `HYDRATING` flag claimed by CAS; losers wait or return `PENDING`. |
| **Delete vs evict vs expire concurrently** | All three are CAS from a specific expected state; exactly one wins; losers observe the new state and no-op. |
| **Master destroyed while workers hold entries** | `master_gen` in each entry; workers check before acting; Slaves are bulk-`Deleted` first. |
| **Worker acts on a slot whose Master was changed** | Slot→Master mapping is immutable for a slot's generation. |

### 7.2 Making `touch` cheap
Recording recency by writing a timestamp into the shared status word on every read would dirty the line on every access (a write per read). Instead:
- `touch_hint` writes to a **per-thread access ring** (or sets a bit in a per-thread CLOCK bitmap), and the eviction worker reads those. The shared status word is written only on state transitions. **[A]**
- Consequence: recency is approximate and delayed. Acceptable for eviction; not suitable for exact-LRU semantics. If exact LRU is required, accept the per-access write cost.

### 7.3 Capacity accounting
`resident_bytes` is sharded per core and summed lazily by the eviction worker; the check on insert reads a possibly stale sum. Overshoot is bounded by `cores × max_unflushed_delta`. Hard limits (OOM-sensitive) use a slower exact path near the limit. **[A]**

## 8. Failure and Edge Cases

| Case | Behavior |
|------|----------|
| **Expiry storm** (many TTLs aligned) | Jitter deadlines; cap per-tick expirations; defer remainder |
| **Eviction thrash** (evict then immediately rehydrate) | Admission control (don't admit one-hit-wonders); minimum residency after hydrate; coordinate with `CTMM` hysteresis |
| **Loader fails or is slow** | Bounded timeout; `get` returns `MISS`/error; slot remains `Evicted`; negative-caching option |
| **Loader not idempotent** | Contract requirement: idempotent and side-effect free; violation is a programmer error (documented) |
| **Backlog growth** (workers can't keep up) | Foreground help; scale workers; report `reclaim_backlog_bytes`; as last resort apply allocation backpressure |
| **Pinned slave** | Skipped by eviction; counted; may delay capacity enforcement |
| **Slave referenced across Masters** | Disallowed by scope rule; cross-Master references must use handles with `master_gen`, and are `STALE` on Master destruction |
| **Worker thread starvation** (priority/affinity) | Watchdog; deadline escalation to foreground help |
| **Clock source** | Monotonic coarse clock (e.g., updated by a ticker thread) so `now()` is a load, not a syscall |
| **Memory pressure while hydrating** | Hydrate requires allocation; reserve a small hydration pool; otherwise fail with `MISS` |
| **Crash safety** | Not provided; in-memory only |

## 9. Trade-offs

| Benefit | Cost |
|---------|------|
| Hot path: one CAS + enqueue for delete; no allocator work **[A]** | Deferred memory; cross-core traffic for status/queues **[A]** |
| Policy centralized, testable, auditable **[A]** | Programmers must classify objects into Masters; discipline about Slave scope |
| Explicit statuses make "evicted" and "deleted" distinct, enabling rehydration **[A]** | Every read pays a status check and an epoch guard |
| Bulk-delete via region death **[A]** | Mixed-lifetime Slaves in one region pin it |
| Natural integration point for `LastUseLifetime` `Defer` | Reclaim latency is nondeterministic (depends on worker scheduling) |

## 10. Relationship to Other Techniques

| Technique | Relationship |
|-----------|--------------|
| `LastUseLifetime` | Emits `Defer` into Master queues when release must be off-thread or policy-owned |
| `SegGenPool` | Slave storage substrate; `gen` shared with handles; region death for bulk delete |
| `TileCompact` | Evicted/Deleted slots shrink tile live counts, making tiles candidates; Master-owned relocation obeys `moving` protocol |
| `CTMM` | Overlap: `Evicted` (policy-driven, Master-owned) vs `Sublimation` (automatic, temperature-driven). Rule in Paper 05: a Master may *force* a state; otherwise CTMM manages tiers. `OPEN:` unify the two descriptors |
| `StallHarvest` | Workers compete with micro-steps for idle time; shared budget needed (Paper 05) |

## 11. Hypotheses

| ID | Hypothesis | Falsified if |
|----|------------|--------------|
| H2.1 | Offloading delete/reclaim lowers mutator p99 vs inline `free` on request-heavy workloads | Mutator p99 not lower at equal throughput |
| H2.2 | Per-thread touch rings reduce coherence misses vs shared timestamp writes | No difference in LLC/coherence counters or throughput |
| H2.3 | Foreground-help watermark keeps deferred bytes below a configured bound under bursty deletes | Backlog exceeds bound |
| H2.4 | Total CPU (mutator + workers) is within X% of inline reclamation | Total CPU exceeds budget |

## 12. Open Questions

- `OPEN:` Optimal status-word packing to separate read-mostly and CAS-hot bits across cache lines.
- `OPEN:` Whether `Expired` needs to be a distinct state or can be a flag on `Alive` plus a policy.
- `OPEN:` Worker sharding strategy under NUMA (shard by segment owner node vs by Master).
- `OPEN:` Unified representation of `Evicted` and `CTMM` Sublimation descriptors.
