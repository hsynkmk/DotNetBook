# 05 — Entity Framework Core

## ⚡ 30-second answer

EF Core is .NET's ORM: it maps entities to tables and translates **LINQ → SQL**. The
**`DbContext` is a Unit of Work plus a set of repositories (`DbSet<T>`)** — you query, mutate
tracked entities, and `SaveChanges()` diffs them against their snapshots and commits every
INSERT/UPDATE/DELETE **in one transaction**. It is **not thread-safe** and is registered
**scoped** — one per request ([03](03-Hosting-DI-Config.md)). The dominant performance bug is
**N+1** (a query per row from lazy loading); the fix is `Include` or, better, **projecting to a
DTO**. Use **`AsNoTracking`** for read-only queries, a **rowversion concurrency token** to stop
silent lost updates, and **migrations** to evolve the schema — applied in production as a
reviewed idempotent script, never auto-migrated on startup.

---

## Core mechanics

### The context is a unit of work

```csharp
builder.Services.AddDbContext<AppDb>(o => o.UseSqlServer(cs));   // scoped by default

var order = await db.Orders.Include(o => o.Lines)
                          .FirstAsync(o => o.Id == id, ct);
order.Status = OrderStatus.Shipped;        // just mutate — no Update() call needed
await db.SaveChangesAsync(ct);             // one transaction: UPDATE …
```

The DI scope disposes it — **don't dispose an injected context**. Because it is *already* a UoW
+ repository, wrapping it in a generic `Repository<T>` is usually redundant
([22](22-BestPractices-Architecture.md)).

### Query pipeline

LINQ over `DbSet<T>` builds an **expression tree** (`IQueryable`), translated to SQL and
executed at an **async terminal** (`ToListAsync`, `FirstAsync`, `AnyAsync`, `CountAsync`) —
always pass the `CancellationToken`. Modern EF (3.0+) **throws** on anything it can't translate
rather than silently evaluating client-side; materialize with `ToListAsync()` first if you
genuinely need client logic.

```csharp
// ✅ projection — only the needed columns, untracked, one query
var rows = await db.Orders
    .Where(o => o.CustomerId == id)
    .OrderByDescending(o => o.Created).ThenBy(o => o.Id)   // stable order BEFORE paging
    .Select(o => new OrderDto(o.Id, o.Total, o.Customer.Name))
    .Skip(page * size).Take(size)
    .ToListAsync(ct);
```

### Change tracking

Tracked entities get a **snapshot**; `SaveChanges` runs **`DetectChanges`**, diffs current
values against it, and emits minimal SQL per **state**:

```csharp
db.Products.Add(p);                          // Added     → INSERT
var e = await db.Products.FindAsync(id);     // Unchanged → nothing
e.Name = "New";                              // Modified  → UPDATE (only changed columns)
db.Products.Remove(e);                       // Deleted   → DELETE
```

**Identity resolution**: within one context, one instance per key — query the same row twice and
you get the *same* object back. `AsNoTracking` skips both tracking and identity resolution
(`AsNoTrackingWithIdentityResolution()` if you want de-duplication without tracking).

For **disconnected** entities (deserialized from a request body), prefer **load-then-mutate**
over `Update`, so only real changes are written and concurrency tokens still apply.

### Bulk without loading (EF 7+)

```csharp
await db.Products.Where(p => p.Category == "Obsolete")
    .ExecuteUpdateAsync(s => s.SetProperty(p => p.IsActive, false), ct);
await db.Orders.Where(o => o.Created < cutoff).ExecuteDeleteAsync(ct);
```

One SQL statement, nothing loaded or tracked. **Caveat**: they execute immediately and **bypass
the change tracker, `SaveChanges`, and its interceptors** — so auditing/soft-delete interceptors
don't run, and already-tracked entities go stale.

### Relationships

FKs + navigations. **One-to-many**: the dependent holds the FK. **One-to-one**: name the
dependent via `HasForeignKey<T>`. **Many-to-many**: implicit join table (EF 5+), or an explicit
join entity when the relationship carries data (e.g. `Quantity`).

