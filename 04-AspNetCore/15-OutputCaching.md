# Output Caching

## Caching whole responses

Output caching stores the **rendered response** of an endpoint and serves it from the cache for subsequent matching requests — skipping the handler, database, and serialization entirely. For read-heavy endpoints with data that changes infrequently, it's the single biggest performance/scale win. ASP.NET Core has built-in **output caching** middleware (.NET 7+).

```csharp
builder.Services.AddOutputCache();
var app = builder.Build();
app.UseOutputCache();                          // add the middleware

app.MapGet("/products", (IProductSvc svc) => svc.All())
    .CacheOutput();                            // cache this endpoint's response

app.MapGet("/products/{id:int}", (int id, IProductSvc svc) => svc.Get(id))
    .CacheOutput(p => p.Expire(TimeSpan.FromMinutes(5)).SetVaryByRouteValue("id"));
app.Run();
```

The first request runs the handler and caches the response; subsequent requests within the cache window get the stored response without invoking the handler.

---

## Output caching vs response caching vs distributed cache

| Mechanism | What it caches | Where | Control |
|---|---|---|---|
| **Output caching** (.NET 7+) | full responses | server (in-memory or distributed) | **server decides** (recommended) |
| **Response caching** (older) | full responses | client/proxy via HTTP headers | client/proxy honors `Cache-Control` |
| **`IDistributedCache`/`IMemoryCache`** | arbitrary data you choose | server | you cache *data*, not responses ([Ch06](../06-DataAndCaching/README.md)) |

