# ADO.NET

## The foundation under every data library

ADO.NET is the low-level data-access API that **everything else is built on** — EF Core, Dapper, and the database providers all sit on top of `DbConnection`, `DbCommand`, and `DbDataReader`. You rarely use it directly (Dapper/EF are more productive), but understanding it explains connection pooling, parameterization, and what the higher layers actually do.

```csharp
await using var conn = new NpgsqlConnection(connectionString);
await conn.OpenAsync(ct);

await using var cmd = conn.CreateCommand();
cmd.CommandText = "SELECT Id, Name, Price FROM Products WHERE Category = @category";
cmd.Parameters.AddWithValue("@category", category);          // parameterized

await using var reader = await cmd.ExecuteReaderAsync(ct);
var products = new List<Product>();
while (await reader.ReadAsync(ct))
    products.Add(new Product {
        Id = reader.GetInt32(0), Name = reader.GetString(1), Price = reader.GetDecimal(2)
    });
```

That's the raw machinery: open a connection, create a command, add parameters, execute, read rows. Dapper collapses all of this into one `QueryAsync<Product>` call — which is why you usually use Dapper or EF instead.

---

## The core types

| Type | Role |
|---|---|
| `DbConnection` | a connection to the database (provider-specific: `NpgsqlConnection`, `SqlConnection`) |
| `DbCommand` | a SQL statement / stored procedure to execute |
| `DbParameter` | a parameter (safe value substitution) |
| `DbDataReader` | forward-only, read-only stream of result rows |
| `DbTransaction` | a transaction scope |
| `DbDataSource` (.NET 7+) | a factory for connections/commands (modern entry point) |

Each provider (Npgsql for PostgreSQL, Microsoft.Data.SqlClient for SQL Server, Microsoft.Data.Sqlite) implements these abstract base classes. Programming against the `Db*` base types keeps code provider-agnostic; using the concrete types unlocks provider-specific features.

---

## Executing commands

```csharp
// ExecuteReader — rows (SELECT)
await using var reader = await cmd.ExecuteReaderAsync(ct);

// ExecuteNonQuery — affected row count (INSERT/UPDATE/DELETE/DDL)
int rows = await cmd.ExecuteNonQueryAsync(ct);

// ExecuteScalar — a single value (COUNT, SUM, a returned id)
var count = (long)(await cmd.ExecuteScalarAsync(ct))!;
```

`ExecuteReader` streams rows via a `DbDataReader` (forward-only — read each row once, in order). `ExecuteNonQuery` returns the affected-row count. `ExecuteScalar` returns the first column of the first row (for aggregates or `RETURNING`/`OUTPUT` ids).

---

## Parameters — safety and correctness

```csharp
// AddWithValue — convenient but lets the provider infer the type (can pick wrong)
cmd.Parameters.AddWithValue("@name", name);

// Explicit type — preferred for correctness/performance (avoids type-inference surprises)
var p = cmd.CreateParameter();
p.ParameterName = "@price"; p.DbType = DbType.Decimal; p.Value = price;
cmd.Parameters.Add(p);
```

**Always use parameters** — never concatenate values into `CommandText` (SQL injection). `AddWithValue` is handy but can infer a suboptimal type (e.g., the wrong string length, causing plan-cache bloat or implicit conversions); for hot paths or precise types, set the parameter type explicitly. Output/return parameters (`ParameterDirection.Output`) capture stored-proc outputs.

---

## Connection pooling (the key performance fact)

Opening a physical database connection is expensive (TCP handshake, auth). ADO.NET **pools** connections: `OpenAsync` usually takes one from the pool, and `Dispose`/`CloseAsync` **returns it to the pool** rather than closing the physical connection. So:

```csharp
// ✓ — open late, close early; the pool makes this cheap. Don't hold connections open.
await using var conn = new NpgsqlConnection(cs);
await conn.OpenAsync(ct);
// ... use it briefly ...
// disposed → returned to the pool (NOT physically closed)
```

