# 06 — Data Access & Caching

## ⚡ 30-second answer

Below EF Core sits **Dapper** (a micro-ORM: you write the SQL, it parameterizes and maps) and
**ADO.NET** (`DbConnection`/`DbCommand`/`DbDataReader`, which both build on). The common
production pattern is **EF Core for writes and the domain, Dapper for read-heavy or complex
queries**. The one ADO.NET fact that matters everywhere: **connections are pooled**, so `Open`
and `Dispose` rent and return — **open late, dispose early**, and never hold one across a slow
call. On caching there are three tiers: **`IMemoryCache`** (in-process, fastest, per-instance),
**`IDistributedCache`** (Redis — shared across instances, pays serialization plus a network
hop), and **`HybridCache`** (.NET 9+, L1+L2 behind one API with **stampede protection** and
**tag invalidation**) — which is the recommended default for new apps. Always **bound your
cache**, cache **immutable** data, and **invalidate on write**.

---

## Core mechanics

### Dapper

```csharp
var orders = await conn.QueryAsync<Order>(
    "SELECT * FROM Orders WHERE CustomerId = @id AND Status = @status",
    new { id, status = "Open" });                 // ← anonymous object → SQL parameters
```

Near-raw-ADO.NET speed, far less boilerplate, **no change tracking and no migrations**. The API
worth knowing: `Query`/`QuerySingle`/`QueryFirstOrDefault`, `Execute`, `ExecuteScalar`,
**`QueryMultiple`** (several result sets in one round-trip), and **multi-mapping** with `splitOn`
for joins. Pass a `CancellationToken` via `CommandDefinition`.

### ADO.NET

```csharp
await using var conn = await dataSource.OpenConnectionAsync(ct);   // DbDataSource, .NET 7+
await using var cmd = conn.CreateCommand();
cmd.CommandText = "UPDATE Orders SET Status = @s WHERE Id = @id";
cmd.Parameters.Add(new SqlParameter("@s", SqlDbType.VarChar, 20) { Value = status });
await cmd.ExecuteNonQueryAsync(ct);
```

`ExecuteReader` (rows), `ExecuteNonQuery` (affected count), `ExecuteScalar` (single value).
**Always parameterize** — never concatenate. Prefer explicit parameter types over `AddWithValue`
on hot paths, because inferred types cause implicit conversions that can defeat an index.

**Connection pooling** is the fact to state in an interview: `Open` rents from a pool, `Dispose`
returns it. The pool is small (100 by default for SQL Server), so holding connections open across
slow work exhausts it and every other request blocks waiting.

Reach for raw ADO.NET only for **bulk copy** (`SqlBulkCopy`, Postgres `COPY`), streaming large
values, or provider-specific features. Otherwise Dapper or EF Core.

### The three caches

```csharp
// HybridCache — the modern default
var product = await cache.GetOrCreateAsync(
    $"product:{id}",
    async ct => await db.Products.FindAsync([id], ct),
    tags: ["products"], cancellationToken: ct);

await cache.RemoveByTagAsync("products", ct);      // invalidate across L1 + L2, all instances
```

| | `IMemoryCache` | `IDistributedCache` | **`HybridCache`** (.NET 9+) |
|---|---|---|---|
| Location | in-process | out-of-process (Redis) | **L1 in-process + L2 distributed** |
| Shared across instances | ❌ | ✅ | ✅ (via L2) |
| Cost per hit | none | serialize + network | L1 hit: none |
| Stores | any object | **`byte[]`** | typed objects |
| Stampede protection | ❌ | ❌ | ✅ **built in** |
| Tag invalidation | ❌ | ❌ | ✅ |
| Verdict | hot per-instance data | cross-instance state | **default for new apps** |

**Expiration**: **absolute** (a fixed lifetime) and **sliding** (resets on access). Combine
sliding with an absolute cap, or a frequently-read entry never expires. **Change tokens** tie
expiry to a source of truth.

**`HybridCache` works without Redis** — you get L1 only, but still stampede protection. Adding a
distributed cache enables L2.

### Cache stampede

When a hot key expires under load, every concurrent request misses and hits the database at once —
a thundering herd that can take down the database that the cache was protecting.
`IMemoryCache.GetOrCreateAsync` **does not** prevent this: N concurrent misses run N factories.
`HybridCache` does — one factory per key, the rest await it.

### Output caching vs data caching

