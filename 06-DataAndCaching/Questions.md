# Chapter 06 — Data & Caching — Q & A

---

### Q1. What is Dapper and when do you use it over EF Core?

A fast micro-ORM: you write SQL, it parameterizes and maps results to objects — no change tracking, migrations, or LINQ translation. Use it for read-heavy/complex/perf-critical queries where you want SQL control and minimal overhead; EF Core for rich domain modeling, change tracking, and migrations. Many apps use both (EF for writes, Dapper for reads).

---

### Q2. How does Dapper prevent SQL injection?

Pass parameters via an anonymous object (`new { category }`) — the properties become SQL parameters sent separately from the query text. Never concatenate user input into the SQL string. Same protection as EF/ADO.NET.

---

### Q3. What does `QueryMultiple` do?

Executes several queries in **one round-trip**, returning multiple result sets you read in sequence — efficient for fetching an aggregate (order + its lines) without N+1 or a complex join.

---

### Q4. What is ADO.NET's role?

The low-level data API (`DbConnection`/`DbCommand`/`DbDataReader`/`DbParameter`) that EF Core and Dapper are built on. You rarely use it directly, but it explains connection pooling, parameterization, and what the higher layers do.

---

### Q5. Explain ADO.NET connection pooling and its main rule.

Opening a physical connection is expensive, so ADO.NET pools them: `Open` rents from the pool, `Dispose`/`Close` returns it (doesn't physically close). The rule: **open late, dispose early** — holding connections open ties up the small pool (default ~100) and causes "timeout obtaining a connection." This is why EF/Dapper open per-operation.

---

### Q6. Why prefer explicit parameter types over `AddWithValue`?

`AddWithValue` lets the provider infer the DB type, which can be wrong (string length, decimal precision) — causing implicit conversions or plan-cache bloat. On hot paths, set `DbType`/size explicitly for correctness and performance.

---

### Q7. What is `IMemoryCache` and its primary method?

An in-process, per-instance cache (fastest — no serialization/network). The primary method is `GetOrCreateAsync`: return the cached value, or run the factory on a miss, cache it, and return. The factory (expensive fetch) runs only on a miss.

---

### Q8. Absolute vs sliding expiration?

**Absolute** expires at a fixed time/after a fixed duration regardless of access. **Sliding** expires only if not accessed within the window (each access resets the timer). Combine sliding with an absolute cap so a constantly-accessed item still refreshes eventually.

---

### Q9. Why must you bound `IMemoryCache`, and how?

It shares process memory — an unbounded cache (e.g., high-cardinality keys) OOMs the app. Set `SizeLimit` and give each entry a `Size`; the cache evicts by priority/expiration when over the limit. Always cap a production memory cache.

---

### Q10. What is cache stampede and does `GetOrCreateAsync` prevent it?

When a hot key expires, many concurrent requests all miss and run the expensive factory simultaneously (thundering herd), hammering the DB. Raw `IMemoryCache`/`IDistributedCache` `GetOrCreateAsync` does **not** prevent it (multiple factories run). `HybridCache` does (one factory per key; others await it).

---

### Q11. `IMemoryCache` vs `IDistributedCache`?

`IMemoryCache` is in-process, per-instance, fastest (no serialization/network), lost on restart. `IDistributedCache` is out-of-process (Redis/SQL), **shared across instances**, survives restarts, but each access costs serialization + a network round-trip. Use memory for hot per-instance data, distributed for cross-instance consistency.

---

### Q12. Why does `IDistributedCache` deal in `byte[]`?

The value crosses a process/network boundary, so it must be serialized. You serialize on `Set` and deserialize on `Get` (or use the string helpers) — every access pays serialization + network cost, unlike in-process `IMemoryCache`.

---

### Q13. Is `AddDistributedMemoryCache` actually distributed?

No — it implements the `IDistributedCache` interface but stores in-process (per-instance). It's for dev/testing so code targets the interface uniformly; use Redis (or SQL) for real cross-instance distribution in production.

---

### Q14. What is `HybridCache` and why is it the recommended default (.NET 9+)?

It combines in-process **L1** + optional distributed **L2** behind one `GetOrCreateAsync`, and adds **stampede protection**, **built-in serialization**, and **tag-based invalidation** — subsuming both raw caches and fixing what they lack. The recommended caching abstraction for new apps.

---

### Q15. How does `HybridCache` prevent stampede?

For a given key, only **one** factory execution runs under concurrent misses; the other callers await that single execution and share its result. So 1000 simultaneous requests for an expired key cause one DB hit, not 1000 — protecting the database from thundering herds.

---

### Q16. What does `HybridCache` L1+L2 lookup look like?

Check L1 (in-process, fastest) → on miss, check L2 (distributed, shared) → on full miss, run the factory and populate both layers. You get in-process speed for hot data and cross-instance sharing without manually coordinating two caches.

---

### Q17. How does `HybridCache` solve cross-instance staleness?

**Tag-based invalidation**: tag related entries, then `RemoveByTagAsync` on a write evicts them across L1 and L2 / all instances — so a write on one server invalidates the cache everywhere, fixing the per-instance-staleness problem of raw `IMemoryCache`.

---

### Q18. Output caching vs data caching — what's the difference?

**Output caching** caches whole HTTP **responses** server-side (skips routing/handler/DB/serialization) — keyed by the request. **Data caching** (`IMemoryCache`/`HybridCache`/`IDistributedCache`) caches **values/objects your code stores** by your key, reused inside handlers. Different levels; they layer.

---

### Q19. Output caching vs the older response caching?

Output caching (.NET 7+) is **server-controlled** and supports **tag invalidation**. Response caching only set HTTP cache headers and relied on the client/proxy to honor them (no server-side eviction). Prefer output caching for response-level caching.

---

### Q20. First-level vs second-level cache in EF Core?

First-level (built in) is the change tracker / identity map within one `DbContext` — dedupes loads within that context. Second-level (a community interceptor) caches query **results across** contexts/requests so identical queries skip the database; it's not built into EF.

---

### Q21. How does EFCoreSecondLevelCacheInterceptor handle invalidation, and what are its limits?

It tracks which tables a cached query touched and evicts related entries when `SaveChanges` writes those tables. Limits: invalidation is **table-granular** (any write to a table evicts all its cached queries) and misses **out-of-band** writes (`ExecuteUpdate`/raw SQL/other apps that bypass `SaveChanges`) → stale cache. Works best when EF is the sole writer of read-mostly tables.

---

### Q22. Should you use EF second-level caching or explicit caching?

Usually **explicit** (`HybridCache.GetOrCreateAsync` around specific queries, or output caching) — precise control over what's cached, for how long, and when it's invalidated. Reach for a second-level interceptor only when many identical read-heavy queries make per-query caching impractical and the tables are EF-written-only.

---

### Q23. Why cache only immutable data?

A cached object is shared by all callers (especially in-process — L1 returns the same instance). If one caller mutates it, everyone sees the change and it's not re-fetched. Cache immutable data (or copies) to avoid corruption.

---

### Q24. How should an app behave if Redis (distributed cache) is down?

Degrade gracefully — treat a cache error/miss as "fetch from the source" so a cache outage doesn't take down the app. Apply resilience (timeouts, circuit breaker — [Ch11](../11-Resilience/README.md)). Don't let cache unavailability cascade into request failures.

---

### Q25. How do you prevent cache penetration for missing items?

Cache **negative results** (item-not-found) for a **short** time, so repeated requests for a missing key don't all hit the database — but keep it short so a newly-created item becomes visible quickly. Caching nulls forever hides new items; never caching them causes penetration.

---

→ Next: [Coding.md](Coding.md)
