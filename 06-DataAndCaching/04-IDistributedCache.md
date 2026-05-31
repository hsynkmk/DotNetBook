# IDistributedCache

## Caching shared across instances

`IDistributedCache` is an abstraction over an **out-of-process, shared** cache — most commonly **Redis** — so all instances of your app see the same cached data. Where `IMemoryCache` is per-instance (each server has its own copy), a distributed cache is the single source of cached truth across a load-balanced or scaled-out deployment.

```csharp
// Redis (the common choice)
builder.Services.AddStackExchangeRedisCache(o => {
    o.Configuration = builder.Configuration.GetConnectionString("Redis");
    o.InstanceName = "myapp:";
});

public class ProductService(IDistributedCache cache, IProductRepository repo) {
    public async Task<Product?> GetAsync(int id, CancellationToken ct) {
        string key = $"product:{id}";
        byte[]? cached = await cache.GetAsync(key, ct);
        if (cached is not null)
            return JsonSerializer.Deserialize<Product>(cached);   // hit: deserialize

        var product = await repo.GetAsync(id, ct);                 // miss: fetch
        if (product is not null)
            await cache.SetAsync(key, JsonSerializer.SerializeToUtf8Bytes(product),
                new() { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10) }, ct);
        return product;
    }
}
```

The key difference from `IMemoryCache`: the cache stores **`byte[]`** (you serialize/deserialize), and it's **shared** across all app instances over the network.

---

## Providers

`IDistributedCache` is an abstraction with several backends:

| Provider | Registration | Use |
|---|---|---|
| **Redis** | `AddStackExchangeRedisCache` | the standard — fast, feature-rich, widely deployed |
| **SQL Server** | `AddDistributedSqlServerCache` | when you already have SQL Server and don't want Redis |
| **In-memory** | `AddDistributedMemoryCache` | dev/testing only — NOT actually distributed (per-instance) |
| **Azure / others** | provider packages | cloud-managed caches |

**Redis** is the de-facto choice (in-memory speed, rich data structures, pub/sub, clustering). `AddDistributedMemoryCache` implements the *interface* but is per-process — useful for local dev/tests so code targets `IDistributedCache` uniformly, but it provides no actual sharing.

---

## API (byte-array based)

```csharp
await cache.GetAsync(key, ct);                    // byte[]? — null on miss
await cache.SetAsync(key, bytes, options, ct);    // store with expiration
await cache.RemoveAsync(key, ct);                 // evict
await cache.RefreshAsync(key, ct);                // reset sliding expiration without fetching

// String helpers (extension methods)
await cache.SetStringAsync(key, json, options, ct);
string? s = await cache.GetStringAsync(key, ct);
```

The interface deals in `byte[]` because the value crosses a process/network boundary — you must **serialize** on set and **deserialize** on get. `SetStringAsync`/`GetStringAsync` are convenience wrappers for string (e.g., JSON) values.

---

## Serialization — the cost you don't have with IMemoryCache

Every distributed-cache operation pays **serialization + network**:

```csharp
// Serialize on set, deserialize on get — choose an efficient serializer
byte[] bytes = JsonSerializer.SerializeToUtf8Bytes(product);          // JSON (readable, interoperable)
// or MessagePack / protobuf for compact, fast binary (high-throughput)
```

This is the fundamental trade-off vs `IMemoryCache`: a distributed cache is **shared and survives instance restarts**, but each access costs serialization + a network round-trip (sub-millisecond to Redis, but not free). So:
- Cache items worth the round-trip (expensive to compute/fetch, reused across instances).
- Use an efficient serializer (System.Text.Json with source-gen, or MessagePack/protobuf for hot paths — [Ch02 §05](../02-BCL/05-Serialization.md)).
- Don't cache tiny, cheap-to-recompute values in a distributed cache (the round-trip may cost more than recomputing).

---

## Expiration