**Output caching** caches whole HTTP **responses** server-side, skipping the handler, the database
and serialization entirely — the broadest, highest-leverage cache for read-heavy endpoints
([04](04-AspNetCore.md)). **Data caching** (`IMemoryCache`/`HybridCache`) caches **values your
code stores** inside a handler. They layer, and both are server-controlled with tag invalidation
in modern .NET — unlike legacy **response caching**, which only asked clients and proxies to
cooperate.

### EF second-level caching

EF's **first-level cache** is the change tracker — it dedupes loads *within* one `DbContext`. A
**second-level cache** (a community interceptor) caches query results *across* contexts and
requests. It auto-invalidates by tracking which tables a query touched and evicting on
`SaveChanges` to those tables — but invalidation is **table-granular** and **misses out-of-band
writes** (`ExecuteUpdate`, raw SQL, another application). So it works only when EF is the sole
writer of read-mostly tables. **Prefer explicit caching** around specific queries.

---

## 🪤 Traps & gotchas

- **Unbounded cache** — the classic memory leak that looks like a feature. `IMemoryCache` has no
  limit unless you set `SizeLimit` *and* a per-entry `Size`. It grows until the process OOMs.
- **Sliding expiration with no absolute cap** — a key read once a second never expires, so it
  serves stale data forever.
- **Cache stampede** — a hot key expires and every in-flight request hits the database
  simultaneously. `IMemoryCache` won't save you; `HybridCache` will.
- **Caching mutable objects** — `IMemoryCache` hands every caller the **same reference**. One
  caller mutating the cached object corrupts it for everyone. Cache immutable types or clone.
- **Forgetting the tenant or user in the cache key** — one tenant's data served to another. The
  highest-severity caching bug ([22](22-BestPractices-Architecture.md)).
- **Caching without invalidating on write** — the write succeeds, the cache still serves the old
  value, and the user reports "my change didn't save."
- **Assuming `IMemoryCache` is shared across instances** — it isn't. Behind a load balancer, each
  instance has its own copy, so invalidation on one doesn't reach the others and users see
  different data depending on which pod they land on.
- **Caching cheap things in Redis** — a distributed cache hit costs serialization plus a network
  round-trip. If regenerating the value is cheaper than that, the cache makes you slower.
- **No graceful degradation when Redis is down** — the cache is an optimization; a cache outage
  should mean slow, not broken. Catch and fall through to the source.
- **`AddDistributedMemoryCache` in production** — despite the name it is **not distributed**; it's
  an in-process implementation of the interface for dev and tests.
- **Holding a database connection open across a slow call** (an HTTP request, a long computation)
  — the pool is small and you've just serialized your whole application on it.
- **`AddWithValue`** infers the parameter type from the CLR value, producing implicit conversions
  in the SQL that can prevent index use — a classic "the same query is fast in SSMS" mystery.
- **Concatenating SQL in Dapper or ADO.NET** — injection. Anonymous objects and `DbParameter`
  exist precisely to make the safe path the easy one.
- **Not disposing readers/commands/connections** — `await using` everywhere; a leaked connection is
  a pool slot gone until GC.
- **Second-level cache with out-of-band writes** — `ExecuteUpdate`, raw SQL, or another service
  writing the same table leaves the cache confidently stale, and nothing tells you.
- **Caching negative results forever** — a "not found" cached with a long TTL means creating the
  record doesn't help. Cache negatives, but briefly.

---

## ❓ Likely questions

**Q: When would you use Dapper instead of EF Core?**
A: Read-heavy, complex, or performance-critical queries where you want to write the SQL — reports,
aggregations, queries EF translates poorly. They coexist happily: EF for writes and the domain
model, Dapper for reads. EF wins for rich modeling, change tracking, and migrations.

**Q: What does Dapper not give you?**
A: Change tracking, migrations, a model, or LINQ translation. You write and own the SQL, which is
the point — it's a mapper, not an ORM.

**Q: Explain connection pooling and its practical consequence.**
A: `Open` rents a connection from a pool and `Dispose` returns it, because establishing a
connection (TCP plus auth) is expensive. The pool is bounded — 100 by default for SQL Server — so
the rule is open late and dispose early. Holding a connection across a slow operation exhausts the
pool and every other request blocks.

**Q: `IMemoryCache` vs `IDistributedCache`?**
A: `IMemoryCache` is in-process: fastest possible, no serialization, but each instance has its own
copy, so it's inconsistent behind a load balancer. `IDistributedCache` is shared (usually Redis),
so all instances see the same data and invalidation is global — at the cost of serialization and a
network round-trip, and it only stores `byte[]`.

