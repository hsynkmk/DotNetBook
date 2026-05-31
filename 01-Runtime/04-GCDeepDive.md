# Garbage Collection Deep Dive

## What the GC does

The garbage collector automatically reclaims memory occupied by objects your program can no longer reach. You allocate with `new`; you (almost) never free. The GC periodically finds unreachable objects and reclaims their memory, compacting the heap to keep allocation fast.

> CSharpBook Chapter 09 covers GC from the **language/usage** angle (when objects die, `IDisposable`, finalizers, avoiding leaks). This file covers the **runtime internals**: generations, segments, write barriers, GC modes, and the .NET 10 defaults.

---

## The generational hypothesis

The GC's design rests on an empirical observation: **most objects die young.** A temporary string, a LINQ iterator, a request DTO — allocated, used, discarded almost immediately. A few objects live long (caches, singletons, the DI container).

So the GC partitions the heap into **generations** and collects the young ones often (cheap) and the old ones rarely (expensive):

```
Gen 0   — newest objects. Collected very frequently, very fast.
Gen 1   — survived one+ Gen 0 collection. A "buffer" between young and old.
Gen 2   — long-lived objects. Collected rarely (full collection).

LOH     — Large Object Heap (objects ≥ 85,000 bytes). Collected with Gen 2.
POH     — Pinned Object Heap (.NET 5+). For pinned objects, kept separate to avoid fragmenting the normal heap.
```

A **Gen 0 collection** only looks at the youngest objects — fast (sub-millisecond typically). Survivors are *promoted* to Gen 1, then Gen 2. A **Gen 2 (full) collection** examines everything — the expensive one you want to minimize.

---

## How a collection works (mark-compact)

