# Querying

## LINQ to Entities → SQL

EF Core translates LINQ queries over `DbSet<T>` into SQL, executes them on the database, and materializes the results into entities. You write C#; EF generates SQL. The key skills: knowing **what translates** (and what silently doesn't), projecting efficiently, controlling tracking, and shaping joins.

```csharp
var results = await db.Products
    .Where(p => p.Price < 100 && p.Category == "Tools")    // → WHERE Price < 100 AND Category = 'Tools'
    .OrderBy(p => p.Name)                                   // → ORDER BY Name
    .Take(20)                                               // → LIMIT/TOP 20
    .ToListAsync();                                         // executes the query
```

A `DbSet<T>` is `IQueryable<T>`, so LINQ operators **build an expression tree** (not execute immediately — deferred, like LINQ-to-Objects but translated to SQL). Execution happens at a **terminal** operator (`ToListAsync`, `FirstAsync`, `CountAsync`, `AnyAsync`, etc.). (IQueryable vs IEnumerable: CSharpBook Ch06 §08.)

---

## Always use the async terminal operators

```csharp
await db.Products.ToListAsync(ct);
await db.Products.FirstOrDefaultAsync(p => p.Id == id, ct);
await db.Products.AnyAsync(p => p.Price > 1000, ct);
await db.Products.CountAsync(ct);
await db.Products.SumAsync(p => p.Price, ct);
await db.Products.FindAsync(id);   // by PK; checks the tracker first (no query if already loaded)
```

Use the **`*Async`** terminal operators (and pass the request `CancellationToken`) — they free the thread during the DB round-trip ([Ch01 §08](../01-Runtime/08-Threading.md)). Blocking (`.ToList()` on a DB query, or `.Result`) ties up a thread per query and risks pool starvation under load.

---

## What translates (and what doesn't)

EF translates what it can express in SQL; an operation it can't translate either throws or — historically a footgun — **silently evaluates on the client**:

```csharp
// ✓ translates to SQL
.Where(p => p.Price > 10 && p.Name.StartsWith("A"))
.Where(p => EF.Functions.Like(p.Name, "%phone%"))     // provider functions
.OrderBy(p => p.Created).Skip(20).Take(10)
.GroupBy(p => p.Category).Select(g => new { g.Key, Count = g.Count() })

// ✗ can't translate — calling YOUR C# method in a Where can't become SQL
.Where(p => MyCustomCheck(p))                          // throws: could not be translated
```

Modern EF Core (3.0+) **throws** when it can't translate a query, rather than silently fetching everything and filtering in memory (the old "accidental client evaluation" that killed performance). If you need client logic, **materialize first** (`ToListAsync`) then apply it in memory — deliberately, knowing you fetched the rows:

```csharp
var loaded = await db.Products.Where(p => p.Active).ToListAsync(ct);   // SQL filter
var filtered = loaded.Where(p => MyCustomCheck(p));                     // client filter (explicit)
```

---

## Projection — select only what you need

Fetching whole entities when you need three columns wastes bandwidth and tracking overhead. **Project to a DTO/anonymous type** so EF selects only those columns:

```csharp
// ✗ — fetches all columns + tracks entities
var list = await db.Products.ToListAsync();

// ✓ — SELECT only Id, Name, Price; no tracking needed (projections aren't tracked)
var summaries = await db.Products
    .Select(p => new ProductSummary(p.Id, p.Name, p.Price))
    .ToListAsync(ct);
```

