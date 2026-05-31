# HybridCache

## The best of both, with stampede protection (.NET 9+)

`HybridCache` is the modern caching API that combines an in-process **L1** cache (fast, like `IMemoryCache`) with an optional distributed **L2** cache (shared, like `IDistributedCache`) behind one interface — and adds the things both raw caches lack: **stampede protection**, built-in **serialization**, and **tag-based invalidation**. For most new apps, `HybridCache` is the recommended caching abstraction.

```csharp
builder.Services.AddHybridCache();              // L1 only by default
// Add a distributed L2 (Redis) and HybridCache layers L1 in front of it:
builder.Services.AddStackExchangeRedisCache(o => o.Configuration = redisConnectionString);
builder.Services.AddHybridCache();

public class ProductService(HybridCache cache, IProductRepository repo) {
    public async Task<Product?> GetAsync(int id, CancellationToken ct) =>
        await cache.GetOrCreateAsync(
            $"product:{id}",
            async token => await repo.GetAsync(id, token),       // factory: runs on miss
            cancellationToken: ct);
}
```

One `GetOrCreateAsync` call: checks L1, then L2, then runs the factory on a full miss, **populating both layers** — and ensures the factory runs **once** even under concurrent misses.

---

## L1 + L2: how it works

```
Request → check L1 (in-process, fastest)
            ↓ miss
          check L2 (distributed, shared — if configured)
            ↓ miss
          run factory (DB/compute) → store in L2 → store in L1 → return
```

- **L1 (in-process)** — fastest, per-instance; serves repeat requests on the same server with no serialization/network.
- **L2 (distributed)** — shared across instances; serves requests that missed L1 (e.g., a different server, or after L1 eviction) without re-running the factory.

You get in-process speed for hot data **and** cross-instance sharing — without manually coordinating two caches. L2 is optional: with just `AddHybridCache` you get L1 + stampede protection; add a distributed cache to get L2 too.

---

## Stampede protection (the headline feature)

The problem with `IMemoryCache`/`IDistributedCache`: when a hot key expires, **many concurrent requests all miss and run the expensive factory at once** (a "stampede" / "cache stampede" / "thundering herd") — hammering the database exactly when it's least able to cope.

`HybridCache` prevents this: for a given key, **only one factory execution runs**; concurrent callers for the same key **await that single execution** and share its result.

```csharp
// 1000 simultaneous requests for an expired key:
//   IMemoryCache.GetOrCreateAsync → up to 1000 factory runs (1000 DB hits!)
//   HybridCache.GetOrCreateAsync  → ONE factory run; the other 999 await it
var product = await cache.GetOrCreateAsync($"product:{id}", async t => await repo.GetAsync(id, t), ct);
```

This alone is a reason to prefer `HybridCache` — stampede protection is hard to get right by hand (per-key locks/semaphores) and is built in here. It protects your database from the thundering-herd effect on popular cache keys.

---

## Built-in serialization

Unlike `IDistributedCache` (where you serialize `byte[]` yourself), `HybridCache` **handles serialization** for the L2 layer automatically — you cache and retrieve typed objects:

```csharp
// Typed in, typed out — no manual JsonSerializer calls
Product? p = await cache.GetOrCreateAsync(key, factory, ct);
```

It uses a configurable serializer (System.Text.Json by default; pluggable for MessagePack/protobuf). This removes the serialize/deserialize boilerplate that the raw distributed cache requires.

---

## Tag-based invalidation

