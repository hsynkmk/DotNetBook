# Transactions

## Atomic units of work

A transaction groups operations so they **all** succeed or **all** roll back — no partial state. EF Core gives you transactions implicitly (per `SaveChanges`) and explicitly (spanning multiple saves or raw commands). The goal: keep your data consistent even when something fails midway.

```csharp
// Implicit — SaveChanges is ALREADY a transaction
db.Orders.Add(order);
order.Customer.LoyaltyPoints += 10;
db.Inventory.First(i => i.ProductId == id).Quantity -= 1;
await db.SaveChangesAsync(ct);
// → INSERT + 2 UPDATEs in ONE transaction; if any fails, NONE are applied
```

---

## Implicit transactions (the common case)

**Every `SaveChanges` call is wrapped in a transaction automatically.** All the INSERTs/UPDATEs/DELETEs from one `SaveChanges` commit together or roll back together. So if you make all your related changes and call `SaveChanges` **once**, you already have atomicity — no explicit transaction needed:

```csharp
// All these changes are one atomic unit because they share one SaveChanges:
db.Accounts.First(a => a.Id == from).Balance -= amount;
db.Accounts.First(a => a.Id == to).Balance += amount;
await db.SaveChangesAsync(ct);   // both updates commit together, or neither does
```

This is the **unit-of-work** guarantee from [01-DbContext.md](01-DbContext.md). The mistake to avoid: calling `SaveChanges` multiple times for one logical operation — each is a separate transaction, so a failure between them leaves partial state.

```csharp
// ✗ — two transactions; if the second fails, the first is already committed (inconsistent!)
db.Accounts.First(a => a.Id == from).Balance -= amount;
await db.SaveChangesAsync(ct);   // committed
db.Accounts.First(a => a.Id == to).Balance += amount;
await db.SaveChangesAsync(ct);   // if this fails → money debited but not credited
```

---

## Explicit transactions

When a unit of work spans **multiple `SaveChanges`** calls, or mixes EF with raw SQL, wrap them in an explicit transaction:

```csharp
await using var tx = await db.Database.BeginTransactionAsync(ct);
try {
    db.Orders.Add(order);
    await db.SaveChangesAsync(ct);                       // first save

    await db.Database.ExecuteSqlAsync($"UPDATE Stats SET OrderCount = OrderCount + 1", ct);  // raw SQL

    order.Status = OrderStatus.Confirmed;
    await db.SaveChangesAsync(ct);                       // second save

    await tx.CommitAsync(ct);                            // commit ALL of it together
} catch {
    await tx.RollbackAsync(ct);                          // (or rely on dispose to roll back)
    throw;
}
```

`BeginTransaction` returns an `IDbContextTransaction`; everything until `Commit` is in one transaction. **If you don't commit (exception, or dispose without commit), it rolls back** — so the `await using` + commit-at-the-end pattern is safe by default. Use explicit transactions only when one `SaveChanges` isn't enough.

---

## Isolation levels

A transaction's **isolation level** controls how it interacts with concurrent transactions — the trade-off between consistency and concurrency:

```csharp
await using var tx = await db.Database.BeginTransactionAsync(IsolationLevel.Serializable, ct);
```

| Level | Prevents | Trade-off |
|---|---|---|
| `ReadUncommitted` | nothing (dirty reads possible) | fastest, least safe |
| `ReadCommitted` (typical default) | dirty reads | good balance |
| `RepeatableRead` | dirty + non-repeatable reads | more locking |
| `Serializable` | all anomalies (as if serial) | safest, most contention |
| `Snapshot` | readers don't block writers (MVCC) | DB-specific (SQL Server) |

The default is usually `ReadCommitted` (database-dependent). Higher isolation prevents more anomalies (dirty/non-repeatable/phantom reads) but increases locking/contention. Most apps run at the default and use **optimistic concurrency tokens** ([07-Concurrency.md](07-Concurrency.md)) for cross-request conflicts rather than raising isolation. Raise the level only for specific operations that need stronger guarantees.

---

## Execution strategies & retries

With a retrying execution strategy (for transient faults — common with cloud databases), you **cannot** just wrap `BeginTransaction` in a manual try/catch, because a retry would re-run only part. Use the strategy's `ExecuteAsync` to make the whole transaction retryable:

```csharp
// Enable retries (e.g., for SQL Azure transient errors)
o.UseSqlServer(cs, sql => sql.EnableRetryOnFailure());

// Wrap a multi-step transaction so the WHOLE thing retries atomically
var strategy = db.Database.CreateExecutionStrategy();
await strategy.ExecuteAsync(async () => {
    await using var tx = await db.Database.BeginTransactionAsync(ct);
    await db.SaveChangesAsync(ct);
    await db.Database.ExecuteSqlAsync($"...", ct);
    await tx.CommitAsync(ct);
});
```

`EnableRetryOnFailure` retries transient errors (deadlocks, timeouts, transient cloud faults) automatically for single operations; for explicit multi-step transactions, wrap them in `CreateExecutionStrategy().ExecuteAsync` so a retry re-runs the entire transaction. (Resilience: [Ch11](../11-Resilience/README.md).)

---

## Sharing a transaction across contexts

Occasionally multiple `DbContext`s (or a context + raw ADO.NET) must share one transaction:

```csharp
await using var conn = new NpgsqlConnection(cs);
await conn.OpenAsync(ct);
await using var tx = await conn.BeginTransactionAsync(ct);

using var ctx1 = new AppDbContext(optionsUsing(conn));
ctx1.Database.UseTransaction(tx.GetDbTransaction());
// ... ctx1 operations ...
await tx.CommitAsync(ct);
```

`UseTransaction` enlists a context in an externally-managed transaction (sharing one connection). Niche; needed when coordinating multiple contexts atomically.

---

## Distributed transactions — avoid when possible

A transaction spanning **multiple databases/resources** (two databases, a DB + a message broker) requires a distributed transaction (two-phase commit), which is **complex, slow, and poorly supported** (limited/Windows-only in .NET). Strongly prefer to avoid them:

- **Don't** span a DB write and a message publish in one transaction. Use the **outbox pattern** instead: write the message to an outbox table **in the same DB transaction** as the business change, then a background process publishes it ([Ch08 Background Processing](../08-BackgroundProcessing/README.md), [Ch07 Messaging](../07-Messaging/README.md)).
- For cross-service consistency, use **sagas** / eventual consistency, not distributed transactions.
- Keep each transaction within a single database/resource.

This is a key distributed-systems principle: design so a single local transaction suffices; achieve cross-resource consistency through outbox/sagas/idempotency rather than 2PC.

---

## Common gotchas

### Multiple `SaveChanges` for one logical operation

Each `SaveChanges` is its own transaction; a failure between them leaves partial state. Make all related changes and `SaveChanges` **once**, or wrap multiple saves in an explicit transaction.

### Manual transaction + retry strategy conflict

With `EnableRetryOnFailure`, manually wrapping `BeginTransaction` in try/catch breaks retries. Use `CreateExecutionStrategy().ExecuteAsync` around the whole transaction.

### Forgetting to commit

An explicit transaction not committed (or disposed before commit) rolls back. The `await using` + commit-last pattern ensures rollback on any exception — but make sure you *do* reach `CommitAsync`.

### Raising isolation instead of using concurrency tokens

`Serializable` everywhere causes heavy contention/deadlocks. For cross-request conflicts use optimistic **concurrency tokens** ([07-Concurrency.md](07-Concurrency.md)); raise isolation only for specific operations that truly need it.

### Distributed transactions across DB + broker

Don't try to atomically commit a DB change and a message send. Use the **outbox pattern** (message in the same DB transaction, published later).

### Long-running transactions

Holding a transaction open across user think-time or slow work locks rows and kills concurrency. Keep transactions short.

---

## Summary

- **Every `SaveChanges` is a transaction** — make all related changes and save **once** for atomicity; calling `SaveChanges` multiple times for one operation risks partial state.
- Use **explicit transactions** (`BeginTransactionAsync` + `CommitAsync`, `await using` for auto-rollback) only when a unit of work spans multiple saves or mixes raw SQL.
- **Isolation levels** trade consistency for concurrency (default `ReadCommitted`); prefer **optimistic concurrency tokens** over high isolation for cross-request conflicts.
- With **retrying strategies** (`EnableRetryOnFailure`), wrap multi-step transactions in `CreateExecutionStrategy().ExecuteAsync` so retries re-run the whole transaction.
- **Avoid distributed transactions** (slow, poorly supported) — keep each transaction within one database and use the **outbox pattern**/sagas for cross-resource consistency. Keep transactions short.

→ Next: [09-Performance.md](09-Performance.md)
