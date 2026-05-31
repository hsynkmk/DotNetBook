# Dapper

## The micro-ORM

Dapper is a tiny, fast data-access library that maps SQL query results to objects — without EF Core's change tracking, migrations, or LINQ translation. You write the SQL; Dapper handles parameterization and materialization. It's the go-to when you want **SQL control + speed** and don't need EF's higher-level features.

```csharp
using Dapper;
using var conn = new NpgsqlConnection(connectionString);

// Query → list of mapped objects
var products = await conn.QueryAsync<Product>(
    "SELECT Id, Name, Price FROM Products WHERE Category = @category",
    new { category });                                  // parameterized (safe)

// Single result
var product = await conn.QuerySingleOrDefaultAsync<Product>(
    "SELECT * FROM Products WHERE Id = @id", new { id });

// Non-query
int rows = await conn.ExecuteAsync(
    "UPDATE Products SET Price = @price WHERE Id = @id", new { id, price });
```

Dapper extends `IDbConnection` with `Query`/`Execute` methods. It's essentially "ADO.NET with automatic object mapping and parameterization" — near-raw-SQL performance with far less boilerplate.

---

## Dapper vs EF Core

| | Dapper | EF Core |
|---|---|---|
| You write | SQL | LINQ (translated to SQL) |
| Change tracking | none | yes |
| Migrations | none (manage schema yourself) | yes |
| Mapping | result columns → object | full ORM |
| Performance | minimal overhead (near raw ADO.NET) | more overhead |
| Best for | read-heavy queries, complex SQL, perf-critical | rich domain modeling, CRUD, productivity |

They're not mutually exclusive — many apps use **EF Core for writes/domain logic** (change tracking, migrations) and **Dapper for read-heavy or complex queries** (reporting, dashboards) where hand-tuned SQL and minimal overhead matter. Use the right tool per operation.

---

## Parameterization — always (SQL injection safety)

Dapper parameterizes via anonymous objects — **never** concatenate user input into SQL:

```csharp
// ✓ SAFE — @category becomes a SQL parameter
await conn.QueryAsync<Product>("SELECT * FROM Products WHERE Category = @category", new { category });

// ✗ DANGER — string concatenation → SQL injection
await conn.QueryAsync<Product>($"SELECT * FROM Products WHERE Category = '{category}'");   // VULNERABLE
```

The anonymous object's properties become named parameters (`@category` → the `category` value, sent separately from the SQL). This is injection-safe and the same protection EF and ADO.NET provide. Concatenating values into the SQL string is the classic vulnerability — don't.

```csharp
// IN clauses — Dapper expands a list to multiple parameters
await conn.QueryAsync<Product>("SELECT * FROM Products WHERE Id IN @ids", new { ids = new[] { 1, 2, 3 } });
//   → WHERE Id IN (@ids1, @ids2, @ids3)
```

---

## Query methods

```csharp
conn.Query<T>(sql, param)                 // IEnumerable<T> (buffered by default)
conn.QueryAsync<T>(sql, param)            // async
conn.QueryFirstOrDefaultAsync<T>(...)     // first row or null
conn.QuerySingleAsync<T>(...)             // exactly one (throws if 0 or >1)
conn.QuerySingleOrDefaultAsync<T>(...)    // 0 or 1
conn.ExecuteAsync(sql, param)             // non-query → affected rows
conn.ExecuteScalarAsync<T>(sql, param)    // single scalar value (COUNT, SUM)
conn.QueryMultipleAsync(sql, param)       // multiple result sets in one round-trip
```

```csharp
// Multiple result sets — one round-trip, several queries
using var multi = await conn.QueryMultipleAsync(
    "SELECT * FROM Orders WHERE Id=@id; SELECT * FROM OrderLines WHERE OrderId=@id", new { id });
var order = await multi.ReadSingleAsync<Order>();
var lines = (await multi.ReadAsync<OrderLine>()).ToList();
```

`QueryMultiple` batches several queries into one DB round-trip — efficient for fetching an aggregate (order + its lines) without N+1 or a complex join.

---

## Multi-mapping (joins → object graphs)

Dapper can split a joined row across multiple objects and let you assemble the graph:

```csharp
var sql = @"SELECT o.*, c.* FROM Orders o JOIN Customers c ON o.CustomerId = c.Id WHERE o.Id = @id";
var orders = await conn.QueryAsync<Order, Customer, Order>(sql,
    (order, customer) => { order.Customer = customer; return order; },   // map function
    new { id }, splitOn: "Id");                                           // where the second object starts
```

The `splitOn` parameter tells Dapper which column begins the next object. Multi-mapping handles joins explicitly (you control the SQL and the assembly) — more manual than EF's `Include`, but you get exactly the SQL you want. For one-to-many, you typically dedupe parents in the map function (or use `QueryMultiple`).

---

## A Dapper repository

```csharp
public class ProductRepository(IDbConnectionFactory factory) {   // factory creates connections
    public async Task<Product?> GetAsync(int id, CancellationToken ct) {
        using var conn = await factory.CreateConnectionAsync(ct);
        return await conn.QuerySingleOrDefaultAsync<Product>(
            new CommandDefinition("SELECT * FROM Products WHERE Id = @id", new { id }, cancellationToken: ct));
    }

    public async Task<int> CreateAsync(CreateProduct p, CancellationToken ct) {
        using var conn = await factory.CreateConnectionAsync(ct);
        return await conn.ExecuteScalarAsync<int>(new CommandDefinition(
            "INSERT INTO Products (Name, Price) VALUES (@Name, @Price) RETURNING Id", p, cancellationToken: ct));
    }
}
```

Use `CommandDefinition` to pass a `CancellationToken`. Manage connections via a factory (or open per-operation) — connections come from the ADO.NET pool ([02-ADO.NET.md](02-ADO.NET.md)), so opening/closing per query is cheap. Don't share a connection across threads.

---

## When to use Dapper

- **Read-heavy / reporting** queries where you want tuned SQL and minimal overhead.
- **Complex SQL** (window functions, CTEs, vendor features) that's awkward in LINQ.
- **Performance-critical** hot paths where EF's overhead matters (measured).
- Alongside EF Core (EF for writes/domain, Dapper for reads).

When **not** to: rich domain modeling with change tracking, schema migrations, and rapid CRUD — EF Core is more productive there. Dapper trades EF's features for control and speed.

---

## Common gotchas

### String concatenation → SQL injection

The cardinal sin. Always pass parameters via the anonymous object; never interpolate user input into the SQL.

### Column/property name mismatch

Dapper maps by name (case-insensitive). A column not matching a property is unmapped (null/default). Alias columns in SQL (`SELECT price AS Price`) or match names; for snake_case DBs, enable `DefaultTypeMap.MatchNamesWithUnderscores = true`.

### Sharing a connection across threads

ADO.NET connections aren't thread-safe. Open a connection per operation (cheap — pooled) or use a factory; don't share one across concurrent queries.

### Forgetting `splitOn` in multi-mapping

Without the right `splitOn`, Dapper can't tell where one object ends and the next begins → mapping errors. Set it to the first column of each subsequent object.

### Buffered vs unbuffered queries

`Query` buffers all rows into memory by default. For huge result sets, pass `buffered: false` to stream — but then keep the connection open while enumerating.

### Expecting change tracking

Dapper returns plain objects with no tracking — modifying them and "saving" does nothing. Write an explicit `Execute`/`UPDATE`. (That's the point — it's not an ORM.)

---

## Summary

- **Dapper** is a fast micro-ORM: you write SQL, it parameterizes and maps results to objects — near-raw-ADO.NET performance, far less boilerplate, no change tracking/migrations.
- **Always parameterize** (anonymous objects → SQL parameters); never concatenate user input (injection).
- Rich query API: `Query`/`QuerySingle`/`Execute`/`ExecuteScalar`/`QueryMultiple` (batch result sets) and **multi-mapping** (`splitOn`) for joins.
- Use Dapper for **read-heavy/complex/perf-critical** queries — often **alongside EF Core** (EF for writes/domain, Dapper for reads); EF wins for rich modeling, change tracking, and migrations.
- Open connections per operation (pooled), pass a `CancellationToken` via `CommandDefinition`, and match column/property names.

→ Next: [02-ADO.NET.md](02-ADO.NET.md)
