# Chapter 05 — EF Core — Coding Problems

Model a domain, query it performantly, fix N+1, handle concurrency, and implement soft delete + auditing with interceptors and query filters.

---

## Problem 1: Model a domain with relationships

Define `Customer` (1) → `Order` (N), `Order` (N) ↔ `Product` (N) via order lines, and configure with the fluent API.

<details><summary>Solution</summary>

```csharp
public class Customer { public int Id; public string Name = ""; public List<Order> Orders = []; }
public class Order {
    public int Id; public int CustomerId; public Customer Customer = null!;
    public DateTimeOffset Created; public List<OrderLine> Lines = [];
}
public class OrderLine {                       // join with payload (Quantity, UnitPrice)
    public int OrderId; public int ProductId;
    public int Quantity; public decimal UnitPrice;
}
public class Product { public int Id; public string Name = ""; public decimal Price; }

protected override void OnModelCreating(ModelBuilder b) {
    b.Entity<Order>()
        .HasOne(o => o.Customer).WithMany(c => c.Orders)
        .HasForeignKey(o => o.CustomerId).OnDelete(DeleteBehavior.Restrict);   // don't cascade-delete orders

    b.Entity<OrderLine>().HasKey(l => new { l.OrderId, l.ProductId });          // composite key
    b.Entity<OrderLine>().HasOne<Order>().WithMany(o => o.Lines).HasForeignKey(l => l.OrderId);
    b.Entity<OrderLine>().HasOne<Product>().WithMany().HasForeignKey(l => l.ProductId);

    b.Entity<Product>().HasIndex(p => p.Name);                                   // index for lookups
}
```

`OrderLine` is an explicit join entity (carries Quantity/UnitPrice) with a composite key; `Restrict` prevents cascade-deleting orders with a customer. ([04-Relationships.md](04-Relationships.md).)

</details>

---

## Problem 2: Fix an N+1 query

This loads orders then queries items per order. Fix it two ways.

```csharp
var orders = await db.Orders.ToListAsync(ct);
foreach (var o in orders) Console.WriteLine($"{o.Id}: {o.Lines.Sum(l => l.Quantity)}");   // N+1!
```

<details><summary>Solution</summary>

```csharp
// Option A — eager load (one round-trip, get full entities)
var orders = await db.Orders.Include(o => o.Lines).ToListAsync(ct);
foreach (var o in orders) Console.WriteLine($"{o.Id}: {o.Lines.Sum(l => l.Quantity)}");

// Option B — project the aggregate (best: DB computes it, minimal data, untracked)
var summaries = await db.Orders
    .Select(o => new { o.Id, TotalQty = o.Lines.Sum(l => l.Quantity) })
    .ToListAsync(ct);
foreach (var s in summaries) Console.WriteLine($"{s.Id}: {s.TotalQty}");
```

Option B is usually best for reads — the DB does the SUM, only two columns transfer, nothing is tracked. ([02-Querying.md](02-Querying.md), [09-Performance.md](09-Performance.md).)

</details>

---

## Problem 3: Read-optimized query

Return a paged, filtered product list as DTOs, untracked, ordered.

<details><summary>Solution</summary>

```csharp
public record ProductDto(int Id, string Name, decimal Price);

async Task<List<ProductDto>> GetPageAsync(string category, int page, int size, CancellationToken ct) =>
    await db.Products
        .AsNoTracking()                              // read-only
        .Where(p => p.Category == category)           // SQL filter (index it!)
        .OrderBy(p => p.Name)                         // order before paging
        .Skip(size * (page - 1)).Take(size)
        .Select(p => new ProductDto(p.Id, p.Name, p.Price))   // project (only 3 columns)
        .ToListAsync(ct);
```

Projection + `AsNoTracking` + ordering before paging + an index on `Category` — the read-performance checklist. For deep pages, prefer keyset pagination (`WHERE Id > lastSeen`). ([02-Querying.md](02-Querying.md).)

</details>

---

## Problem 4: Bulk update without loading

Mark all products in a category inactive — efficiently.

<details><summary>Solution</summary>

```csharp
// ✓ — one SQL UPDATE, no loading, no tracking (EF 7+)
int affected = await db.Products
    .Where(p => p.Category == "Discontinued")
    .ExecuteUpdateAsync(s => s.SetProperty(p => p.IsActive, false), ct);

// (vs the slow way: load all rows, set IsActive=false on each, SaveChanges)
```