```csharp
modelBuilder.Entity<Order>()
    .HasOne(o => o.Customer).WithMany(c => c.Orders)
    .HasForeignKey(o => o.CustomerId)
    .OnDelete(DeleteBehavior.Restrict);   // required rels default to Cascade — set deliberately
```

Prefer the **fluent API** for non-trivial config (keeps entities free of persistence attributes);
convention is fine for simple cases. NRT nullability of the FK/navigation defines required vs
optional — keep nullable reference types on.

### Migrations

Versioned code: `Up`/`Down` + a **model snapshot** + the `__EFMigrationsHistory` table.

```bash
dotnet ef migrations add AddOrders      # then REVIEW the generated file
dotnet ef migrations script --idempotent -o migrate.sql   # what production runs
```

Commit the model change, the migration, and the snapshot **together**. In production apply a
reviewed **idempotent script** or **migration bundle** as a single controlled step *before* the
new app serves traffic. Make migrations **backward-compatible** with the running version; split
breaking changes across deploys with **expand/contract** (add column → write both → backfill →
drop old).

### Concurrency and transactions

```csharp
public byte[] RowVersion { get; set; } = default!;   // [Timestamp] / .IsRowVersion()
```

The token goes into the UPDATE's `WHERE`; if another transaction changed the row, 0 rows match
and EF throws `DbUpdateConcurrencyException`. Resolve deliberately — **store wins**, **client
wins**, or **merge** — and cap retries. In a web app, round-trip the token to the client (hidden
field, or an HTTP **ETag** + `If-Match` → 409/412), because the entity was loaded in an earlier
request.

Every `SaveChanges` is already a transaction. Use an explicit one only when a unit of work spans
multiple saves or mixes raw SQL — and if `EnableRetryOnFailure` is on, wrap it in an execution
strategy so a retry re-runs the *whole* transaction:

```csharp
var strategy = db.Database.CreateExecutionStrategy();
await strategy.ExecuteAsync(async () =>
{
    await using var tx = await db.Database.BeginTransactionAsync(ct);
    // … multiple SaveChanges / raw SQL …
    await tx.CommitAsync(ct);
});
```

### Raw SQL, safely

`FromSql` (entities, composable), `SqlQuery<T>` (arbitrary shapes), `ExecuteSql` (non-query).
The **interpolated** overloads take a `FormattableString` and parameterize the holes — safe.
The **`…Raw`** overloads take a plain `string`: interpolating into those concatenates *before*
EF sees it → **SQL injection**. Pass explicit parameters instead.

### Cross-cutting: filters, interceptors, owned types

- **Global query filters** (`HasQueryFilter`) add a `WHERE` to *every* query for an entity —
  so you can't forget it. The two big uses: **soft delete** (`!IsDeleted`) and **multi-tenancy**
  (`TenantId == current`). Bypass with `IgnoreQueryFilters()` — carefully; EF 10 adds **named
  filters** so you can ignore soft-delete while keeping tenancy.
- **`SaveChangesInterceptor`** — `SavingChanges` runs before the write with change-tracker
  access: **auditing** (stamp `CreatedAt`/`UpdatedAt`) and **soft delete** (flip
  `Deleted` → `Modified` + `IsDeleted = true`). Reusable across contexts, unlike overriding
  `SaveChanges`. **Transparent soft delete needs both halves** — filter for reads, interceptor
  for writes.
- **Owned types** (`OwnsOne`/`OwnsMany`) are EF's **value object** mechanism: a component with
  no identity of its own (`Address`, `Money`), mapped into the owner's table, loaded with the
  owner (no `Include`, no N+1). `ToJson()` (EF 7+) maps one to a single **JSON column**.

### `IDbContextFactory<T>`

`AddDbContextFactory` gives you fresh, short-lived contexts **you own and dispose**. Use it for
**parallel queries** (a context per task), **Blazor Server** (components outlive a request), and
**singletons/background services** (avoiding the captive dependency). Choose
`IServiceScopeFactory` instead when you need *other* scoped services too.

---

## Comparison tables

| Loading strategy | What it does | When |
|---|---|---|
| **Eager** — `Include` | loads navigations with the query | you know you need the graph |
| **Lazy** — proxies | loads on first access | convenient, **N+1 risk** — avoid |
| **Explicit** — `Entry().LoadAsync` | loads on demand, manually | selective, conditional |
| **Projection** — `Select` DTO | fetches only needed columns | **read APIs — best perf** |