`HybridCache` supports **tags**, so you can evict groups of related entries at once (e.g., all of a product's cache entries when it changes):

```csharp
await cache.GetOrCreateAsync($"product:{id}", factory,
    tags: ["products", $"product:{id}"], cancellationToken: ct);

// Invalidate everything tagged "products" on a write:
await cache.RemoveByTagAsync("products", ct);
await cache.RemoveAsync($"product:{id}", ct);            // or a single key
```

Tag invalidation propagates across L1 and L2, so a write invalidates the entry on **all instances** — solving the "per-instance cache goes stale after a write on another server" problem that plagues raw `IMemoryCache` in multi-instance deployments.

---

## Per-entry options

```csharp
await cache.GetOrCreateAsync(key, factory, new HybridCacheEntryOptions {
    Expiration = TimeSpan.FromMinutes(30),            // L2 (distributed) lifetime
    LocalCacheExpiration = TimeSpan.FromMinutes(5),    // L1 (in-process) lifetime (often shorter)
    Flags = HybridCacheEntryFlags.DisableLocalCache,   // e.g., skip L1 for very large items
}, tags: [...], ct);
```

You can set separate lifetimes for L1 and L2 (L1 often shorter, since it's per-instance and you may want fresher local data), and flags to skip a layer for specific entries. Global defaults are configured in `AddHybridCache`.

---

## HybridCache vs the raw caches

| | IMemoryCache | IDistributedCache | **HybridCache** |
|---|---|---|---|
| In-process (L1) | ✓ | ✗ | ✓ |
| Distributed (L2) | ✗ | ✓ | ✓ (optional) |
| Stampede protection | ✗ | ✗ | **✓** |
| Built-in serialization | n/a | ✗ (manual) | **✓** |
| Tag invalidation | ✗ | ✗ | **✓** |
| Cross-instance invalidation | ✗ | per-key | **✓ (by tag)** |

For new code, **`HybridCache` is the recommended default** — it subsumes both raw caches and adds stampede protection, serialization, and tag invalidation. Use raw `IMemoryCache`/`IDistributedCache` directly only for simple cases or when you specifically don't want the extra machinery.

---

## When to use HybridCache

- **Any read-heavy caching** in a modern (.NET 9+) app — it's the recommended default.
- **Multi-instance apps** needing in-process speed *and* cross-instance sharing/invalidation.
- **Hot keys** vulnerable to stampede (popular products, trending content).
- When you'd otherwise hand-roll L1+L2 + per-key locking — `HybridCache` does it for you.

When raw caches suffice: trivial per-instance caching with no stampede risk (just `IMemoryCache`), or you're on a pre-.NET-9 runtime.

---

## Common gotchas

### Expecting it pre-.NET 9

`HybridCache` requires .NET 9+ (the `Microsoft.Extensions.Caching.Hybrid` package). On older runtimes, layer `IMemoryCache` + `IDistributedCache` manually (and add your own stampede protection).

### Forgetting to add an L2

`AddHybridCache` alone gives L1 + stampede protection but **no** distributed layer — add a distributed cache (Redis) for cross-instance sharing. Without L2, it's L1-only (per-instance), though still stampede-protected.

### Caching mutable objects

L1 returns the cached instance (no serialization round-trip), so mutating it affects other callers — cache **immutable** data, as with `IMemoryCache`.

### Over-long L1 expiration in multi-instance apps

A long L1 lifetime means a write (even with tag invalidation) may not immediately refresh other instances' L1 until the invalidation propagates — keep L1 expiration modest for data that changes.

### Not tagging for invalidation

Without tags you can only remove by exact key. Tag entries you'll want to bulk-invalidate (e.g., `"products"`), so a write can clear related entries across instances.

---

## Summary

- **`HybridCache`** (.NET 9+) combines in-process **L1** + distributed **L2** behind one `GetOrCreateAsync`, and is the **recommended caching default** for new apps — it subsumes both raw caches.
- It adds what they lack: **stampede protection** (only one factory runs per key under concurrent misses — protecting the DB from thundering herds), **built-in serialization** (typed in/out, no manual `byte[]`), and **tag-based invalidation** (evict groups across L1 + L2 / all instances).
- Configure separate **L1/L2 expirations** and per-entry flags; add a distributed cache (Redis) to enable L2 (without it, L1-only but still stampede-protected).
- Use it for read-heavy and multi-instance caching, especially hot keys; cache **immutable** data; tag entries for bulk/cross-instance invalidation.

→ Next: [06-OutputCaching.md](06-OutputCaching.md)
