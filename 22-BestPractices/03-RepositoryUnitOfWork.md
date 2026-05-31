# Repository and Unit of Work

## The pattern — and the EF Core nuance

The **Repository** pattern abstracts data access behind a collection-like interface (`IOrderRepository` with `Add`/`GetById`/`Remove`), and the **Unit of Work** pattern coordinates saving multiple changes as one atomic transaction. They're classic, valuable patterns — but with EF Core there's a crucial twist: **`DbContext` is already a Unit of Work, and `DbSet<T>` is already a repository** ([Ch05 §01](../05-EFCore/01-DbContext.md)). So the central question isn't "how do I implement these patterns?" but "do I need *another* abstraction on top of EF Core's, and if so, why?" Adding repositories reflexively over EF Core is one of the most common over-engineering mistakes.

```csharp
// DbContext IS a Unit of Work; DbSet IS a repository:
public class AppDbContext : DbContext {
    public DbSet<Order> Orders => Set<Order>();      // a repository of Orders
}
// Tracks changes across many entities, then commits them atomically:
order.Ship();
customer.Promote();
await db.SaveChangesAsync();   // ← Unit of Work: one transaction for all tracked changes
```

---

## What the patterns are

- **Repository** — mediates between the domain and data mapping, exposing a **collection-like** interface for an aggregate ([02-DomainPersistence.md](02-DomainPersistence.md)): `Add`, `GetById`, `Remove`, query methods. The domain depends on the *interface*; the implementation handles storage.
- **Unit of Work** — tracks changes made during a business operation and commits them as **one atomic transaction** (all-or-nothing), so partial updates don't corrupt state.

These let the domain be persistence-ignorant ([02-DomainPersistence.md](02-DomainPersistence.md)) and changes be transactional.

---

## EF Core already implements both

The key realization: EF Core's `DbContext` **is** a Unit of Work (it tracks changes across entities and commits them atomically on `SaveChanges`), and each `DbSet<T>` **is** a repository (a queryable collection you `Add`/`Remove`/`Find` on). So wrapping EF Core in a *generic* repository/UoW often **adds a layer that duplicates what EF Core already provides** — with downsides:

```csharp
// ✗ A generic repository over EF Core — usually redundant, and it leaks/limits EF Core:
public class Repository<T> where T : class {
    public Task<T?> GetByIdAsync(int id) => _db.Set<T>().FindAsync(id).AsTask();
    public IQueryable<T> Query() => _db.Set<T>();    // leaks IQueryable anyway — abstraction is thin
    // ...wraps Add/Remove/SaveChanges that DbContext already exposes
}
```

