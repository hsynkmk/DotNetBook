# Performance

## Where EF Core apps get slow

EF Core makes data access easy — and easy to make slow. The common culprits: N+1 queries, fetching too much, tracking when you don't need it, chatty round-trips, and over-materializing. Most EF performance wins come from a handful of patterns. As always: **measure first** ([Ch21](../21-Performance/README.md)) — log the SQL and find the actual hot queries before optimizing.

---

## #1: Eliminate N+1 queries

The dominant EF performance bug (introduced in [02-Querying.md](02-Querying.md)): one query for parents, then a query **per parent** for children.

```csharp
// ✗ — N+1: lazy loading or a loop fires a query per order
var orders = await db.Orders.ToListAsync(ct);
foreach (var o in orders) total += o.Items.Sum(i => i.Price);   // each o.Items → a query!

// ✓ — eager load in one round-trip
var orders = await db.Orders.Include(o => o.Items).ToListAsync(ct);

// ✓✓ — project the aggregate; the DB computes it, minimal data transferred
var totals = await db.Orders.Select(o => o.Items.Sum(i => i.Price)).ToListAsync(ct);
```

Detect N+1 by **logging SQL** (you'll see the same query shape repeated) or with `dotnet-trace`/EF's diagnostics. Fix with `Include` (eager) or projection. **Disable lazy loading** so it can't sneak in.

---

## #2: Project to DTOs, don't fetch whole entities

For reads, select only the columns you need into a DTO/anonymous type — less data over the wire, no tracking overhead, and the DB can sometimes satisfy the query from an index alone:

```csharp
// ✗ — SELECT * + tracking, for data you'll only read
var products = await db.Products.ToListAsync(ct);

// ✓ — SELECT only 3 columns, untracked (projections aren't tracked)
var summaries = await db.Products
    .Select(p => new ProductSummary(p.Id, p.Name, p.Price))
    .ToListAsync(ct);
```

Projection is the **single biggest easy win** for read-heavy code. Combined with paging, it keeps payloads small.

---

## #3: `AsNoTracking` for reads

Queries returning entities build change-tracking snapshots ([03-ChangeTracking.md](03-ChangeTracking.md)). For read-only queries that's wasted CPU and memory:

```csharp
var products = await db.Products.AsNoTracking().Where(p => p.Active).ToListAsync(ct);
// Or default a read-mostly context to no-tracking:
o.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
```

`AsNoTracking` skips snapshots and identity resolution — faster and lighter for reads you won't modify. (DTO projections are already untracked, so this mainly helps queries returning full entities.)

---

## #4: Bulk operations without loading

To update/delete many rows, don't load them into memory — use **`ExecuteUpdate`/`ExecuteDelete`** (EF 7+), which run a single SQL statement:

```csharp
// ✗ — load 100k rows, modify, save (huge memory + round-trips)
foreach (var p in await db.Products.Where(p => p.Discontinued).ToListAsync(ct)) p.IsActive = false;
await db.SaveChangesAsync(ct);

// ✓ — one UPDATE statement, no loading, no tracking
await db.Products.Where(p => p.Discontinued)
    .ExecuteUpdateAsync(s => s.SetProperty(p => p.IsActive, false), ct);
```

For bulk **inserts**, `AddRange` + a single `SaveChanges` batches them; for very large inserts, a dedicated bulk-copy library (or `COPY` on PostgreSQL) beats EF.

---

## #5: Query splitting for multiple collections

Including several **collection** navigations in one query causes cartesian explosion (rows multiply). Use `AsSplitQuery` ([02-Querying.md](02-Querying.md)):

```csharp
var orders = await db.Orders
    .Include(o => o.Items).Include(o => o.Payments)
    .AsSplitQuery()                 // separate query per collection — avoids row explosion
    .ToListAsync(ct);
```

Trade-off: split = more round-trips but no duplicated data. Use it when including multiple collections; single query for one collection.

---

## #6: Compiled queries (hot paths)

Each query EF executes is compiled from its expression tree to SQL — cached, but the cache lookup and tree processing have a cost. For a **very hot** query run millions of times, a **compiled query** pre-compiles it once:

```csharp
private static readonly Func<AppDbContext, int, Task<Product?>> GetById =
    EF.CompileAsyncQuery((AppDbContext db, int id) => db.Products.FirstOrDefault(p => p.Id == id));

var product = await GetById(db, 42);   // skips query-compilation overhead
```

`EF.CompileQuery`/`CompileAsyncQuery` eliminates the per-call compilation/cache-lookup cost. A micro-optimization — measure before reaching for it; for most apps the built-in query cache is enough.

---

## #7: Reduce round-trips & startup cost

```csharp
// Batch reads with one round-trip where possible (project related data, or use Include)
// Fewer SaveChanges calls = fewer transactions/round-trips (batch related changes)

// Context pooling reduces per-request setup (Ch05 §01)
builder.Services.AddDbContextPool<AppDbContext>(o => o.UseNpgsql(cs));

// Compiled model — faster startup for LARGE models (many entities)
dotnet ef dbcontext optimize    // generates a compiled model
```

- **Fewer round-trips**: each query/`SaveChanges` is a DB round-trip (latency!). Batch reads (Include/projection), batch writes (one `SaveChanges`).
- **Context pooling** (`AddDbContextPool`) cuts per-request context setup in high-throughput apps.
- **Compiled model** (`dbcontext optimize`) speeds startup for large models.

---

## Diagnosing EF performance

```csharp
// Log generated SQL (dev) — the #1 diagnostic for N+1 and bad queries
o.UseNpgsql(cs).LogTo(Console.WriteLine, LogLevel.Information).EnableSensitiveDataLogging();   // dev only!
```

- **Log the SQL** — see exactly what EF generates; spot N+1 (repeated query shapes), missing indexes (slow scans), and over-fetching. `EnableSensitiveDataLogging` shows parameter values (**dev only** — it logs data).
- **`dotnet-trace`/`dotnet-counters`** — EF emits events; correlate with overall perf ([Ch21](../21-Performance/README.md)).
- **Database tooling** — `EXPLAIN`/query plans reveal missing indexes (an EF problem is often a *database* problem — no index on a filtered column).
- **Watch the round-trip count** — many small queries (latency-bound) often beats one giant join only if you're not N+1-ing.

---

## Indexes (it's often a DB problem)

EF performance frequently comes down to **database indexes**, not EF itself. Declare indexes in the model so migrations create them:

```csharp
b.Entity<Product>().HasIndex(p => p.Sku).IsUnique();
b.Entity<Order>().HasIndex(o => new { o.CustomerId, o.Created });   // composite, for common queries
```

A query filtering/sorting on an unindexed column does a full table scan. Add indexes for columns you frequently filter, join, or sort on — then verify with the query plan. No EF tuning fixes a missing index.

---

## Common gotchas

### N+1 from lazy loading / loops

The dominant EF perf bug. Disable lazy loading; use `Include`/projection; detect via SQL logging.

### Fetching whole entities for reads

Tracking + all columns. Project to DTOs and/or use `AsNoTracking`.

### Loading rows to bulk-modify

Wasteful. Use `ExecuteUpdate`/`ExecuteDelete`.

### Cartesian explosion

Multiple collection `Include`s multiply rows. Use `AsSplitQuery`.

### Missing indexes

The most common "EF is slow" cause is actually a missing DB index. Check query plans; add `HasIndex`.

### Optimizing without measuring

Reaching for compiled queries/pooling before logging the SQL. Measure first — usually the fix is an index, a projection, or killing an N+1, not a micro-optimization.

### Sensitive data logging in production

`EnableSensitiveDataLogging` logs parameter values (PII, secrets). Dev only.

---

## Summary

- **Measure first** (log the SQL); most EF wins are a few patterns, and many "EF is slow" issues are actually **missing database indexes** (`HasIndex`).
- Kill **N+1** (Include/projection, disable lazy loading), **project to DTOs** for reads, and use **`AsNoTracking`** for read-only entity queries — the biggest easy wins.
- Use **`ExecuteUpdate`/`ExecuteDelete`** for bulk modifications (no loading), **`AsSplitQuery`** for multiple collection includes, and batch writes into **one `SaveChanges`**.
- Reduce **round-trips** (latency-bound), use **context pooling** for high throughput and a **compiled model** for large-model startup; **compiled queries** only for proven-hot paths.
- Diagnose by logging generated SQL (`EnableSensitiveDataLogging` dev-only), `dotnet-trace`, and database query plans.

→ Next: [10-DbContextFactory.md](10-DbContextFactory.md)