Projections to non-entity types are **not change-tracked** (there's nothing to track), so they're inherently faster and lighter. For read-only data (the common case in APIs/queries), **project to DTOs** — it's the biggest easy query win.

---

## Tracking vs no-tracking

By default, queries returning entities are **tracked** — EF keeps a snapshot so it can detect changes for `SaveChanges` ([03-ChangeTracking.md](03-ChangeTracking.md)). For **read-only** queries you don't intend to update, that's wasted work:

```csharp
// Read-only → AsNoTracking (faster, less memory; no change-tracking snapshots)
var products = await db.Products.AsNoTracking().Where(p => p.Active).ToListAsync(ct);

// Set it as the default for a context that's read-mostly:
o.UseNpgsql(cs).UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
```

Rule: **`AsNoTracking` for reads you won't modify** (lists, reports, API responses); tracking only when you'll change and `SaveChanges` the entities. Projections to DTOs are already untracked, so the no-tracking concern mainly applies to queries returning full entities.

---

## Loading related data: Include, projection, lazy

To pull in related entities (navigations), choose deliberately:

```csharp
// Eager loading — Include (one query with JOINs, or split queries)
var orders = await db.Orders
    .Include(o => o.Customer)                       // JOIN Customer
    .Include(o => o.Items).ThenInclude(i => i.Product)   // nested includes
    .ToListAsync(ct);

// Projection — often better: select exactly the shape you need (no over-fetch)
var dtos = await db.Orders.Select(o => new OrderDto(
    o.Id, o.Customer.Name, o.Items.Count, o.Items.Sum(i => i.Price))).ToListAsync(ct);

// Explicit loading — load a navigation on demand
await db.Entry(order).Collection(o => o.Items).LoadAsync(ct);
```

- **Eager (`Include`)** — load related data up front in the same round-trip. Prevents N+1 (below).
- **Projection** — often the best: fetch only the fields you need, shaped as a DTO (no `Include` needed for read scenarios).
- **Lazy loading** — navigations load automatically when accessed (requires proxies + virtual navigations). **Convenient but dangerous**: it triggers a query per access, the classic **N+1** generator. Generally avoid lazy loading in EF Core; prefer `Include` or projection.

---

## The N+1 problem (the cardinal query bug)

```csharp
// ✗ — N+1: one query for orders, then ONE query per order for its items (lazy or in a loop)
var orders = await db.Orders.ToListAsync(ct);
foreach (var o in orders)
    Console.WriteLine(o.Items.Count);   // each access → a separate SQL query (with lazy loading)

// ✓ — one (or few) queries
var orders = await db.Orders.Include(o => o.Items).ToListAsync(ct);   // eager
// or project the count directly:
var counts = await db.Orders.Select(o => new { o.Id, ItemCount = o.Items.Count }).ToListAsync(ct);
```

N+1 — 1 query for the parents + N queries for each parent's children — is the most common EF performance killer. It hides behind lazy loading and loops. Fix with `Include` (eager) or projection. Detect it with EF's logging or `dotnet-trace` ([09-Performance.md](09-Performance.md)).

---

## Single vs split queries

`Include`-ing multiple **collection** navigations in one query causes a **cartesian explosion** (rows multiply across the joins). EF can instead run **split queries** — one SQL query per collection:

```csharp
var orders = await db.Orders
    .Include(o => o.Items)
    .Include(o => o.Payments)
    .AsSplitQuery()                  // separate query per collection (avoids row explosion)
    .ToListAsync(ct);
```

- **Single query** (default) — one round-trip, but multiple collection includes multiply rows (e.g., 10 orders × 5 items × 3 payments = 150 rows of duplicated order data).
- **Split query** (`AsSplitQuery`) — multiple round-trips, no duplication. Better when including several collections.

Trade-off: single query = fewer round-trips but possible data explosion; split = more round-trips but less data. Use split when including multiple collections; single for one collection or scalar navigations. (Set a global default with `UseQuerySplittingBehavior`.)

---

## Filtering, paging, aggregation

```csharp
// Paging — always order before Skip/Take (else order is undefined)
var page = await db.Products.OrderBy(p => p.Id).Skip(pageSize * (pageNum - 1)).Take(pageSize).ToListAsync(ct);

// Keyset pagination (better for large offsets — no slow OFFSET)
var page = await db.Products.Where(p => p.Id > lastSeenId).OrderBy(p => p.Id).Take(pageSize).ToListAsync(ct);

// Aggregation translates to SQL GROUP BY
var byCategory = await db.Products
    .GroupBy(p => p.Category)
    .Select(g => new { Category = g.Key, Count = g.Count(), Avg = g.Average(p => p.Price) })
    .ToListAsync(ct);
```

For deep paging, prefer **keyset pagination** (`WHERE id > lastSeen`) over `Skip`/`Take` (`OFFSET`), which gets slower as the offset grows. Always `OrderBy` before paging (SQL ordering isn't guaranteed otherwise).

---

## Common gotchas

### N+1 from lazy loading / loops

The #1 EF perf bug. Avoid lazy loading; use `Include` or projection; detect via query logging.

### Fetching whole entities for read-only data

Tracking + all columns is wasteful. Project to DTOs (untracked, fewer columns) for reads.

### Forgetting `AsNoTracking` for reads

Read queries returning entities still build tracking snapshots. Use `AsNoTracking` (or a no-tracking default) for read-only entity queries.

### Cartesian explosion with multiple collection Includes

Row counts multiply. Use `AsSplitQuery` when including several collections.

### Client evaluation surprises

Calling a non-translatable C# method in `Where`/`Select` throws (modern EF). Materialize first (`ToListAsync`) then apply client logic deliberately.

### Blocking on queries

`.ToList()`/`.Result` on a DB query blocks a thread. Use `*Async` + `CancellationToken`.

### Paging without ordering

`Skip`/`Take` without `OrderBy` returns arbitrary rows. Always order; prefer keyset pagination for large offsets.

---

## Summary

- LINQ over `DbSet<T>` (`IQueryable`) builds an expression tree **translated to SQL**, executed at an **async terminal** operator (`ToListAsync`, etc. — pass `CancellationToken`).
- Modern EF **throws** on untranslatable queries (no silent client evaluation); materialize first if you need client logic.
- **Project to DTOs** for reads (selects only needed columns, untracked) and use **`AsNoTracking`** for read-only entity queries — the biggest easy wins.
- Load related data with **`Include`** (eager) or **projection**; **avoid lazy loading** (it causes **N+1**, the cardinal perf bug — fix with Include/projection).
- Use **`AsSplitQuery`** when including multiple collections (avoid cartesian explosion); always **`OrderBy` before paging**, and prefer **keyset pagination** for deep offsets.

→ Next: [03-ChangeTracking.md](03-ChangeTracking.md)