`ExecuteUpdate` issues a single `UPDATE ... WHERE` — no round-trips to fetch, no tracking. Note it bypasses `SaveChanges`/interceptors. ([03-ChangeTracking.md](03-ChangeTracking.md), [09-Performance.md](09-Performance.md).)

</details>

---

## Problem 5: Safe partial update (load-then-mutate)

Update a product's name and price from a DTO without over-writing other columns or risking lost updates.

<details><summary>Solution</summary>

```csharp
async Task<IResult> UpdateAsync(int id, UpdateProductDto dto, byte[] rowVersion, CancellationToken ct) {
    var product = await db.Products.FindAsync([id], ct);
    if (product is null) return Results.NotFound();

    db.Entry(product).Property(p => p.RowVersion).OriginalValue = rowVersion;   // concurrency check
    product.Name = dto.Name;          // only changed columns are updated
    product.Price = dto.Price;

    try { await db.SaveChangesAsync(ct); return Results.NoContent(); }
    catch (DbUpdateConcurrencyException) { return Results.Conflict("Modified by another user"); }
}
```

Load-then-mutate updates only changed columns (vs `Update()` writing all), and the rowversion check detects concurrent modifications → 409. ([03-ChangeTracking.md](03-ChangeTracking.md), [07-Concurrency.md](07-Concurrency.md).)

</details>

---

## Problem 6: Optimistic concurrency with rowversion

Add a concurrency token and handle the conflict with a "store wins" reload.

<details><summary>Solution</summary>

```csharp
public class Account { public int Id; public decimal Balance; [Timestamp] public byte[] RowVersion = null!; }

async Task WithdrawAsync(int id, decimal amount, CancellationToken ct) {
    while (true) {
        var account = await db.Accounts.FindAsync([id], ct);
        if (account!.Balance < amount) throw new InvalidOperationException("Insufficient funds");
        account.Balance -= amount;
        try { await db.SaveChangesAsync(ct); return; }
        catch (DbUpdateConcurrencyException ex) {
            // store wins: reload the current DB values and retry
            await ex.Entries.Single().ReloadAsync(ct);
            // loop retries with fresh balance (cap retries in real code)
        }
    }
}
```

`[Timestamp]` makes `RowVersion` a DB-managed concurrency token. On conflict, reload and retry (store-wins). Cap retries in production to avoid infinite loops. ([07-Concurrency.md](07-Concurrency.md).)

</details>

---

## Problem 7: Soft delete (interceptor + query filter)

Implement transparent soft delete so `Remove` flags rows and queries hide them.

<details><summary>Solution</summary>

```csharp
public interface ISoftDelete { bool IsDeleted { get; set; } }

// 1. Interceptor — convert DELETE to flag update (write side)
public class SoftDeleteInterceptor : SaveChangesInterceptor {
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData e, InterceptionResult<int> r, CancellationToken ct = default) {
        foreach (var entry in e.Context!.ChangeTracker.Entries<ISoftDelete>())
            if (entry.State == EntityState.Deleted) {
                entry.State = EntityState.Modified;
                entry.Entity.IsDeleted = true;
            }
        return base.SavingChangesAsync(e, r, ct);
    }
}

// 2. Query filter — hide deleted rows (read side)
protected override void OnModelCreating(ModelBuilder b) =>
    b.Entity<Product>().HasQueryFilter(p => !p.IsDeleted);

// 3. Register the interceptor
o.UseNpgsql(cs).AddInterceptors(new SoftDeleteInterceptor());

// Usage — looks like a delete, behaves as soft delete:
db.Products.Remove(product);
await db.SaveChangesAsync(ct);                 // row kept, IsDeleted=true
var visible = await db.Products.ToListAsync(ct); // excludes soft-deleted
var all = await db.Products.IgnoreQueryFilters().ToListAsync(ct);   // admin: see deleted too
```

Both halves are required — interceptor for writes, filter for reads. ([11-Interceptors.md](11-Interceptors.md), [13-GlobalQueryFilters.md](13-GlobalQueryFilters.md).)

</details>

---

## Problem 8: Audit stamping via interceptor