| | Tracked (default) | `AsNoTracking()` |
|---|---|---|
| Snapshot taken | yes | no |
| Identity resolution | yes | no |
| `SaveChanges` sees changes | yes | **no** |
| Use for | you will modify + save | read-only queries |

| Entity state | `SaveChanges` emits |
|---|---|
| `Unchanged` | nothing |
| `Added` | INSERT |
| `Modified` | UPDATE (changed columns only) |
| `Deleted` | DELETE |
| `Detached` | nothing |

| Disconnected save | Tracks as | Writes |
|---|---|---|
| `Attach` | `Unchanged` | nothing until you mark props `IsModified` |
| `Update` | `Modified` (**all** props) | **every column** — can clobber |
| **load-then-mutate** | `Unchanged` → `Modified` | only what really changed ✅ |

| Test approach | Fidelity | Verdict |
|---|---|---|
| Mock `DbContext`/`DbSet` | none | ❌ tests the mock, not LINQ-to-Entities |
| **InMemory** provider | low | ❌ not relational — no constraints, no translation |
| **SQLite in-memory** | medium | ✅ the sensible **default** (keep the connection open) |
| **Testcontainers** | high | ✅ query-heavy, DB-specific, migration tests |

| Bulk approach | Round-trips | Runs interceptors |
|---|---|---|
| load → mutate → `SaveChanges` | N + 1 | yes |
| `ExecuteUpdate`/`ExecuteDelete` | **1** | **no** |

---

## 🪤 Traps & gotchas

- **N+1 queries** — the cardinal EF bug. A navigation touched inside a loop fires one query per
  row. Fix with `Include` or projection; detect by logging the generated SQL (repeated shapes).

  ```csharp
  var orders = await db.Orders.ToListAsync(ct);
  foreach (var o in orders)
      Console.WriteLine(o.Customer.Name);   // 🐛 1 + N queries
  // ✅ db.Orders.Select(o => new { o.Id, Customer = o.Customer.Name })
  ```

- **Cartesian explosion** — two collection `Include`s in one query multiply the rows
  (10 lines × 10 payments = 100 rows). Use **`AsSplitQuery()`**: more round-trips, no duplicated
  payload.
- **`DbContext` is not thread-safe.** Running queries in parallel on one context throws
  *"A second operation was started on this context"*. One context per task, via
  `IDbContextFactory`.

  ```csharp
  await Task.WhenAll(ids.Select(i => db.Orders.FindAsync(i).AsTask()));   // 🐛 races
  ```

- **Captive dependency** — injecting the scoped `DbContext` into a **singleton** pins one
  context for the app's lifetime: it never releases tracked entities and is shared across
  threads. Use `IDbContextFactory` or `IServiceScopeFactory` ([03](03-Hosting-DI-Config.md)).
- **`Update()` overwrites unchanged columns** — it marks *every* property modified, so the
  UPDATE writes all columns and can clobber fields another user changed. Load-then-mutate.
- **No concurrency token = silent lost updates.** Two users load, both save, the second wins and
  the first user's edit vanishes with no error. Add a rowversion.
- **Catching `DbUpdateConcurrencyException` and retrying blindly** is just "client wins" with
  extra steps — decide the policy explicitly.
- **Auto-migrate on startup** (`Database.Migrate()`) across multiple instances races, and
  couples deployment to schema change. Idempotent script as a deploy step.
- **A property rename becomes `DropColumn` + `AddColumn`** — silent data loss. Review every
  generated migration and hand-edit to `RenameColumn`.
- **Never edit an already-applied migration** — add a new one. And always commit the model
  snapshot, or the next migration diffs against the wrong baseline.
- **Cascade delete by default**: required relationships default to `DeleteBehavior.Cascade`, so
  deleting a customer deletes all their orders. Set `OnDelete` deliberately.
- **Interpolating into `FromSqlRaw`** — injection. The compiler picks the `string` overload and
  concatenation happens before EF ever sees it.

  ```csharp
  db.Users.FromSqlRaw($"SELECT * FROM Users WHERE Name = '{name}'");  // 🐛 injectable
  db.Users.FromSql($"SELECT * FROM Users WHERE Name = {name}");       // ✅ parameterized
  ```