Implications:
- **Open connections as late as possible, dispose as soon as done** — holding a connection open ties up a pool slot; the pool is small (default ~100). Holding many open → pool exhaustion → timeouts.
- The connection **string** controls pool size (`Max Pool Size`, `Min Pool Size`). Connections are pooled per distinct connection string.
- This is why EF/Dapper open a connection per operation and close it immediately — the pool makes that cheap, and it maximizes connection reuse. **Don't** keep a connection open "to be efficient" — it's the opposite.

`DbDataSource` (.NET 7+) is the modern way to manage this — create one `DbDataSource` per connection string at startup and get pooled connections/commands from it.

---

## Transactions

```csharp
await using var conn = new NpgsqlConnection(cs);
await conn.OpenAsync(ct);
await using var tx = await conn.BeginTransactionAsync(ct);
try {
    await using var cmd = conn.CreateCommand();
    cmd.Transaction = tx;                              // enlist the command in the transaction
    cmd.CommandText = "UPDATE Accounts SET Balance = Balance - @amt WHERE Id = @from";
    // ... execute multiple commands ...
    await tx.CommitAsync(ct);
} catch { await tx.RollbackAsync(ct); throw; }
```

Commands must be enlisted in the transaction (`cmd.Transaction = tx`). Same atomic-unit principle as EF's transactions ([Ch05 §08](../05-EFCore/08-Transactions.md)) — commit all or roll back all; keep transactions short.

---

## When to use raw ADO.NET

Almost never directly in application code — Dapper gives you the same control with mapping and far less boilerplate. Reach for raw ADO.NET only for:
- **Bulk operations** via provider-specific APIs (`SqlBulkCopy`, Npgsql `COPY`) — far faster than row-by-row inserts.
- **Streaming huge result sets** with fine-grained reader control.
- **Provider-specific features** not exposed by Dapper/EF.
- **Writing a library** (like Dapper) on top of it.

For everyday queries, use **Dapper** ([01-Dapper.md](01-Dapper.md)) or **EF Core** ([Ch05](../05-EFCore/README.md)). Understanding ADO.NET matters mainly so you grasp pooling, parameterization, and what those libraries do underneath.

---

## Common gotchas

### Holding connections open

The biggest ADO.NET mistake — keeping a connection open across work ties up a pool slot; many held connections exhaust the pool → "timeout obtaining a connection." Open late, dispose early; let pooling handle reuse.

### String concatenation → injection

Concatenating values into `CommandText` is the classic vulnerability. Always use `DbParameter`s.

### `AddWithValue` type inference

It can infer the wrong DB type (string length, decimal precision), causing implicit conversions or plan-cache bloat. Set parameter types explicitly on hot paths.

### Not disposing readers/commands/connections

Leaks connections (and pool slots). Use `await using` for connection, command, and reader.

### Sharing a connection across threads

ADO.NET connections aren't thread-safe. One connection per operation/thread.

### Forgetting to enlist commands in a transaction

A command without `cmd.Transaction = tx` runs outside the transaction. Enlist every command.

---

## Summary

- **ADO.NET** (`DbConnection`/`DbCommand`/`DbDataReader`/`DbParameter`) is the low-level data API that EF Core and Dapper build on; you rarely use it directly but should understand it.
- Execute via `ExecuteReader` (rows), `ExecuteNonQuery` (affected count), `ExecuteScalar` (single value); **always parameterize** (`DbParameter`s) — never concatenate (injection); prefer explicit parameter types over `AddWithValue` on hot paths.
- **Connection pooling** is the key fact: `Open`/`Dispose` rent/return pooled connections — **open late, dispose early**; holding connections open exhausts the small pool. Use `DbDataSource` (.NET 7+) as the modern entry point.
- Enlist commands in a transaction (`cmd.Transaction = tx`); keep transactions short.
- Use raw ADO.NET only for bulk copy (`SqlBulkCopy`/`COPY`), streaming, or provider-specific features — otherwise use **Dapper** or **EF Core**.

→ Next: [03-IMemoryCache.md](03-IMemoryCache.md)