**Output caching** is the modern, server-controlled response cache — the server fully decides what/when to cache and can **invalidate** entries (response caching couldn't). Use output caching for response-level caching; use `IMemoryCache`/`IDistributedCache` ([Ch06](../06-DataAndCaching/README.md)) for caching *data/computations* inside your logic. This file is about output (response) caching.

---

## Cache policies & expiration

```csharp
builder.Services.AddOutputCache(options => {
    options.AddBasePolicy(b => b.Expire(TimeSpan.FromSeconds(30)));      // default for all cached endpoints
    options.AddPolicy("long", b => b.Expire(TimeSpan.FromHours(1)));
    options.AddPolicy("byQuery", b => b.SetVaryByQuery("page", "size").Expire(TimeSpan.FromMinutes(5)));
});

app.MapGet("/feed", ...).CacheOutput("long");
app.MapGet("/list", ...).CacheOutput("byQuery");
```

Define named policies (expiration, vary-by rules, tags) and apply by name. `Expire(...)` sets the time-to-live. Without a vary-by rule, **all requests to the endpoint share one cached entry** — which is wrong if the response depends on query/route/headers (below).

---

## Vary-by — caching the right variations

A cached response must be keyed by everything the response depends on, or you'll serve the wrong content to someone:

```csharp
app.MapGet("/products", ...).CacheOutput(p => p
    .SetVaryByQuery("category", "page")       // separate cache entry per category+page
    .SetVaryByHeader("Accept-Language")        // per language
    .SetVaryByRouteValue("tenantId")
    .VaryByValue(ctx => ("region", GetRegion(ctx))));   // custom key contributor
```

- **`SetVaryByQuery`** — different `?page=`/`?category=` → different cached entries.
- **`SetVaryByHeader`** — e.g., per `Accept-Language` (don't serve English to a French client).
- **`SetVaryByRouteValue`** — per route value (e.g., per-tenant).
- **`VaryByValue`** — any custom key.

Getting vary-by wrong is the cardinal output-caching bug: too few keys → wrong content served; too many → low hit rate. Vary by exactly what changes the response.

---

## Cache invalidation by tag

When underlying data changes, you must **evict** stale cached responses. Output caching supports **tag-based invalidation**:

```csharp
app.MapGet("/products", ...).CacheOutput(p => p.Tag("products"));
app.MapGet("/products/{id:int}", ...).CacheOutput(p => p.Tag("products"));

// When a product changes, evict everything tagged "products"
app.MapPost("/products", async (Product p, IProductSvc svc, IOutputCacheStore cache, CancellationToken ct) => {
    svc.Create(p);
    await cache.EvictByTagAsync("products", ct);   // invalidate cached product responses
    return Results.Created($"/products/{p.Id}", p);
});
```

Tag cached endpoints, then `EvictByTagAsync` on writes to drop stale entries. This solves the hardest caching problem ("the two hard things in CS are cache invalidation and naming"): a write immediately invalidates affected reads, so clients don't see stale data. Choose a TTL as a backstop *and* tag-evict on changes for freshness.

---

## What (not) to cache

Cache:
- **GET** responses for data that's read far more than written (catalogs, config, public content).
- Expensive-to-compute or expensive-to-fetch responses.
- Responses identical across many users (or correctly varied per the dimensions that differ).

Don't cache (by default, the middleware already skips most of these):
- **Non-GET/HEAD** methods (writes).
- **Authenticated/personalized** responses — unless you vary by user (risk: serving one user's data to another!).
- Responses with **`Set-Cookie`** or that depend on per-request auth state.
- Rapidly-changing data where staleness is unacceptable (use a very short TTL or don't cache).

The biggest risk: **caching personalized responses without varying by user** → leaking one user's data to another. Be very careful caching anything behind auth.

---

## Distributed output cache

In-memory output caching is **per-instance** (each server has its own cache). For consistency across a multi-instance deployment, back it with a distributed store (Redis):

```csharp
builder.Services.AddStackExchangeRedisOutputCache(o => o.Configuration = redisConnectionString);
```

This shares cached responses (and tag invalidation) across instances — important so a write on one instance evicts entries on all, and so the cache hit rate isn't fragmented. For single-instance or where per-instance caching is acceptable, in-memory is simpler. (Distributed caching: [Ch06](../06-DataAndCaching/README.md).)

---

## Common gotchas

### Wrong/missing vary-by → serving wrong content

The #1 bug. If the response depends on query/route/header/user, vary by it — otherwise everyone gets the first cached response. Especially dangerous for personalized data.

### Caching authenticated responses without per-user keys

Leaks one user's data to another. Don't cache personalized responses unless you vary by user (and even then, be cautious).

### No invalidation → stale data

A TTL alone means stale reads until expiry. Tag responses and `EvictByTagAsync` on writes for immediate freshness.

### Per-instance cache assumed shared

In-memory output cache isn't shared across instances; a write evicting on one instance leaves others stale. Use a distributed (Redis) output cache for multi-instance consistency.

### Caching writes or rapidly-changing data

Caching POST/PUT or volatile data causes incorrect behavior. Cache GETs for stable-ish data; use short TTLs for semi-volatile.

### Confusing output caching with `IMemoryCache`

Output caching caches **responses** (server-controlled); `IMemoryCache`/`IDistributedCache` cache **data** you choose inside logic. Use the right one ([Ch06](../06-DataAndCaching/README.md)).

---

## Summary

- **Output caching** (.NET 7+: `AddOutputCache` + `UseOutputCache` + `CacheOutput()`) stores full responses server-side and serves them without re-running the handler — the biggest win for read-heavy endpoints.
- It's **server-controlled** (unlike old response caching) and supports **invalidation**; use it for response caching, and `IMemoryCache`/`IDistributedCache` for caching *data* ([Ch06](../06-DataAndCaching/README.md)).
- Define **policies** (expiration, vary-by, tags); **vary by exactly what changes the response** (query/route/header/user) — wrong vary-by serves wrong content, the cardinal bug.
- **Tag** cached endpoints and **`EvictByTagAsync`** on writes for freshness (TTL as backstop).
- **Don't cache** writes or personalized responses without per-user keys (data-leak risk); use a **distributed (Redis)** output cache for multi-instance consistency.

→ Next: [16-HealthChecks.md](16-HealthChecks.md)
