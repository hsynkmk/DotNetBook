# EF Core Second-Level Caching

## Caching query results automatically

EF Core has a **first-level cache** (the change tracker / identity map within one `DbContext` — [Ch05 §03](../05-EFCore/03-ChangeTracking.md)): within a single context, querying the same entity twice returns the tracked instance without a second DB hit. But that cache lives only for the context's (short) lifetime. A **second-level cache** persists query *results* **across** contexts and requests, so identical queries don't re-hit the database — implemented in EF Core via a community interceptor like **EFCoreSecondLevelCacheInterceptor**.

```csharp
builder.Services.AddEFSecondLevelCache(o => o
    .UseMemoryCacheProvider()                          // or a Redis provider for distributed
    .ConfigureLogging(false)
    .UseCacheKeyPrefix("EF_"));

builder.Services.AddDbContext<AppDbContext>((sp, o) => o
    .UseNpgsql(cs)
    .AddInterceptors(sp.GetRequiredService<SecondLevelCacheInterceptor>()));

// Mark a query as cacheable:
var products = await db.Products.Where(p => p.Active).Cacheable().ToListAsync(ct);
//   first call → SQL + cache the result; subsequent identical queries → served from cache
```

The interceptor caches the materialized results of marked (or, optionally, all) queries, keyed by the generated SQL + parameters, and serves them on identical subsequent queries.

---

## First-level vs second-level cache

| | First-level (built in) | Second-level (interceptor) |
|---|---|---|
| Scope | one `DbContext` instance | across contexts/requests |
| Caches | tracked entities (identity map) | query results |
| Lifetime | the context's lifetime | configurable (TTL) |
| Built into EF | yes | no — community package |

EF's **first-level** cache prevents duplicate loads *within* a context (you query `Find(42)` twice → one DB hit). It does **nothing** across requests — each request's new context re-runs queries. The **second-level** cache fills that gap, caching results so the same query across different contexts/requests skips the database.

---

## When it helps

- **Read-heavy, rarely-changing data** queried identically across many requests — lookup tables, configuration, catalogs, reference data.
- Reducing database load for popular queries that return the same results repeatedly.
- Apps where the same parameterized query runs constantly (e.g., "active products in category X").

The win: identical queries are served from cache (memory or Redis) instead of round-tripping to the database — like output caching but at the **query** level rather than the response level.

---

## The hard part: invalidation

The challenge with caching query results is keeping them **fresh** when the underlying data changes. EFCoreSecondLevelCacheInterceptor handles this by **tracking which tables a cached query touched** and **automatically invalidating** related cache entries when `SaveChanges` modifies those tables:

```csharp
// Cached query touches the Products table:
var products = await db.Products.Where(p => p.Active).Cacheable().ToListAsync(ct);

// A write to Products invalidates the cached query automatically:
db.Products.Add(newProduct);
await db.SaveChangesAsync(ct);   // interceptor evicts cached queries involving Products
```

This automatic, table-based invalidation is the feature that makes second-level caching practical — you don't manually track which cache entries to evict. Caveats: invalidation is **table-granular** (any write to a table invalidates *all* cached queries on it, even unrelated rows), and **out-of-band changes** (another app, raw SQL, or `ExecuteUpdate`/`ExecuteDelete` which bypass `SaveChanges`) won't trigger invalidation → stale cache. So it works best when EF is the **sole writer** of the cached tables.

---

## Should you use it? (weigh carefully)

Second-level caching is powerful but adds **complexity and staleness risk**. Often **simpler, more targeted caching is better**:

| Need | Often better than EF 2nd-level cache |
|---|---|
| Cache a few specific expensive query results | **`HybridCache.GetOrCreateAsync`** around the query (explicit, controllable — [05](05-HybridCache.md)) |
| Cache whole endpoint responses | **output caching** ([06](06-OutputCaching.md)) |
| Cache reference/lookup data | `IMemoryCache`/`FrozenDictionary` loaded at startup |

The trade-off: EF second-level cache is **transparent** (mark a query `.Cacheable()`, get caching + auto-invalidation), but it's **broad and implicit** (you cache at the query layer with table-granular invalidation and staleness risk from out-of-band changes). **Explicit caching** (`HybridCache` around a specific call) gives you precise control over what's cached, for how long, and when it's invalidated — usually clearer and safer.

**Recommendation**: prefer **explicit caching** (`HybridCache`/output caching) for most needs — you know exactly what's cached and when it's evicted. Reach for a second-level cache interceptor when you have **many** identical read-heavy queries across the app that would be tedious to cache individually, the cached tables are EF-written-only, and table-granular invalidation is acceptable.

---

## Common gotchas

### Stale cache from out-of-band writes

`ExecuteUpdate`/`ExecuteDelete`, raw SQL, or another application modifying the tables bypass `SaveChanges`, so the interceptor doesn't invalidate → stale results. Use second-level caching only when EF is the sole writer (or invalidate manually).

### Table-granular invalidation over-evicts

Any write to a cached table evicts *all* cached queries on it (even for unrelated rows) — for write-heavy tables this defeats the cache. Best for read-mostly tables.

### Caching everything

Enabling caching for *all* queries (not just `.Cacheable()` ones) caches volatile/personalized queries too → staleness and memory bloat. Cache deliberately (mark specific queries).

### Hidden complexity / debugging

Implicit query caching makes "why am I seeing stale data?" harder to diagnose. Explicit caching is more transparent. Prefer explicit unless the breadth justifies it.

### Distributed consistency

With a memory-cache provider, the second-level cache is per-instance (stale across servers). Use a Redis provider for multi-instance consistency.

---

## Summary

- EF's **first-level cache** (the change tracker) dedupes loads **within** one `DbContext`; a **second-level cache** (community interceptor like **EFCoreSecondLevelCacheInterceptor**) caches **query results across** contexts/requests so identical queries skip the database.
- It auto-invalidates by **tracking the tables** a query touched and evicting on `SaveChanges` to those tables — but invalidation is **table-granular** and misses **out-of-band** writes (`ExecuteUpdate`/raw SQL/other apps) → works best when EF is the sole writer of read-mostly tables.
- It's transparent (`.Cacheable()`) but broad and staleness-prone; **prefer explicit caching** (`HybridCache.GetOrCreateAsync` around specific queries, or **output caching**) for precise control. Use second-level caching only when many identical read-heavy queries make per-query caching impractical.
- Use a Redis provider for cross-instance consistency; cache deliberately, not everything.

→ Next: [Questions.md](Questions.md)
