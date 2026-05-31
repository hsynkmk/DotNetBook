# Raw SQL

## When LINQ isn't enough

Sometimes you need SQL EF can't generate: a complex query, a stored procedure, a database-specific feature, or a bulk operation. EF Core lets you drop to raw SQL while staying integrated with the context — and, critically, while staying **safe from SQL injection** if you use the parameterized APIs.

```csharp
// Query returning entities
var products = await db.Products
    .FromSql($"SELECT * FROM Products WHERE Category = {category}")   // parameterized!
    .ToListAsync(ct);

// Non-query (UPDATE/DELETE/DDL)
await db.Database.ExecuteSqlAsync($"UPDATE Products SET Price = Price * 1.1 WHERE Category = {category}", ct);
```

---

## `FromSql` — raw SQL queries returning entities

```csharp
// FromSql with an interpolated string → parameters are auto-extracted (SAFE)
var results = await db.Products
    .FromSql($"SELECT * FROM Products WHERE Price > {minPrice} AND Category = {category}")
    .Where(p => p.IsActive)        // you can COMPOSE LINQ on top (becomes a subquery)
    .OrderBy(p => p.Name)
    .ToListAsync(ct);
```

`FromSql` (and `FromSqlInterpolated`) runs raw SQL but returns tracked entities and lets you **compose** further LINQ on top (EF wraps your SQL as a subquery). Requirements:
- The SQL must return columns matching the entity's mapped properties.
- You can compose `Where`/`OrderBy`/`Include`/etc. afterward.

For querying arbitrary shapes (not full entities), use `SqlQuery<T>`:

```csharp
// Query scalar/DTO results not mapped to an entity
var totals = await db.Database
    .SqlQuery<decimal>($"SELECT SUM(Price) FROM Orders WHERE CustomerId = {customerId}")
    .ToListAsync(ct);
```

---

## `ExecuteSql` — non-query commands

```csharp
int rows = await db.Database.ExecuteSqlAsync(
    $"UPDATE Products SET IsActive = false WHERE LastSold < {cutoff}", ct);

await db.Database.ExecuteSqlAsync($"CALL refresh_materialized_view()", ct);   // stored proc / DDL
```

`ExecuteSql`/`ExecuteSqlAsync` run UPDATE/DELETE/INSERT/DDL/stored procedures that don't return entities, returning the affected-row count. These execute **immediately** (not staged for `SaveChanges`) and **bypass the change tracker** — the context's tracked entities won't reflect the change unless you reload.

---

## `ExecuteUpdate` / `ExecuteDelete` — typed bulk operations (EF 7+)

For bulk modifications, prefer the **strongly-typed** `ExecuteUpdate`/`ExecuteDelete` (LINQ-based, no raw SQL strings) over hand-written SQL — they're injection-safe by construction and refactor-friendly:

```csharp
await db.Products.Where(p => p.Category == "Obsolete")
    .ExecuteUpdateAsync(s => s
        .SetProperty(p => p.IsActive, false)
        .SetProperty(p => p.Price, p => p.Price * 0.5m), ct);

await db.Orders.Where(o => o.Created < cutoff).ExecuteDeleteAsync(ct);
```

These generate a single SQL UPDATE/DELETE without loading entities (covered in [03-ChangeTracking.md](03-ChangeTracking.md)). Use them for bulk operations expressible in LINQ; reserve raw `ExecuteSql` for SQL EF can't express (stored procs, DB-specific DDL).

---

## SQL injection — parameterize, never concatenate

The cardinal rule of raw SQL: **never build SQL by concatenating user input.** EF's interpolated-string APIs parameterize automatically — the interpolated values become **SQL parameters**, not literal text:

```csharp
// ✓ SAFE — interpolation is parameterized by FromSql/ExecuteSql
db.Products.FromSql($"SELECT * FROM Products WHERE Name = {userInput}");
//   → SELECT * FROM Products WHERE Name = @p0   (userInput bound as a parameter)

// ✗ DANGER — string concatenation/format injects raw text → SQL injection!
db.Products.FromSqlRaw("SELECT * FROM Products WHERE Name = '" + userInput + "'");   // VULNERABLE
db.Products.FromSqlRaw($"SELECT * FROM Products WHERE Name = '{userInput}'");          // ALSO vulnerable!
```