A generic repo tends to either **leak `IQueryable`** (so it's not really abstracting anything) or **hide EF Core's power** (Include, projections, AsNoTracking, split queries — [Ch05 §09](../05-EFCore/09-Performance.md)) behind a lowest-common-denominator interface that forces inefficient access. Either way, it's often ceremony without benefit.

---

## When a repository *is* worth it

Repositories aren't useless — they're valuable when they earn their keep:

- **A domain-meaningful interface for an aggregate** — not a generic `Repository<T>`, but `IOrderRepository` with **intention-revealing** methods (`GetPendingOrdersForCustomer`, `Add`) that express how the domain accesses *that* aggregate ([02-DomainPersistence.md](02-DomainPersistence.md)). This abstracts persistence behind the domain's language, encapsulates query complexity, and keeps the domain testable.
- **Decoupling the domain from EF Core** — if you want the domain/application to not reference EF Core at all (Clean architecture — [01-SolutionLayout.md](01-SolutionLayout.md)), a repository interface in the domain (implemented in infrastructure) provides that seam.
- **Encapsulating complex/repeated queries** — a place to put a non-trivial query so it's defined once and named.
- **Testability** — a focused repository interface is easy to fake in unit tests ([Ch17 §03](../17-Testing/03-Mocking.md)) (though integration tests against a real DB via Testcontainers are often better for data logic — [Ch17 §06](../17-Testing/06-TestContainers.md)).

The rule of thumb: a **specific, domain-oriented repository per aggregate** can be worthwhile; a **generic `Repository<T>` that just wraps `DbSet<T>`** usually isn't.

---

## When to skip it

For many apps — especially CRUD-style ([04-CQRS.md](04-CQRS.md)) — using `DbContext`/`DbSet` **directly** (or via CQRS query/command handlers that use EF Core directly) is simpler and more powerful than adding a repository layer. You keep EF Core's full querying capability, write less code, and avoid an abstraction that adds nothing. Modern guidance (including from the EF Core team) is that **EF Core already provides Repository/UoW**, so add another layer only for a concrete reason (domain decoupling, intention-revealing aggregate access), not by default.

---

## Transactions beyond one SaveChanges

`SaveChanges` is atomic for one call. When a business operation spans **multiple** `SaveChanges` or external systems, use an explicit **transaction** ([Ch05 §08](../05-EFCore/08-Transactions.md)) or, for cross-service consistency, the **outbox pattern** ([Ch07 §07](../07-Messaging/07-Patterns.md), [05-DomainEvents.md](05-DomainEvents.md)). Don't conflate "Unit of Work" with "I need a repository wrapper" — EF Core's transaction support covers the multi-step case directly.

---

## Common gotchas

### Generic `Repository<T>` over EF Core by default

It usually duplicates `DbSet<T>`, leaks `IQueryable` (no real abstraction), or hides EF Core's power (Include/projections/AsNoTracking). Add a repository only for a **specific, domain-oriented** reason — not reflexively.

### A UoW wrapper around DbContext

`DbContext` *is* the Unit of Work. Wrapping `SaveChanges` in an `IUnitOfWork.Commit()` adds indirection for no gain. Use `SaveChanges`/transactions directly.

### Repository methods that return `IQueryable`

If the repository returns `IQueryable`, callers compose queries on it — so it isn't abstracting persistence at all. Either fully encapsulate queries (return materialized results / domain objects) or skip the repository.

### Hiding performance features

A repository that only exposes `GetAll`/`GetById` forces loading whole entities, blocking projections, `AsNoTracking`, and split queries ([Ch05 §09](../05-EFCore/09-Performance.md)). Don't let the abstraction cause N+1/over-fetching.

### Over-abstracting a CRUD app

For simple CRUD, direct `DbContext`/`DbSet` (or CQRS handlers using EF Core) is simpler and better than a repository layer. Match the abstraction to the need.

---

## Summary

- **Repository** abstracts data access behind a collection-like interface; **Unit of Work** commits multiple changes atomically — but **EF Core's `DbContext` is already a Unit of Work and `DbSet<T>` is already a repository** ([Ch05 §01](../05-EFCore/01-DbContext.md)).
- A **generic `Repository<T>` over EF Core** is usually over-engineering — it duplicates `DbSet`, **leaks `IQueryable`** (no real abstraction), or **hides EF Core's power** (Include/projections/AsNoTracking — [Ch05 §09](../05-EFCore/09-Performance.md)).
- A repository **is** worth it as a **specific, domain-oriented interface per aggregate** (`IOrderRepository` with intention-revealing methods) — to decouple the domain from EF Core ([01-SolutionLayout.md](01-SolutionLayout.md)), encapsulate complex queries, and aid testing — not as a reflexive generic wrapper.
- For **CRUD/simple apps**, use `DbContext`/`DbSet` (or CQRS handlers) directly; for multi-step/cross-service atomicity use explicit **transactions** ([Ch05 §08](../05-EFCore/08-Transactions.md)) or the **outbox** ([Ch07 §07](../07-Messaging/07-Patterns.md)) — not a UoW wrapper.

→ Next: [04-CQRS.md](04-CQRS.md)
