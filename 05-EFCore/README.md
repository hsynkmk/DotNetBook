# Chapter 05 — Entity Framework Core

> .NET's flagship ORM. Maps C# objects to relational databases, translates LINQ to SQL, tracks changes, applies migrations. EF Core 10 is the version current with .NET 10.

**Prerequisites**: CSharpBook Chapter 06 (LINQ). SQL basics.

**Time to read**: ~12-16 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-DbContext.md](01-DbContext.md) | The unit-of-work: `DbContext`, `DbSet<T>`, connection management, dispose timing, pooling. |
| [02-Querying.md](02-Querying.md) | LINQ-to-Entities, what translates, projection, `AsNoTracking`, `AsSplitQuery`, single-vs-split. |
| [03-ChangeTracking.md](03-ChangeTracking.md) | The change tracker, entity states, `SaveChanges`, identity resolution. |
| [04-Relationships.md](04-Relationships.md) | 1:1, 1:N, M:N, fluent vs convention vs data annotations, principal/dependent. |
| [05-Migrations.md](05-Migrations.md) | `dotnet ef migrations add/update/script`, conflicts, production deployment, idempotent SQL. |
| [06-RawSQL.md](06-RawSQL.md) | `FromSql`, `ExecuteSql`, `ExecuteUpdate`/`ExecuteDelete` (EF 7+), interpolation safety. |
| [07-Concurrency.md](07-Concurrency.md) | Optimistic concurrency, RowVersion, conflict resolution. |
| [08-Transactions.md](08-Transactions.md) | Implicit vs explicit, IDbContextTransaction, distributed (when avoidable). |
| [09-Performance.md](09-Performance.md) | N+1 detection, compiled queries, query splitting, `AsNoTracking`, hot paths. |
| [10-DbContextFactory.md](10-DbContextFactory.md) | When to use `IDbContextFactory<T>` (parallel queries, Blazor, console workers). |
| [11-Interceptors.md](11-Interceptors.md) | `IDbCommandInterceptor`, `ISaveChangesInterceptor` — auditing, soft delete. |
| [12-OwnedTypes.md](12-OwnedTypes.md) | Owned types, value objects, JSON columns. |
| [13-GlobalQueryFilters.md](13-GlobalQueryFilters.md) | Multi-tenancy, soft delete, default filters. |
| [14-Testing.md](14-Testing.md) | InMemory provider (and its traps), SQLite in-memory, TestContainers for real DBs. |
| [Questions.md](Questions.md) | ~30 drilling questions. |
| [Coding.md](Coding.md) | Build a DbContext, write migrations, optimize N+1, implement soft delete. |

---

## Learning objectives

After this chapter you should be able to:
- Model a domain with EF Core; configure relationships via fluent API.
- Write performant queries; avoid N+1; pick include vs split queries appropriately.
- Apply migrations safely in production.
- Build interceptors for cross-cutting concerns.
- Use the right tooling for testing data-layer code.

→ Begin: [01-DbContext.md](01-DbContext.md)
