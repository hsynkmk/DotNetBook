# Common Wins — The Usual Suspects

## Where the bottlenecks actually are

After profiling enough .NET apps, the same culprits recur. This file catalogs the **usual suspects** — high-frequency, high-impact problems you'll find again and again — so that once a profiler ([04-DotnetTrace.md](04-DotnetTrace.md), [05-DotnetDump.md](05-DotnetDump.md)) points you at a hot spot, you recognize the pattern and know the fix. **Still measure first** ([01-Mindset.md](01-Mindset.md)) — but these are where the wins usually are. They cluster into a few themes: **allocations/GC pressure**, **database access**, **async misuse**, and **doing avoidable work**.

---

## 1. Excessive allocations (GC pressure)

The #1 .NET performance issue is **too many allocations** in hot paths, which drive GC ([Ch01 §04](../01-Runtime/04-GCDeepDive.md)) — visible as high allocation rate / frequent Gen2 in counters ([03-DotnetCounters.md](03-DotnetCounters.md)). Common offenders and fixes:

- **String concatenation in loops** — `s += x` is O(n²) allocations. Use **`StringBuilder`** or `string.Join` ([Ch02 §01](../02-BCL/01-Strings.md)).
- **LINQ in hot loops** — each `Where`/`Select` allocates iterators/closures; in a tight hot path, a plain loop avoids them ([CSharpBook Ch06](../../CSharpBook/06-LINQ/README.md)).
- **Boxing** value types (e.g., into `object`/non-generic APIs) — allocates; use generics ([CSharpBook Ch03 §07](../../CSharpBook/03-TypeSystem/README.md)).
- **Unnecessary intermediate collections** — `.ToList()` then iterate; stream instead.
- **Large/repeated buffers** — reuse via **`ArrayPool<T>`**, and use **`Span<T>`/`Memory<T>`** to slice without copying ([CSharpBook Ch09 §05](../../CSharpBook/09-MemoryPerformance/README.md)).

The fix theme: **allocate less** on hot paths — `Span`, pooling, `StringBuilder`, avoiding LINQ/boxing where it's hot. Allocation profiling ([04-DotnetTrace.md](04-DotnetTrace.md)) finds the worst offenders.

---

## 2. Database: N+1 and over-fetching

Data access is the most common *whole-app* bottleneck (and often dwarfs CPU micro-optimizations):

- **N+1 queries** — iterating entities and lazily loading a relation per item → 1 + N queries instead of 1. Fix with **eager loading** (`Include`) or a projection ([Ch05 §02](../05-EFCore/02-Querying.md)). The **VS Database profiler** ([07-VisualStudioProfiler.md](07-VisualStudioProfiler.md)) exposes this instantly (a flood of identical small queries).
- **Over-fetching** — `SELECT *`/loading whole entities when you need two columns. **Project** to a DTO (`Select`) so the DB returns only what's needed ([Ch05 §09](../05-EFCore/09-Performance.md)).
- **Missing `AsNoTracking`** — read-only queries pay change-tracking overhead; add `AsNoTracking()` for reads ([Ch05 §03](../05-EFCore/03-ChangeTracking.md)).
- **Missing indexes** — a slow query is often a missing index, not slow code. Check the query plan.
- **Chatty round-trips** — many small queries; batch or restructure.

Database wins usually beat CPU micro-tuning by orders of magnitude — profile the **Database** dimension early for data apps.

---

## 3. Async misuse

- **Sync-over-async** — `.Result` / `.Wait()` / `.GetAwaiter().GetResult()` blocks a thread waiting on async work, causing **thread-pool starvation** under load (growing thread count + queue in counters — [03-DotnetCounters.md](03-DotnetCounters.md); blocked threads in a dump — [05-DotnetDump.md](05-DotnetDump.md)). Fix: **async all the way** — `await` instead of blocking ([CSharpBook Ch08](../../CSharpBook/08-Concurrency/README.md)).
- **Not using async for I/O** — synchronous I/O ties up threads. Use the async APIs ([Ch02 §04](../02-BCL/04-IO.md)).
- **Async over trivial sync work** — wrapping CPU-trivial work in `Task.Run` adds overhead without benefit.
- **Missing `CancellationToken` flow** — work continues after the caller gave up, wasting resources ([Ch03 §01](../03-HostingAndDI/01-GenericHost.md)).

Thread-pool starvation from sync-over-async is one of the most common "the app is mysteriously slow/unresponsive under load" causes.

---

