# IMemoryCache

## In-process caching

`IMemoryCache` stores objects in the app's own memory, keyed by a string (or any object key). It's the fastest cache — no serialization, no network — for data you read far more than you compute/fetch: reference data, expensive query results, computed values. It lives in the process, so it's per-instance (not shared across servers).

```csharp
builder.Services.AddMemoryCache();

public class ProductService(IMemoryCache cache, IProductRepository repo) {
    public async Task<Product?> GetAsync(int id, CancellationToken ct) =>
        await cache.GetOrCreateAsync($"product:{id}", async entry => {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
            return await repo.GetAsync(id, ct);                  // only runs on cache miss
        });
}
```

`GetOrCreateAsync` is the workhorse: return the cached value if present, otherwise run the factory, cache the result, and return it. The factory (the expensive fetch) runs **only on a miss**.

---

## Core API

```csharp
cache.TryGetValue(key, out Product? p);                    // check without creating
cache.Set(key, value, TimeSpan.FromMinutes(5));            // set with expiration
var v = cache.GetOrCreate(key, e => { e.SlidingExpiration = ...; return Compute(); });
await cache.GetOrCreateAsync(key, async e => await FetchAsync());
cache.Remove(key);                                          // explicit eviction
```

`GetOrCreate`/`GetOrCreateAsync` is the pattern you'll use most. `Set` for write-through; `TryGetValue` to peek; `Remove` to invalidate on updates.

---

## Expiration policies

```csharp
cache.Set(key, value, new MemoryCacheEntryOptions {
    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30),   // expires 30 min after set, no matter what
    SlidingExpiration = TimeSpan.FromMinutes(5),                  // expires if not accessed for 5 min
    Priority = CacheItemPriority.High,                            // evicted last under memory pressure
    Size = 1                                                       // for size-limited caches
});
```

- **Absolute expiration** — entry expires at a fixed time/after a fixed duration, regardless of use. Use for data with a known freshness window.
- **Sliding expiration** — expires only if not accessed within the window; each access resets the timer. Keeps hot items alive, drops cold ones.
- **Combine them** — sliding with an absolute cap (e.g., "expire after 5 min idle, but at most 1 hour total") so a constantly-accessed item still refreshes eventually.

```csharp
new MemoryCacheEntryOptions {
    SlidingExpiration = TimeSpan.FromMinutes(5),
    AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)   // hard cap
};
```

---

## Size limits & memory pressure

`IMemoryCache` shares your process memory — an **unbounded** cache is a memory leak waiting to OOM the app. Bound it:

```csharp
builder.Services.AddMemoryCache(o => o.SizeLimit = 10_000);   // total "size" units

cache.Set(key, value, new MemoryCacheEntryOptions { Size = 1 });   // each entry MUST declare a Size
```

When a `SizeLimit` is set, **every entry must specify a `Size`** (or `Set` throws), and the cache evicts entries (by priority, then expiration) when the total exceeds the limit. "Size" is a unit you define (count, bytes, weight). Always cap a memory cache in production — caching unbounded keys (e.g., per-user data with high cardinality) without a limit will exhaust memory.

`CacheItemPriority` (`Low`/`Normal`/`High`/`NeverRemove`) influences eviction order under pressure.

---

## Eviction callbacks & change tokens

```csharp
cache.Set(key, value, new MemoryCacheEntryOptions()
    .RegisterPostEvictionCallback((k, v, reason, state) =>
        logger.LogDebug("Evicted {Key}: {Reason}", k, reason))    // observe evictions
    .AddExpirationToken(changeToken));                             // expire when a token signals
```

- **Post-eviction callbacks** let you react when an entry is removed (log, refresh, clean up).
- **Change tokens** (`IChangeToken`) expire an entry when an external source changes — e.g., a config-file change token expires cached config (this is how `IOptionsMonitor` reloads work — [Ch03 §08](../03-HostingAndDI/08-Options.md)). Use them to tie cache lifetime to a source of truth rather than only time.

---

## Caching nulls / negative results

A subtle pitfall: if the factory returns `null` (item not found), `GetOrCreateAsync` caches the `null` by default in some patterns — or, if you skip caching nulls, every request for a missing item hits the database (cache penetration). Decide deliberately:

```csharp
var product = await cache.GetOrCreateAsync($"product:{id}", async entry => {
    var p = await repo.GetAsync(id, ct);
    entry.AbsoluteExpirationRelativeToNow = p is null
        ? TimeSpan.FromSeconds(30)     // cache "not found" briefly (prevent penetration) ...
        : TimeSpan.FromMinutes(10);    // ... but cache real results longer
    return p;
});
```

Cache negative results for a **short** time to prevent repeated DB hits for missing keys, but not so long that a newly-created item stays invisible.

---

## When to use IMemoryCache

- **Reference/lookup data** read constantly, changed rarely (categories, config, feature flags).
- **Expensive computations / query results** reused across requests.
- **Per-instance** caching where you don't need cross-server sharing.

When **not** to: data that must be **consistent across instances** (use a distributed cache — [04-IDistributedCache.md](04-IDistributedCache.md)), or where stale data is unacceptable. And `IMemoryCache` is **per-instance** — in a load-balanced deployment, each server has its own copy (different freshness, no shared invalidation).

---

## Common gotchas

### Unbounded cache → OOM

Caching high-cardinality keys without a `SizeLimit` exhausts memory. Set a limit and declare per-entry `Size`; cap what you cache.

### Caching mutable objects

A cached object is shared by all callers. If one caller mutates it, everyone sees the change (and it's not re-fetched). Cache **immutable** data (or copies), not objects callers will mutate.

### Per-instance inconsistency

With multiple servers, each has its own `IMemoryCache` — an update on one doesn't invalidate the others (stale data across instances). Use a distributed cache (or HybridCache) when consistency across instances matters.

### Cache stampede

When a hot entry expires, many concurrent requests all miss and run the expensive factory simultaneously. `GetOrCreateAsync` does **not** by itself prevent this (multiple factories can run). For stampede protection, use **`HybridCache`** ([05-HybridCache.md](05-HybridCache.md)) or guard with a lock/semaphore per key.

### Caching nulls forever (or never)

Caching "not found" forever hides newly-created items; never caching it causes penetration (every miss hits the DB). Cache negatives briefly.

### Forgetting to invalidate on writes

Updating the underlying data without `cache.Remove(key)` serves stale data until expiration. Evict on write for freshness.

---

## Summary

- **`IMemoryCache`** is the fastest, in-process, per-instance cache (no serialization/network) — use `GetOrCreateAsync` (factory runs only on a miss) for expensive fetches/computations and rarely-changing reference data.
- Control freshness with **absolute** (fixed lifetime) and **sliding** (idle-based) expiration — combine sliding + an absolute cap; **change tokens** tie expiry to a source of truth.
- **Always bound it** (`SizeLimit` + per-entry `Size`) — an unbounded cache OOMs the app; use priority for eviction order. Cache **immutable** data; **invalidate on writes**; cache negatives briefly.
- It's **per-instance** — for cross-server consistency use a **distributed cache** ([04](04-IDistributedCache.md)); `GetOrCreateAsync` doesn't prevent **cache stampede** — use **`HybridCache`** ([05](05-HybridCache.md)) for that.

→ Next: [04-IDistributedCache.md](04-IDistributedCache.md)