```csharp
new DistributedCacheEntryOptions {
    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30),   // hard lifetime
    SlidingExpiration = TimeSpan.FromMinutes(5)                   // idle timeout (RefreshAsync resets it)
};
```

Same absolute/sliding model as `IMemoryCache`, enforced by the backend (Redis TTLs). `RefreshAsync` extends a sliding entry without re-reading the value. The cache provider evicts on TTL (and Redis can evict under memory pressure per its `maxmemory-policy`).

---

## Distributed cache vs IMemoryCache — choosing

| | IMemoryCache | IDistributedCache |
|---|---|---|
| Location | in-process | out-of-process (Redis/SQL) |
| Shared across instances | no (per-instance) | **yes** |
| Speed | fastest (no serialize/network) | fast, but serialize + network |
| Survives app restart | no | yes |
| Capacity | bounded by app memory | bounded by the cache server |
| Best for | hot, per-instance, small | shared state, large, cross-instance consistency |

Use **`IMemoryCache`** for hot per-instance data; **`IDistributedCache`** when instances must share cached data (consistency across servers, or a shared session/token store). Often you want **both** layered — which is exactly what **`HybridCache`** ([05-HybridCache.md](05-HybridCache.md)) provides (in-process L1 + distributed L2).

---

## Common uses for distributed cache

- **Shared session / token state** across instances (sticky-session-free scaling).
- **Expensive shared computations** reused by all instances.
- **Rate-limit counters** that must be global ([Ch04 §14](../04-AspNetCore/14-RateLimiting.md)).
- **Output cache backing store** for multi-instance consistency ([Ch04 §15](../04-AspNetCore/15-OutputCaching.md)).
- Reducing load on a shared database from many app instances.

---

## Common gotchas

### Treating it like IMemoryCache (forgetting serialization cost)

Every access serializes + does a network round-trip. Don't cache tiny cheap values, and use an efficient serializer. It's not free like in-process caching.

### `AddDistributedMemoryCache` in production

It's per-instance (not distributed) — fine for dev/tests, but in production it provides no sharing. Use Redis (or SQL) for real distribution.

### No stampede protection

Like `IMemoryCache`, the raw `IDistributedCache` doesn't prevent many instances/requests from all missing and recomputing simultaneously. Use `HybridCache` (stampede protection built in) or coordinate (a distributed lock).

### Caching huge objects

Large serialized blobs are slow to transfer and pressure the cache server. Cache reasonably-sized, frequently-reused data; consider compression for large values.

### Redis unavailability

If Redis is down, cache calls fail/timeout — make sure the app degrades gracefully (treat a cache miss/error as "fetch from source"), and apply resilience (timeouts, circuit breaker — [Ch11](../11-Resilience/README.md)). Don't let a cache outage take down the app.

### Stale data without invalidation

Updating the source without removing the cache key serves stale data until TTL. `RemoveAsync` on writes (and since it's shared, the invalidation applies to all instances — an advantage over per-instance caches).

---

## Summary

- **`IDistributedCache`** is an out-of-process, **shared** cache (usually **Redis**) so all app instances see the same data — vs `IMemoryCache`'s per-instance copies.
- It stores **`byte[]`** — every access pays **serialization + a network round-trip**; cache data worth that cost (expensive, reused, large) with an efficient serializer.
- Providers: **Redis** (standard), **SQL Server**, and `AddDistributedMemoryCache` (dev/test only — not actually distributed). Same absolute/sliding expiration model.
- Use it for **cross-instance consistency** (shared sessions/tokens, global counters, shared computations, output-cache backing); use `IMemoryCache` for hot per-instance data — or layer both via **`HybridCache`** ([05](05-HybridCache.md)).
- Mind serialization cost, lack of built-in stampede protection, and graceful degradation if the cache server is unavailable; invalidate on writes (shared invalidation is an advantage).

→ Next: [05-HybridCache.md](05-HybridCache.md)