- **`ExecuteUpdate`/`ExecuteDelete`/`ExecuteSql` bypass the tracker** — interceptors (auditing,
  soft delete) don't run, and tracked entities are now stale.
- **Soft delete with only half the mechanism**: the query filter alone doesn't soft-delete (rows
  are really deleted); the interceptor alone leaves deleted rows visible to every query.
- **`IgnoreQueryFilters()` in a multi-tenant app** drops the *tenant* filter too — a cross-tenant
  data leak. Prefer EF 10 named filters.
- **Forgetting filters apply to `Include`s** — a filtered navigation silently returns fewer
  children than the raw table has.
- **Paging without `OrderBy`** — SQL guarantees no order without one, so rows repeat or vanish
  across pages. For deep offsets prefer **keyset pagination** over `Skip`.
- **`DetectChanges` is O(entities × properties)** — thousands of tracked entities make
  `SaveChanges` crawl. `AsNoTracking` for reads, short contexts.
- **Multiple `SaveChanges` for one logical operation** = multiple transactions = partial state on
  failure. Batch the changes, save once.
- **Manual transaction + `EnableRetryOnFailure`** throws unless you go through
  `CreateExecutionStrategy()`.
- **`EnableSensitiveDataLogging` in production** writes parameter values — including PII and
  credentials — into your logs. Dev only.
- **SQLite in-memory dies when the connection closes** — hold the connection open for the test's
  lifetime or you get an empty database.
- **"EF is slow"** is very often a **missing index**, not EF. Check the query plan before
  rewriting anything.

---

## ❓ Likely questions

**Q: What design patterns does `DbContext` implement?**
A: **Unit of Work** (tracks all changes, commits atomically in one transaction on
`SaveChanges`) and its `DbSet<T>`s act as **Repositories**. That's why a generic
`Repository<T>`/`IUnitOfWork` wrapper over EF is usually redundant — add a custom repository
only for domain-specific aggregate access.

**Q: What is the N+1 problem and how do you fix it?**
A: One query for the parents plus one more per parent to load its children — usually from lazy
loading or a navigation touched in a loop. Fix with `Include` (one query, eager) or by
projecting to a DTO so the join happens in SQL.

**Q: When do you use `AsNoTracking`?**
A: Read-only queries. It skips the change-tracking snapshot and identity resolution — less
memory and CPU. DTO projections are already untracked, so it's redundant there.

**Q: Is `DbContext` thread-safe? What lifetime should it have?**
A: Not thread-safe. Register it **scoped** — one per request. For parallel work, Blazor Server,
or background services, use `IDbContextFactory<T>` and dispose what you create.

**Q: How does optimistic concurrency work in EF Core?**
A: A concurrency token (ideally a DB-managed rowversion) is added to the UPDATE/DELETE `WHERE`
clause. If another transaction changed the row, zero rows match and EF throws
`DbUpdateConcurrencyException` — so you resolve the conflict instead of silently overwriting.
No locks are held.

**Q: Single vs split query?**
A: Including multiple **collection** navigations in one query causes a cartesian explosion.
`AsSplitQuery()` issues one query per collection — more round-trips, no row multiplication. One
collection: single query is fine.

**Q: `Attach` vs `Update` vs load-then-mutate?**
A: `Attach` tracks as `Unchanged` (nothing written until you mark properties modified). `Update`
tracks as `Modified` with *all* properties modified, so it writes every column. Load-then-mutate
is the safe default: minimal UPDATE, and concurrency checks still apply.

**Q: How should migrations be applied in production?**
A: Not via `Database.Migrate()` at startup — multiple instances race and deploy gets coupled to
schema change. Generate a reviewed **idempotent SQL script** or a **migration bundle** and run it
as a single controlled step before the new version takes traffic.

**Q: How do you make a schema change zero-downtime?**
A: Keep it backward-compatible with the currently-running code. Additive changes are safe;
breaking ones use **expand/contract** — add the new column, deploy code writing both, backfill,
then drop the old column in a later deploy.