## 4. Caching what you recompute / refetch

Repeatedly computing or fetching the same thing is wasted work. Add a cache at the right layer ([Ch06](../06-DataAndCaching/README.md)):

- **In-memory cache** (`IMemoryCache`) for hot, per-instance data; **distributed/HybridCache** for shared/scaled data (with stampede protection — [Ch06 §05](../06-DataAndCaching/05-HybridCache.md)).
- **Output caching** for whole responses ([Ch04 §15](../04-AspNetCore/15-OutputCaching.md)).
- **Memoize** expensive pure computations.

But cache deliberately — wrong invalidation causes stale-data bugs. Cache the things profiling shows are recomputed/refetched hot.

---

## 5. Doing avoidable work

- **Reflection in hot paths** — cache `MethodInfo`/`PropertyInfo` or use source generators ([Ch01 / CSharpBook Ch12](../../CSharpBook/12-Reflection/README.md)); reflection per call is slow.
- **Logging too much / expensive log args** — string-interpolating log messages that are filtered out, or logging in tight loops. Use structured logging with level checks ([Ch12 §02](../12-Observability/02-ILogger.md)).
- **Serializing more than needed** / reflection-based serialization in hot paths — use System.Text.Json **source generation** ([Ch02 §05](../02-BCL/05-Serialization.md)).
- **Exceptions for control flow** — throwing/catching is expensive (high exception count in counters — [03-DotnetCounters.md](03-DotnetCounters.md)); use `Try...` patterns / result types for expected cases ([CSharpBook Ch17](../../CSharpBook/17-BestPractices/README.md)).
- **Regex compiled per call** — use a source-generated/compiled, cached `Regex`.

---

## The recurring themes

```
Allocate less    → Span/Memory, ArrayPool, StringBuilder, avoid LINQ/boxing on hot paths
Touch the DB less→ fix N+1 (Include/project), AsNoTracking, index, batch
Don't block      → async all the way; no .Result/.Wait()
Don't repeat     → cache recomputed/refetched results
Don't do junk    → cache reflection, gate logging, source-gen serialization, no exceptions-as-flow
```

Most real-world .NET performance wins come from these five themes — but **which** one applies is what profiling tells you. Recognize the pattern at the hot spot, apply the matching fix, **re-measure** ([01-Mindset.md](01-Mindset.md)).

---

## Common gotchas

### Fixing a "usual suspect" that isn't the bottleneck

These are *likely* culprits, not *certain* ones. Optimizing string concat when the bottleneck is an N+1 query wastes effort. **Profile first**; match the fix to the proven hot spot.

### Micro-optimizing allocations while ignoring the DB

A data app's bottleneck is usually database access (N+1/over-fetch), which dwarfs allocation micro-tuning. Check the DB dimension before micro-optimizing CPU/allocations.

### "Async everywhere" without understanding

Async helps **I/O** (frees threads); wrapping trivial CPU work in `Task.Run` adds overhead. And the real killer is **sync-over-async** (`.Result`) — fix blocking, don't cargo-cult async.

### Over-caching

Caching everything causes stale-data bugs and memory growth. Cache what's *proven* hot, with correct invalidation/expiry ([Ch06](../06-DataAndCaching/README.md)).

### Re-measuring skipped

After applying a fix, **re-measure** to confirm it helped (and didn't regress). An "optimization" that doesn't move the metric should be reverted ([01-Mindset.md](01-Mindset.md)).

---

## Summary

- The recurring .NET bottlenecks cluster into five themes: **excessive allocations** (GC pressure — fix with `Span`/`ArrayPool`/`StringBuilder`, avoid LINQ/boxing on hot paths), **database** (N+1/over-fetch — `Include`/project/`AsNoTracking`/index — often the biggest win), **async misuse** (**sync-over-async** → thread-pool starvation — go async all the way), **missing caching** (cache recomputed/refetched results — [Ch06](../06-DataAndCaching/README.md)), and **avoidable work** (reflection, over-logging, exceptions-as-flow, reflection serialization).
- These are **likely** suspects to recognize once profiling points at a hot spot — **not** an excuse to skip measuring ([01-Mindset.md](01-Mindset.md)).
- For **data apps**, check the **database** dimension first (N+1 via the VS Database profiler — [07](07-VisualStudioProfiler.md)) — it usually dwarfs CPU/allocation micro-tuning.
- Apply the fix matching the proven bottleneck, then **re-measure** to confirm the win.

→ Next: [Questions.md](Questions.md)