**Q: What is `HybridCache` and why is it the new default?**
A: It layers an in-process L1 over a distributed L2 behind one `GetOrCreateAsync`, and adds the
three things the older APIs lack: **stampede protection** (one factory per key under concurrent
misses), **built-in typed serialization**, and **tag-based invalidation** across both tiers and
all instances.

**Q: What is a cache stampede and how do you prevent it?**
A: A hot key expires and every concurrent request misses at once, so they all hit the database
together — often taking down the thing the cache was protecting. `IMemoryCache.GetOrCreateAsync`
doesn't prevent it. `HybridCache` does, or you serialize the misses yourself with a per-key
`SemaphoreSlim`.

**Q: Absolute vs sliding expiration?**
A: Absolute expires at a fixed time after creation regardless of use. Sliding resets the clock on
each access, so idle entries expire and busy ones stay. Combine them — sliding for the eviction
behavior, with an absolute cap so a constantly-read entry can't serve stale data forever.

**Q: How do you keep a cache consistent with the database?**
A: Invalidate on write, in the same code path that does the write — ideally by tag so a group of
related entries goes at once. Use TTL as a backstop, not the primary mechanism. And accept that
across instances, `IMemoryCache` can't be invalidated remotely, which is a reason to use L2.

**Q: What should you not cache?**
A: Anything mutable that callers might modify in place, anything personalized without the user in
the key, anything cheap to recompute relative to the cache round-trip, and anything whose
staleness has real consequences (balances, permissions) unless you invalidate rigorously.

**Q: Output caching or data caching?**
A: Output caching stores the whole response and skips the handler entirely — the biggest win for
anonymous read-heavy endpoints. Data caching stores a value inside your logic and is what you use
when the response is personalized but an expensive sub-computation is shared. They layer.

**Q: What is EF's second-level cache and why be careful?**
A: An interceptor that caches query results across `DbContext` instances, invalidating by tracking
which tables a query touched. The care: invalidation is table-granular, and it can't see writes
made through `ExecuteUpdate`, raw SQL, or another application — so it goes stale silently. Prefer
explicit caching around specific queries.

**Q: When would you drop to raw ADO.NET?**
A: Bulk copy (`SqlBulkCopy`, Postgres `COPY`), streaming large values, or a provider-specific
feature neither EF nor Dapper exposes. Otherwise the ergonomics aren't worth it.

---

## 🎓 Senior Extra

- **`DbDataSource` (.NET 7+)** is the modern entry point — it owns the connection string and
  pooling configuration, is DI-friendly, and replaces passing raw connection strings around.
- **Cache key design is the whole game.** Include every input that changes the value: tenant,
  user (or explicitly not), culture, version. A version prefix (`v2:product:{id}`) gives you
  instant global invalidation on a schema change without touching Redis.
- **Two-tier caching has a coherence problem** the L2 doesn't solve: L1 copies on other instances
  are still stale after an invalidation. `HybridCache` uses backplane notifications; know that the
  window exists.
- **Cache the *shape* the caller needs**, not the entity. Caching a projected DTO avoids
  serialization of navigation properties and accidental lazy-loading on deserialization.
- **Negative caching with a short TTL** protects against a hot miss being used as an attack — a
  key that never exists otherwise reaches the database on every request.
- **Redis is single-threaded per shard**: one `KEYS *` or a huge value blocks every other client.
  Prefer `SCAN`, bound value sizes, and treat a shared Redis as a shared resource with noisy
  neighbours.
- **Serializer choice matters at L2** — System.Text.Json is fine, but MessagePack or protobuf cuts
  both payload size and CPU noticeably on a hot distributed cache.
- **Dapper plus EF in one transaction** works: get the connection from
  `db.Database.GetDbConnection()` and enlist with `db.Database.CurrentTransaction`. Useful when a
  write path needs one hand-tuned query.
- **The read-through pattern is a decision, not a default.** Write-through, write-behind, and
  cache-aside have different failure modes; cache-aside (`GetOrCreate` plus invalidate on write)
  is the one to default to because its failure mode is a stale read, not a lost write.
- **Measure the hit rate.** A cache below ~80% hit rate on a hot path is usually a key-design
  problem, and one at 100% with no invalidation is usually serving stale data nobody has noticed
  yet ([12](12-Observability.md)).

→ Deeper: [`../06-DataAndCaching/`](../06-DataAndCaching/README.md)
