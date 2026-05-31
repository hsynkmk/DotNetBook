# Chapter 06 — Data & Caching — Coding Problems

Build a Dapper repository, layer caching correctly, prevent a stampede, and wire distributed caching with graceful degradation.

---

## Problem 1: A Dapper repository

Build a repository for `Product` with get-by-id, list-by-category, and create — parameterized and async.

<details><summary>Solution</summary>

```csharp
public class ProductRepository(IDbConnectionFactory factory) {
    public async Task<Product?> GetAsync(int id, CancellationToken ct) {
        using var conn = await factory.CreateConnectionAsync(ct);
        return await conn.QuerySingleOrDefaultAsync<Product>(new CommandDefinition(
            "SELECT Id, Name, Price, Category FROM Products WHERE Id = @id", new { id }, cancellationToken: ct));
    }

    public async Task<IReadOnlyList<Product>> GetByCategoryAsync(string category, CancellationToken ct) {
        using var conn = await factory.CreateConnectionAsync(ct);
        var rows = await conn.QueryAsync<Product>(new CommandDefinition(
            "SELECT Id, Name, Price, Category FROM Products WHERE Category = @category ORDER BY Name",
            new { category }, cancellationToken: ct));
        return rows.ToList();
    }

    public async Task<int> CreateAsync(CreateProduct p, CancellationToken ct) {
        using var conn = await factory.CreateConnectionAsync(ct);
        return await conn.ExecuteScalarAsync<int>(new CommandDefinition(
            "INSERT INTO Products (Name, Price, Category) VALUES (@Name, @Price, @Category) RETURNING Id",
            p, cancellationToken: ct));
    }
}
```

Parameterized (injection-safe), async with `CancellationToken` via `CommandDefinition`, connection-per-operation (pooled). ([01-Dapper.md](01-Dapper.md).)

</details>

---

## Problem 2: Cache an expensive read with IMemoryCache

Cache product lookups with a 10-minute lifetime, caching "not found" briefly to prevent penetration.

<details><summary>Solution</summary>

```csharp
public class CachedProductService(IMemoryCache cache, ProductRepository repo) {
    public async Task<Product?> GetAsync(int id, CancellationToken ct) =>
        await cache.GetOrCreateAsync($"product:{id}", async entry => {
            var product = await repo.GetAsync(id, ct);
            entry.AbsoluteExpirationRelativeToNow = product is null
                ? TimeSpan.FromSeconds(30)        // cache "not found" briefly (anti-penetration)
                : TimeSpan.FromMinutes(10);        // cache real results longer
            entry.Size = 1;                         // required if SizeLimit is set
            return product;
        });
}
// builder.Services.AddMemoryCache(o => o.SizeLimit = 10_000);   // bounded!
```

Negative results cached briefly (prevents repeated DB hits for missing ids), real results longer; `Size` set because the cache is bounded. ([03-IMemoryCache.md](03-IMemoryCache.md).)

</details>

---

## Problem 3: Invalidate cache on writes

Ensure updating a product evicts its cache entry.

<details><summary>Solution</summary>

```csharp
public async Task UpdateAsync(int id, UpdateProduct dto, CancellationToken ct) {
    await repo.UpdateAsync(id, dto, ct);
    cache.Remove($"product:{id}");          // evict so the next read re-fetches fresh data
}
```

Without eviction, the cache serves stale data until expiration. Evict the affected key(s) on every write. (With `HybridCache`, prefer tag-based invalidation for groups — Problem 6.) ([03-IMemoryCache.md](03-IMemoryCache.md).)

</details>

---

## Problem 4: Wire Redis distributed caching

Cache products in Redis (shared across instances) with manual serialization.

<details><summary>Solution</summary>

```csharp
builder.Services.AddStackExchangeRedisCache(o => {
    o.Configuration = builder.Configuration.GetConnectionString("Redis");
    o.InstanceName = "myapp:";
});

public class DistributedProductService(IDistributedCache cache, ProductRepository repo) {
    private static readonly JsonSerializerOptions Json = new(JsonSerializerDefaults.Web);

    public async Task<Product?> GetAsync(int id, CancellationToken ct) {
        string key = $"product:{id}";
        var bytes = await cache.GetAsync(key, ct);
        if (bytes is not null) return JsonSerializer.Deserialize<Product>(bytes, Json);

        var product = await repo.GetAsync(id, ct);
        if (product is not null)
            await cache.SetAsync(key, JsonSerializer.SerializeToUtf8Bytes(product, Json),
                new() { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10) }, ct);
        return product;
    }
}
```

Shared across instances; you serialize/deserialize `byte[]` (the cost vs in-process). ([04-IDistributedCache.md](04-IDistributedCache.md).)

</details>

---

## Problem 5: Graceful degradation when the cache is down

Make the distributed cache optional — a cache failure must not break the request.

<details><summary>Solution</summary>

```csharp
public async Task<Product?> GetAsync(int id, CancellationToken ct) {
    string key = $"product:{id}";
    try {
        var bytes = await cache.GetAsync(key, ct);
        if (bytes is not null) return JsonSerializer.Deserialize<Product>(bytes, Json);
    } catch (Exception ex) {
        logger.LogWarning(ex, "Cache read failed for {Key}; falling back to source", key);
        // swallow — treat a cache error as a miss
    }

    var product = await repo.GetAsync(id, ct);   // source of truth

    try {
        if (product is not null)
            await cache.SetAsync(key, JsonSerializer.SerializeToUtf8Bytes(product, Json), Options, ct);
    } catch (Exception ex) {
        logger.LogWarning(ex, "Cache write failed for {Key}", key);   // non-fatal
    }
    return product;
}
```