Auto-set `CreatedAt`/`UpdatedAt` (and `CreatedBy`) on save.

<details><summary>Solution</summary>

```csharp
public interface IAuditable {
    DateTimeOffset CreatedAt { get; set; } DateTimeOffset? UpdatedAt { get; set; } string? CreatedBy { get; set; }
}

public class AuditInterceptor(IUserContext user, TimeProvider clock) : SaveChangesInterceptor {
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData e, InterceptionResult<int> r, CancellationToken ct = default) {
        var now = clock.GetUtcNow();
        foreach (var entry in e.Context!.ChangeTracker.Entries<IAuditable>()) {
            if (entry.State == EntityState.Added) { entry.Entity.CreatedAt = now; entry.Entity.CreatedBy = user.Name; }
            if (entry.State == EntityState.Modified) entry.Entity.UpdatedAt = now;
        }
        return base.SavingChangesAsync(e, r, ct);
    }
}
// Register scoped (needs per-request IUserContext); resolve in the options factory.
```

Every `IAuditable` entity is stamped automatically — no per-service code. Inject `TimeProvider` so it's testable ([Ch02 §06](../02-BCL/06-DateTimeAndTime.md)). ([11-Interceptors.md](11-Interceptors.md).)

</details>

---

## Problem 9: Atomic multi-step operation

Place an order: insert the order, decrement inventory, add loyalty points — atomically.

<details><summary>Solution</summary>

```csharp
async Task PlaceOrderAsync(Order order, CancellationToken ct) {
    // All changes share ONE SaveChanges → one transaction → atomic. No explicit transaction needed.
    db.Orders.Add(order);

    var inventory = await db.Inventory.FindAsync([order.ProductId], ct)
        ?? throw new InvalidOperationException("No inventory record");
    if (inventory.Quantity < order.Quantity) throw new InvalidOperationException("Out of stock");
    inventory.Quantity -= order.Quantity;

    var customer = await db.Customers.FindAsync([order.CustomerId], ct);
    customer!.LoyaltyPoints += 10;

    await db.SaveChangesAsync(ct);   // INSERT + 2 UPDATEs commit together, or none do
}
```

One `SaveChanges` = one transaction = atomic. Don't split into multiple `SaveChanges` (partial-state risk). For cross-DB or DB+broker atomicity, use the outbox pattern, not a distributed transaction. ([08-Transactions.md](08-Transactions.md).)

</details>

---

## Problem 10: Test EF logic with SQLite in-memory

Write an xUnit fixture and a test that verifies a query against a real SQL engine.

<details><summary>Solution</summary>

```csharp
public class DbFixture : IDisposable {
    private readonly SqliteConnection _conn;
    public AppDbContext CreateContext() =>
        new(new DbContextOptionsBuilder<AppDbContext>().UseSqlite(_conn).Options);
    public DbFixture() {
        _conn = new SqliteConnection("DataSource=:memory:");
        _conn.Open();                                  // keep open — closing drops the DB
        using var ctx = CreateContext();
        ctx.Database.EnsureCreated();
    }
    public void Dispose() => _conn.Dispose();
}

public class ProductQueryTests(DbFixture fx) : IClassFixture<DbFixture> {
    [Fact]
    public async Task GetActive_ExcludesInactive() {
        await using var db = fx.CreateContext();
        db.Products.AddRange(
            new Product { Name = "A", IsActive = true },
            new Product { Name = "B", IsActive = false });
        await db.SaveChangesAsync();

        var active = await db.Products.Where(p => p.IsActive).ToListAsync();
        active.Should().ContainSingle(p => p.Name == "A");   // real SQL translation + execution
    }
}
```

SQLite in-memory runs real SQL (catches translation errors, enforces constraints) — far better than the InMemory provider. Keep the connection open for the fixture's lifetime. ([14-Testing.md](14-Testing.md).)

</details>

---

You can now model a domain with EF Core, write performant queries (no N+1, projected, untracked, indexed), do safe partial updates with optimistic concurrency, run atomic multi-step operations, implement transparent soft delete + auditing via interceptors and query filters, and test against a real SQL engine.

→ Back to [Chapter 05 README](README.md) · Next chapter: [Chapter 06 — Data Access & Caching](../06-DataAndCaching/README.md)
