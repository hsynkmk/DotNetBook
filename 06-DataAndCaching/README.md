# Chapter 06 — Data Access & Caching

> Beyond EF Core: lightweight ORMs, raw ADO.NET, and the caching layer that keeps your reads fast.

**Prerequisites**: Chapter 03 (DI), Chapter 05 (EF Core) for context.

**Time to read**: ~6-8 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-Dapper.md](01-Dapper.md) | The micro-ORM: `Query<T>`, `Execute`, parameterization, multi-mapping, dynamic. |
| [02-ADO.NET.md](02-ADO.NET.md) | `DbConnection`, `DbCommand`, `DbDataReader` — when you need raw SQL with no abstraction. |
| [03-IMemoryCache.md](03-IMemoryCache.md) | In-process cache, eviction, sliding/absolute expiration, size limits, `GetOrCreateAsync`. |
| [04-IDistributedCache.md](04-IDistributedCache.md) | Redis, SQL Server, in-memory — the distributed cache abstraction; serialization choices. |
| [05-HybridCache.md](05-HybridCache.md) | `HybridCache` (.NET 9+) — L1 + L2 cache combined, stampede protection. |
| [06-OutputCaching.md](06-OutputCaching.md) | ASP.NET Core output caching (briefly here, deep in chapter 04). |
| [07-EFCoreSecondLevel.md](07-EFCoreSecondLevel.md) | EFCoreSecondLevelCacheInterceptor — when to cache queries automatically. |
| [Questions.md](Questions.md) | Drilling questions. |
| [Coding.md](Coding.md) | Build a Dapper repo, wire Redis, prevent cache stampede. |

→ Begin: [01-Dapper.md](01-Dapper.md)