**Q: How do you avoid SQL injection with raw SQL in EF?**
A: Use `FromSql`/`ExecuteSql` with an interpolated string — EF parameterizes the holes. The
`…Raw` variants take a plain string, so interpolating into them concatenates first and is
injectable; pass explicit parameters there.

**Q: Is `SaveChanges` transactional?**
A: Yes — all its statements run in one transaction, all-or-nothing. So make every related change
and call it **once**. You only need an explicit transaction when a unit of work spans multiple
saves or mixes in raw SQL.

**Q: How do you do a bulk update efficiently?**
A: `ExecuteUpdate`/`ExecuteDelete` (EF 7+) — set-based SQL, nothing loaded or tracked, one
round-trip. The trade-off is that they bypass `SaveChanges` and its interceptors.

**Q: What are global query filters for?**
A: A predicate EF adds to every query for an entity, so it can't be forgotten. Soft delete
(`!IsDeleted`) and multi-tenancy (`TenantId == current`) — the latter makes accidental
cross-tenant reads structurally impossible.

**Q: Owned type or related entity?**
A: Owned when it's a **value object** with no identity or lifecycle of its own (`Address`,
`Money`) — mapped into the owner's table, no `DbSet`, loaded with the owner. A related entity
when it has its own key, lifecycle, and independent queries.

**Q: Why not test against the InMemory provider?**
A: It isn't a relational database — no SQL translation, no FK/unique/not-null constraints, no
real transactions. It passes tests that fail in production. Microsoft advises against it; use
SQLite in-memory or Testcontainers ([17](17-Testing.md)).

**Q: Why shouldn't you mock `DbContext`/`DbSet`?**
A: `IQueryable` over a mock runs LINQ-to-**Objects**, so it never exercises translation, differs
on async, and you end up asserting your own setup. Mock a repository interface, or test the data
layer against a real provider.

---

## 🎓 Senior Extra

- **Compiled queries** (`EF.CompileAsyncQuery`) cache the LINQ→SQL translation for proven-hot
  paths; a **compiled model** (`dotnet ef dbcontext optimize`) cuts startup time for large
  models. Both are targeted fixes — measure first.
- **`AddDbContextPool`** reuses context instances to cut allocation at high request rates. The
  catch: pooled contexts are **reset, not recreated**, so any custom state you stashed in a field
  leaks between requests.
- **Round-trips dominate, not CPU.** Most EF latency is network to the database; batching writes
  into one `SaveChanges` and avoiding split-query fan-out usually beats micro-optimizing LINQ.
- **`ISaveChangesInterceptor` is the right seam** for auditing, soft delete, and **outbox
  writes** — enqueuing the message in the same transaction as the state change gives you
  atomicity without a distributed transaction ([07](07-Messaging.md)).
- **Avoid distributed transactions** (2PC across DB + broker): slow, poorly supported, and
  unnecessary. Outbox or saga instead ([22](22-BestPractices-Architecture.md)).
- **Prefer concurrency tokens to higher isolation levels.** Raising isolation holds locks and
  costs throughput to solve a *cross-request* problem that a rowversion solves lock-free.
- **`EnableRetryOnFailure`** handles transient cloud-DB faults, but silently changes transaction
  semantics — hence the execution-strategy requirement ([11](11-Resilience.md)).
- **Query tags** (`.TagWith("…")`) stamp a comment into the generated SQL, so a slow query in the
  database's own monitoring traces straight back to the C# that issued it.
- **Value converters + owned types** let the domain keep strongly-typed IDs and value objects
  while the schema stays clean — no EF attributes in the domain, all mapping in
  `IEntityTypeConfiguration<T>` ([22](22-BestPractices-Architecture.md)).
- **Multi-tenancy caveat**: a tenant filter reads the tenant from the *scoped* context. Capture
  it at construction, not through a service that might outlive the scope, or you will filter by
  the wrong tenant.
- **The repository debate**: since `DbContext` is already UoW + repository, the honest reasons to
  add one are enforcing aggregate boundaries and decoupling the domain from EF — not
  "testability", which SQLite/Testcontainers cover better.

→ Deeper: [`../05-EFCore/`](../05-EFCore/README.md)