1. **Suspension** — managed threads are brought to safepoints (see [01-CoreCLRArchitecture.md](01-CoreCLRArchitecture.md)).
2. **Mark** — starting from **roots** (statics, stack locals, CPU registers, GC handles, finalization queue), the GC traces all reachable objects, marking them live. Anything unmarked is garbage.
3. **Compact** (for the collected generation) — live objects are moved together to eliminate gaps; the rest of the segment becomes free space. Moving objects means **updating every reference** to them (the JIT's GC info tells the GC where references live).
4. **Resume** — threads continue. The next allocation is a cheap pointer bump into the freed contiguous space.

Compaction is why .NET allocation is so fast (bump-a-pointer) but also why the GC must suspend threads (it's moving objects out from under them).

The **LOH is not compacted by default** (moving huge objects is expensive), so it can fragment — a reason to pool large buffers (`ArrayPool`) rather than churn them.

---

## Roots — what keeps an object alive

An object is live if it's reachable from a **root**:
- **Static fields**
- **Local variables / method arguments** currently on a thread's stack (and in registers)
- **GC handles** (e.g., `GCHandle`, pinned handles, the finalization queue)
- Objects referenced by any of the above, transitively.

When the last root reference is gone, the object becomes collectible. This is the mental model behind memory leaks: an unintended root (a static event handler, a cache that never evicts, a captured closure) keeps objects alive forever. (Leak hunting: CSharpBook Ch09 §13 and [Chapter 21](../21-Performance/README.md).)

---

## Workstation vs Server GC

Two GC flavors tuned for different workloads:

| | Workstation GC | Server GC |
|---|---|---|
| Heaps | one | **one per logical core** |
| GC threads | runs on the app thread (mostly) | **dedicated GC threads**, parallel |
| Goal | low latency, low memory (client apps) | **throughput** (servers) |
| Default for | console/desktop | ASP.NET Core / server apps |

```xml
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>     <!-- Server GC -->
  <ConcurrentGarbageCollection>true</ConcurrentGarbageCollection> <!-- background GC -->
</PropertyGroup>
```

**Server GC** uses multiple heaps collected in parallel by dedicated threads — higher throughput and better scaling on many-core machines, at the cost of more memory (a heap per core). **Workstation GC** is leaner — better for client apps and low-memory containers. ASP.NET Core defaults to Server GC.

---

## Background (concurrent) GC

A full Gen 2 collection could cause a long pause. **Background GC** does most of the Gen 2 marking **concurrently** with the application threads still running, drastically shortening the stop-the-world portion. Gen 0/1 collections can still happen during a background Gen 2. This is on by default and is why modern .NET avoids the long GC pauses of the old days.

---

## DATAS — Dynamically Adapting To Application Sizes

A major modern change (opt-in .NET 7 → default .NET 8/9, refined in .NET 10). Traditionally Server GC sized its heaps based on core count, so a small app on a 64-core box reserved a lot of memory. **DATAS** instead sizes the heap to the application's **actual working set**, adapting as load changes:

- Uses smaller, dynamically managed **regions** instead of large fixed segments.
- Adds/removes heap capacity based on real allocation pressure.
- Cuts memory use 20–40% for small/medium apps with comparable throughput.

```xml
<!-- DATAS is the default in modern Server GC; disable if a throughput-critical
     large-allocator workload measures worse with it -->
<PropertyGroup>
  <GarbageCollectionAdaptationMode>0</GarbageCollectionAdaptationMode>
</PropertyGroup>
```

For most services (especially containerized microservices), DATAS means lower, more predictable memory. (Also covered in CSharpBook Ch11 §08.)

---

## Write barriers — how the GC tracks cross-generation references

Problem: a Gen 0 collection only scans Gen 0, but a long-lived **Gen 2 object might reference a young Gen 0 object** (e.g., you store a fresh object into a cached list's field). If the GC ignored Gen 2, it might wrongly collect that young object.

Solution: a **write barrier**. Every time managed code writes an object reference into a field (`obj.Field = other`), the JIT emits a tiny bit of extra code (the barrier) that records the write in a **card table** — a compact map marking which regions of old generations were modified. During a Gen 0 collection, the GC also scans those marked cards, treating modified old-gen objects as additional roots.

Implications:
- Reference-field writes have a small hidden cost (the barrier). Value-type writes (no object reference) don't.
- .NET 10's runtime work includes **reduced write-barrier** costs — part of why each version gets faster for free.
- This is invisible to you, but it's why "lots of objects mutating references" can pressure the GC more than immutable data.

---

## Finalization and `IDisposable` (runtime view)

Objects with a finalizer (`~Type()`) aren't collected immediately — they're put on the **finalization queue**, their finalizer runs on a dedicated finalizer thread, and only then is the memory reclaimed. This means finalizable objects survive an **extra GC cycle** and add latency. Hence: prefer `IDisposable` for deterministic cleanup, use finalizers only as a safety net (and `SafeHandle` instead of raw finalizers). Full treatment: CSharpBook Ch09 §03–04.

---

## Observing the GC

```bash
dotnet-counters monitor -p <pid>     # gc-heap-size, gen-0/1/2-gc-count, time-in-gc, alloc-rate
dotnet-trace collect -p <pid> --profile gc-verbose
dotnet-gcdump collect -p <pid>       # heap snapshot: object counts + retention paths
```

In a dump: `eeheap -gc` (segments/generations), `dumpheap -stat` (by type), `gcroot <addr>` (why an object is alive). Watch **time-in-gc** and **Gen 2 frequency** — high values mean GC pressure (too much allocation or a leak). See [Chapter 21](../21-Performance/README.md).

---

## Common gotchas

### Treating GC pauses as free

Server GC + background GC make pauses small but not zero. High allocation rates → frequent Gen 0 → measurable tail latency. Reduce allocations on hot paths (`Span`, pooling, structs) — CSharpBook Ch17 §03.

### LOH fragmentation

Large arrays churned repeatedly fragment the (non-compacted) LOH. Pool them with `ArrayPool<T>`.

### Accidental Gen 2 promotion (mid-life crisis)

Objects that live "medium" long get promoted to Gen 2 and then die, forcing expensive Gen 2 work. Caches with churn are the classic cause; tune eviction.

### Wrong GC mode for the workload

Server GC in a tiny 1-core container wastes memory; Workstation GC on a busy many-core server limits throughput. Match the mode (and rely on DATAS for sizing).

---

## Summary

- The GC is **generational** (Gen 0/1/2 + LOH + POH) exploiting "most objects die young": collect young often (cheap), old rarely (expensive full GC).
- Collection is **mark-compact**: trace from roots, move survivors together, update references (using the JIT's GC info); allocation is then a cheap pointer bump.
- **Workstation vs Server GC** (one heap vs per-core heaps), with **background GC** doing Gen 2 marking concurrently to shrink pauses.
- **DATAS** (default in modern Server GC) sizes the heap to the real working set — big memory savings for typical services.
- **Write barriers + card tables** let young collections find references from old objects; reference writes carry a small cost.
- Minimize allocations and watch `time-in-gc`/Gen 2 frequency; usage-level guidance is in CSharpBook Ch09.

→ Next: [05-TypeSystem.md](05-TypeSystem.md)
