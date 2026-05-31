# Concurrency

## The lost-update problem

When two users load the same record, both edit it, and both save, the second save silently **overwrites** the first — a lost update. EF Core's **optimistic concurrency** detects this: it checks, at save time, whether the row changed since you loaded it, and throws if so, letting you handle the conflict instead of clobbering data.

```
User A loads Order (v1) ──┐
User B loads Order (v1) ──┤
User A saves (Total=100) → row now v2  ✓
User B saves (Total=200) → EF detects the row changed → DbUpdateConcurrencyException  ✗ (not a silent overwrite)
```

EF uses **optimistic** concurrency (assume conflicts are rare; detect at save) rather than pessimistic locking (lock the row on read) — better for scalability, since it doesn't hold locks across user think-time.

---

## Concurrency tokens

You mark a property as a **concurrency token**; EF includes it in the `WHERE` clause of UPDATE/DELETE, so the statement only affects the row if the token still matches what you loaded:

```csharp
// UPDATE Orders SET Total=@p WHERE Id=@id AND RowVersion=@originalRowVersion
//   → if RowVersion changed, 0 rows affected → EF throws DbUpdateConcurrencyException
```

Two approaches:

### Rowversion / timestamp (recommended)

A database-managed column that auto-increments on every update — the cleanest token:

```csharp
public class Order {
    public int Id { get; set; }
    public decimal Total { get; set; }
    [Timestamp] public byte[] RowVersion { get; set; } = null!;   // SQL Server rowversion
}
// or fluent:
b.Entity<Order>().Property(o => o.RowVersion).IsRowVersion();

// PostgreSQL uses the system xmin column:
b.Entity<Order>().UseXminAsConcurrencyToken();
```

The database updates `RowVersion` automatically on every write, so any concurrent change is detected. This is the recommended approach — no manual token management.

### Property-as-token

Mark an existing property; EF checks its original value on update:

```csharp
b.Entity<Product>().Property(p => p.Price).IsConcurrencyToken();
// UPDATE ... WHERE Id=@id AND Price=@originalPrice  → fails if Price changed concurrently
```

Useful when you want to detect conflicts only on specific fields, but rowversion (detecting *any* change) is usually what you want.

---

## Handling the conflict

When a concurrency conflict occurs, `SaveChanges` throws `DbUpdateConcurrencyException`. You decide the resolution — there's no universally correct one; it's a business decision:

```csharp
try {
    await db.SaveChangesAsync(ct);
} catch (DbUpdateConcurrencyException ex) {
    foreach (var entry in ex.Entries) {
        var current = entry.CurrentValues;                 // what THIS user wants to save
        var database = await entry.GetDatabaseValuesAsync(ct);  // what's actually in the DB now

        if (database is null) {
            // the row was DELETED by someone else → handle (recreate? abort?)
        } else {
            // Resolution strategies:
            // (a) Database wins: discard this user's changes, reload
            // (b) Client wins:   overwrite — set original values to DB values, retry save
            entry.OriginalValues.SetValues(database);       // (b): make the save succeed with client's values
            // (c) Merge:         field-by-field reconcile, then retry
        }
    }
    // retry SaveChanges after resolving (with a retry limit to avoid infinite loops)
}
```

The three classic strategies:
- **Store wins** — discard the user's changes, reload, tell them to re-edit (safest; no data loss but bad UX if frequent).
- **Client wins** — force the user's values to overwrite (use when last-write-wins is acceptable).
- **Merge** — reconcile field-by-field (best UX, most work — show the user both versions).

Pick per scenario; surface a clear message ("this record was changed by someone else").

---

## Concurrency with disconnected entities (web apps)

In a web app, the original token comes from the client (e.g., a hidden form field or ETag), since the entity was loaded in a *previous* request:

```csharp
// The client sends back the RowVersion it originally received
public async Task<IResult> Update(int id, UpdateDto dto, byte[] originalRowVersion) {
    var order = await db.Orders.FindAsync(id);
    if (order is null) return Results.NotFound();

    // Tell EF the original token value so the WHERE clause uses it
    db.Entry(order).Property(o => o.RowVersion).OriginalValue = originalRowVersion;
    order.Total = dto.Total;

    try { await db.SaveChangesAsync(); return Results.NoContent(); }
    catch (DbUpdateConcurrencyException) { return Results.Conflict("Record was modified by another user."); }
}
```

The flow: send the `RowVersion` to the client with the data → client sends it back on update → EF checks it. This maps naturally to HTTP **ETags** + `If-Match` headers for REST APIs (return 409/412 on conflict). The token round-trips through the client because there's no server-side session holding the original.

---

## Concurrency vs transactions

These solve different problems:
- **Concurrency tokens** detect conflicts across **separate requests/transactions** (user A vs user B over time) — optimistic, no locks held.
- **Transactions** ([08-Transactions.md](08-Transactions.md)) ensure a set of operations within **one** unit of work is atomic/consistent.

A transaction doesn't prevent the lost-update problem across requests (the two saves are in different transactions). You need concurrency tokens for that. They're complementary.

---

## Common gotchas

### No concurrency token = silent lost updates

Without a token, the last save wins silently, overwriting others' changes — often unnoticed until data is wrong. Add a rowversion to entities edited by multiple users.

### Catching the exception but not resolving

Catching `DbUpdateConcurrencyException` and just retrying `SaveChanges` without resolving the conflict can loop forever or still clobber. Decide a strategy (store/client/merge) and cap retries.

### Forgetting to round-trip the token (web)

If the client doesn't send back the original token, EF can't detect the conflict. Send the `RowVersion`/ETag to the client and require it on update.

### Treating concurrency as a transaction problem

A transaction won't stop user A and user B (separate requests) from clobbering each other. Use concurrency tokens for cross-request conflict detection.

### Resolving by blind "client wins"

Always overwriting (`OriginalValues.SetValues(database)` then save) discards others' changes — a lost update by another name. Use it only when last-write-wins is genuinely acceptable.

---

## Summary

- EF Core uses **optimistic concurrency**: a **concurrency token** is added to the UPDATE/DELETE `WHERE` clause so a save fails (0 rows → `DbUpdateConcurrencyException`) if the row changed since you loaded it — preventing **silent lost updates** without holding locks.
- Use a **rowversion/timestamp** (`[Timestamp]`/`IsRowVersion()`, or `UseXminAsConcurrencyToken` on PostgreSQL) — DB-managed, detects any change (recommended); or mark a specific property `IsConcurrencyToken`.
- Handle `DbUpdateConcurrencyException` with a deliberate strategy: **store wins**, **client wins**, or **merge** — surface a clear message; cap retries.
- In web apps, **round-trip the token** through the client (hidden field / HTTP **ETag** + `If-Match` → 409/412) since the entity loaded in a prior request.
- Concurrency tokens (cross-request conflict detection) and **transactions** (intra-operation atomicity) are complementary, not interchangeable.

→ Next: [08-Transactions.md](08-Transactions.md)