Key distinction:
- **`FromSql`/`ExecuteSql`** take a `FormattableString` (interpolated) and **parameterize** the holes — safe.
- **`FromSqlRaw`/`ExecuteSqlRaw`** take a plain string — you must pass parameters explicitly (`FromSqlRaw("... WHERE Name = {0}", userInput)`), and concatenating into them is **injectable**.

⚠ **Beware**: an interpolated string passed to the **`...Raw`** overload is NOT safe — the interpolation happens *before* EF sees it, producing a concatenated string. Use the non-`Raw` (`FromSql`/`ExecuteSql`) methods with interpolation, or pass explicit parameters to the `Raw` methods. This subtlety is a common vulnerability.

```csharp
// If you must use Raw, pass parameters explicitly:
db.Products.FromSqlRaw("SELECT * FROM Products WHERE Name = {0}", userInput);   // parameterized, safe
var p = new NpgsqlParameter("name", userInput);
db.Database.ExecuteSqlRaw("UPDATE Products SET X = 1 WHERE Name = @name", p);
```

---

## Stored procedures

```csharp
// Returning entities
var results = await db.Products.FromSql($"EXEC GetActiveProducts {category}").ToListAsync(ct);
// Non-query
await db.Database.ExecuteSqlAsync($"EXEC ArchiveOldOrders {cutoff}", ct);
```

Call stored procedures via `FromSql` (if they return rows mapping to an entity) or `ExecuteSql` (for action procs). Output parameters require explicit `DbParameter`s. Stored procs trade EF's portability/LINQ for DB-side logic — use when the procedure already exists or the logic genuinely belongs in the database.

---

## When to use raw SQL

- **Complex queries** EF can't translate (advanced window functions, recursive CTEs, vendor-specific features).
- **Stored procedures** that already exist.
- **Bulk operations** — though prefer typed `ExecuteUpdate`/`ExecuteDelete` where possible.
- **Performance-critical** queries where you want exact control of the SQL.

When **not** to: ordinary CRUD/queries LINQ handles fine (you lose translation/portability and gain injection risk). Reach for raw SQL deliberately, not by default.

---

## Common gotchas

### Interpolated string into a `...Raw` method → injection

`FromSqlRaw($"...{userInput}...")` concatenates *before* EF sees it — injectable. Use `FromSql` (non-Raw) with interpolation, or pass explicit parameters to `Raw`.

### Concatenating user input

The classic SQL-injection vector. Always parameterize (interpolation via non-Raw methods, or explicit `DbParameter`s).

### `ExecuteSql` bypasses the tracker

Direct SQL UPDATE/DELETE doesn't update tracked entities in memory — they're now stale. Reload, or run such operations before loading, or use a fresh context.

### `FromSql` column mismatch

The query must return all columns the entity maps (matching names). Missing columns → runtime error. Project with `SqlQuery<T>` for arbitrary shapes.

### Losing portability

Raw SQL is provider-specific (T-SQL vs PostgreSQL syntax differs). It ties your code to one database — fine if intentional, a problem if you target multiple.

### Composing on `FromSqlRaw` with non-composable SQL

You can compose LINQ on `FromSql` only if the SQL is composable (a `SELECT`, not a stored proc with side effects). Non-composable SQL must be the terminal query.

---

## Summary

- Drop to raw SQL with **`FromSql`** (queries returning entities, composable with LINQ), **`SqlQuery<T>`** (arbitrary shapes), and **`ExecuteSql`** (non-query/DDL/procs) — for SQL EF can't generate.
- Prefer typed **`ExecuteUpdate`/`ExecuteDelete`** (EF 7+) for bulk operations (injection-safe by construction) over raw SQL strings.
- **Parameterize, never concatenate**: `FromSql`/`ExecuteSql` (interpolated `FormattableString`) auto-parameterize and are **safe**; **`FromSqlRaw`/`ExecuteSqlRaw`** take plain strings — interpolating into them is **injectable** (pass explicit parameters instead).
- `ExecuteSql`/`ExecuteUpdate`/`ExecuteDelete` **bypass the change tracker** — tracked entities go stale; reload as needed.
- Raw SQL trades EF's portability and LINQ translation for control — use it deliberately (complex queries, procs, perf), not for ordinary CRUD.

→ Next: [07-Concurrency.md](07-Concurrency.md)
