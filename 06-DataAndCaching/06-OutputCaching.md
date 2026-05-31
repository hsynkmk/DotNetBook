# Output Caching (in the caching landscape)

## Where output caching fits

Output caching caches whole **HTTP responses** server-side, so a cached endpoint serves its stored response without re-running the handler. It's covered in depth in **[Chapter 04 §15](../04-AspNetCore/15-OutputCaching.md)** (vary-by rules, tag invalidation, policies); this file places it in the **caching landscape** alongside `IMemoryCache`, `IDistributedCache`, and `HybridCache` so you choose the right layer.

```csharp
builder.Services.AddOutputCache();
var app = builder.Build();
app.UseOutputCache();
app.MapGet("/products", (IProductSvc s) => s.All())
    .CacheOutput(p => p.Tag("products").Expire(TimeSpan.FromMinutes(5)));
```

---

## The caching layers — what caches *what*

| Layer | Caches | Where | Key |
|---|---|---|---|
| **Output caching** | full HTTP **responses** | server (in-mem or distributed) | request (path + vary-by) |
| **`IMemoryCache`** | arbitrary **objects/data** | in-process | your string key |
| **`IDistributedCache`** | arbitrary **byte[]** | out-of-process (Redis) | your string key |
| **`HybridCache`** | arbitrary **objects** (L1+L2) | in-process + distributed | your string key |
| **Response caching** (old) | responses via HTTP headers | client/proxy | request |
| **EF second-level** | **query results** | configurable | the query |

The key distinction: **output caching caches the *response*** (the whole serialized HTTP result for a request), while the data caches (`IMemoryCache`/`HybridCache`/`IDistributedCache`) cache **data/objects** that *your code* chooses to store and retrieve. They operate at different levels and are complementary.

---

## Output caching vs data caching — choosing the level

```
Caching an entire endpoint's response (same output for many requests)?
   → Output caching (skips the handler entirely — biggest win for read-heavy endpoints)

Caching a value/computation/query result used INSIDE your logic?
   → HybridCache (or IMemoryCache / IDistributedCache)
```

- **Output caching** is the broadest, highest-leverage cache for a public read endpoint whose response is identical (or varies by a few dimensions) across requests — it short-circuits routing, the handler, the database, and serialization. Cache at this level when you can.
- **Data caching** (`HybridCache` et al.) is finer-grained — cache a specific expensive value (a product, a config, a computed report fragment) that's reused *within* handlers, possibly assembled into different responses. Cache here when responses differ but share expensive sub-computations.

They layer: an endpoint might be output-cached *and* its handler use `HybridCache` for the data it assembles (relevant on output-cache misses).

---

## Server-controlled, with invalidation

Modern **output caching** (.NET 7+) is server-controlled and supports **tag-based invalidation** — unlike the older **response caching**, which only emitted HTTP cache headers and relied on the client/proxy to honor them (no server-side eviction):

```csharp
// Tag responses, evict on writes — keeps cached responses fresh
app.MapPost("/products", async (Product p, IProductSvc s, IOutputCacheStore cache, CancellationToken ct) => {
    s.Create(p);
    await cache.EvictByTagAsync("products", ct);   // invalidate cached product responses
    return Results.Created($"/products/{p.Id}", p);
});
```

This invalidation capability (shared with `HybridCache`'s tags) is what makes server-side caching practical — you keep cached data fresh on writes rather than waiting for TTL. Prefer **output caching** over the legacy **response caching** for response-level caching.

---

## Distributed output cache

Like the data caches, in-memory output caching is **per-instance**. For multi-instance consistency (a write on one server invalidates cached responses on all), back output caching with a distributed store:

```csharp
builder.Services.AddStackExchangeRedisOutputCache(o => o.Configuration = redisConnectionString);
```

This shares cached responses and tag invalidation across instances — the same per-instance-vs-distributed consideration that applies to `IMemoryCache` vs `IDistributedCache`/`HybridCache`.

---

## The cardinal rules (recap from Ch04)

Two rules carry over and are worth repeating because they're the dangerous ones:
- **Vary-by correctly** — cache keyed by everything the response depends on (query/route/header/user). Wrong vary-by serves the wrong content; **caching personalized responses without per-user keys leaks one user's data to another**.
- **Don't cache writes or volatile/personalized data** without care; cache GETs for stable-ish data, invalidate by tag on writes.

Full detail — policies, vary-by, what (not) to cache — is in [Chapter 04 §15](../04-AspNetCore/15-OutputCaching.md).

---

## Common gotchas

### Confusing output caching with data caching

Output caching caches **responses**; `IMemoryCache`/`HybridCache` cache **data** your code stores. Use output caching for whole-response caching, data caches for values reused inside logic. (Both can apply.)

### Using legacy response caching expecting server control

Old `ResponseCaching` only sets HTTP headers (client/proxy decides) and can't invalidate. Use **output caching** (server-controlled, tag invalidation) instead.

### Per-instance output cache assumed shared

In-memory output cache isn't shared across instances; a write invalidating on one leaves others stale. Use the **Redis output cache** for multi-instance consistency.

### Wrong vary-by / caching personalized responses

The cardinal output-cache bug — serves wrong/leaked content. Vary by everything the response depends on; don't cache personalized responses without per-user keys ([Ch04 §15](../04-AspNetCore/15-OutputCaching.md)).

---

## Summary

- **Output caching** caches whole HTTP **responses** server-side (skips handler/DB/serialization) — the broadest, highest-leverage cache for read-heavy endpoints; **deep coverage in [Chapter 04 §15](../04-AspNetCore/15-OutputCaching.md)**.
- It's distinct from **data caching** (`IMemoryCache`/`IDistributedCache`/`HybridCache`), which caches **values/objects your code stores** — use output caching for whole responses, data caches for expensive values reused inside handlers; they layer.
- Modern output caching is **server-controlled with tag invalidation** (prefer it over legacy header-based **response caching**); back it with **Redis** for multi-instance consistency.
- Recap the dangerous rules: **vary-by correctly** and **don't cache personalized responses without per-user keys** (data-leak risk).

→ Next: [07-EFCoreSecondLevel.md](07-EFCoreSecondLevel.md)