A Redis outage degrades to "always fetch from source" rather than failing requests. Pair with resilience (timeouts/circuit breaker — [Ch11](../11-Resilience/README.md)). ([04-IDistributedCache.md](04-IDistributedCache.md).)

</details>

---

## Problem 6: HybridCache with stampede protection and tag invalidation

Replace the manual L1+L2 + locking with `HybridCache`.

<details><summary>Solution</summary>

```csharp
builder.Services.AddStackExchangeRedisCache(o => o.Configuration = redisConnectionString);   // L2
builder.Services.AddHybridCache();                                                             // L1 + L2

public class ProductService(HybridCache cache, ProductRepository repo) {
    public async Task<Product?> GetAsync(int id, CancellationToken ct) =>
        await cache.GetOrCreateAsync(
            $"product:{id}",
            async token => await repo.GetAsync(id, token),     // ONE execution per key under contention
            new HybridCacheEntryOptions {
                Expiration = TimeSpan.FromMinutes(10),          // L2
                LocalCacheExpiration = TimeSpan.FromMinutes(2)  // L1 (shorter)
            },
            tags: ["products", $"product:{id}"], cancellationToken: ct);

    public async Task UpdateAsync(int id, UpdateProduct dto, CancellationToken ct) {
        await repo.UpdateAsync(id, dto, ct);
        await cache.RemoveByTagAsync("products", ct);           // invalidate across L1+L2 / all instances
    }
}
```

One call gives L1+L2, **stampede protection** (one factory run per key), built-in serialization, and **tag invalidation** across instances — replacing the manual machinery of Problems 2–5. ([05-HybridCache.md](05-HybridCache.md).)

</details>

---

## Problem 7: Demonstrate (and fix) a cache stampede

Show why `IMemoryCache.GetOrCreateAsync` stampedes and fix it.

<details><summary>Solution</summary>

```csharp
// ✗ — with IMemoryCache, 100 concurrent requests for an expired key run the factory ~100 times
var tasks = Enumerable.Range(0, 100).Select(_ =>
    memCache.GetOrCreateAsync("hot", async e => {
        Interlocked.Increment(ref _factoryRuns);     // observe: runs many times!
        await Task.Delay(500); return await repo.GetExpensiveAsync();
    }));
await Task.WhenAll(tasks);   // _factoryRuns ≈ 100 → 100 DB hits

// ✓ — HybridCache: factory runs ONCE; the other 99 await it
var tasks2 = Enumerable.Range(0, 100).Select(_ =>
    hybridCache.GetOrCreateAsync("hot", async t => {
        Interlocked.Increment(ref _factoryRuns);     // runs ONCE
        await Task.Delay(500); return await repo.GetExpensiveAsync();
    }, cancellationToken: ct).AsTask());
await Task.WhenAll(tasks2);   // _factoryRuns == 1 → 1 DB hit
```

`IMemoryCache` lets concurrent misses all run the factory (DB hammered). `HybridCache` coalesces them into one execution — built-in stampede protection. (Pre-.NET-9, you'd guard with a per-key `SemaphoreSlim`.) ([05-HybridCache.md](05-HybridCache.md), [03-IMemoryCache.md](03-IMemoryCache.md).)

</details>

---

## Problem 8: Choose the right cache layer

For each scenario, pick the cache and justify.
1. A `/products` API endpoint returning the same JSON for all anonymous users.
2. A `Category` lookup table read on every request, changed monthly.
3. A user's shopping cart, needed consistently across 3 load-balanced instances.
4. An expensive report fragment reused inside several different endpoints' responses.

<details><summary>Solution</summary>

1. **Output caching** ([06](06-OutputCaching.md)) — caching the whole response skips the handler entirely; vary-by nothing (or by query) since it's identical for anonymous users. Highest leverage.
2. **`IMemoryCache`** (or a `FrozenDictionary` loaded at startup) — small, hot, rarely-changing, per-instance is fine; cheap in-process reads. Invalidate/reload monthly or on change.
3. **`IDistributedCache` / `HybridCache`** — must be consistent across instances (a cart updated on instance A must be visible on B), so it needs a shared store. HybridCache gives L1 speed + L2 sharing.
4. **`HybridCache.GetOrCreateAsync`** — explicit data caching of the expensive fragment, reused inside different responses (so output caching of whole responses doesn't fit); stampede protection guards the expensive computation.

The principle: **output caching** for whole identical responses, **in-process** for hot per-instance data, **distributed/Hybrid** for cross-instance consistency, **explicit data caching** for expensive values reused inside varying responses. ([06](06-OutputCaching.md), [03](03-IMemoryCache.md), [04](04-IDistributedCache.md), [05](05-HybridCache.md).)

</details>

---

You can now build Dapper repositories, layer caching at the right level (output vs data; in-process vs distributed vs hybrid), invalidate correctly, prevent stampedes with `HybridCache`, and degrade gracefully when the cache is unavailable.

→ Back to [Chapter 06 README](README.md) · Next chapter: [Chapter 07 — Messaging](../07-Messaging/README.md)
