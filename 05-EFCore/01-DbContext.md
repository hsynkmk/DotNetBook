# DbContext

## The unit of work and gateway to the database

`DbContext` is the heart of EF Core: it represents a **session** with the database, exposes your tables as `DbSet<T>` properties, tracks changes to entities, and translates LINQ queries to SQL. It's simultaneously a **unit of work** (batches changes and saves them together) and a set of **repositories** (`DbSet<T>` per entity) — which is why wrapping it in *another* repository/UoW layer is often redundant ([Ch22](../22-BestPractices/README.md)).

```csharp
public class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options) {
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder b) {
        // fluent configuration (relationships, keys, constraints) — see 04-Relationships.md
        b.Entity<Product>().Property(p => p.Name).HasMaxLength(100).IsRequired();
    }
}
```

```csharp
// Registration (in the host — Ch03)
builder.Services.AddDbContext<AppDbContext>(o =>
    o.UseNpgsql(builder.Configuration.GetConnectionString("Default")));   // or UseSqlServer/UseSqlite

// Usage (injected, scoped per request)
public class ProductService(AppDbContext db) {
    public async Task<Product?> GetAsync(int id) => await db.Products.FindAsync(id);
}
```

---

## `DbSet<T>` — your tables as queryable collections

Each `DbSet<T>` represents a table (or view) and is an `IQueryable<T>` — LINQ over it translates to SQL ([02-Querying.md](02-Querying.md)):

```csharp
var cheap = await db.Products.Where(p => p.Price < 10).ToListAsync();   // SELECT ... WHERE Price < 10
db.Products.Add(new Product { Name = "Widget", Price = 9.99m });        // staged for INSERT
var p = await db.Products.FindAsync(42);                                 // by primary key (checks tracker first)
db.Products.Remove(p);                                                    // staged for DELETE
await db.SaveChangesAsync();                                              // executes all staged changes in one transaction
```

`Set<T>()` accesses a `DbSet` without a declared property. Operations on a `DbSet` either **query** (translated to SQL) or **stage changes** (tracked until `SaveChanges`).

---

## `DbContext` is scoped — and not thread-safe

The single most important operational fact: **`DbContext` is not thread-safe** and is designed to be **short-lived** (one per unit of work / web request). EF Core registers it as **Scoped** ([Ch03 §03](../03-HostingAndDI/03-Lifetimes.md)):

```csharp
// In ASP.NET Core, one DbContext per request (scoped). Don't share across requests/threads.
```

Consequences:
- **Never use one `DbContext` from multiple threads concurrently** — you'll get "A second operation was started on this context" exceptions or corruption. For parallel work, use a `DbContext` per task via `IDbContextFactory<T>` ([10-DbContextFactory.md](10-DbContextFactory.md)).
- **Never inject a scoped `DbContext` into a singleton** (captive dependency — [Ch03 §03](../03-HostingAndDI/03-Lifetimes.md)). In a singleton/background service, create a scope per work item and resolve the `DbContext` from it.
- A `DbContext` accumulates tracked entities; a long-lived one grows memory and slows down. Keep it to one logical operation.

```csharp
// ✗ — sharing a context across parallel queries
await Task.WhenAll(ids.Select(id => db.Products.FindAsync(id).AsTask()));   // race! same context

// ✓ — a context per parallel operation (via factory)
await Task.WhenAll(ids.Select(async id => {
    await using var ctx = await factory.CreateDbContextAsync();
    return await ctx.Products.FindAsync(id);
}));
```

---

## `SaveChanges` — the unit of work commits

`DbContext` tracks all changes (adds, updates, deletes) you make to tracked entities and applies them **together** in a single transaction when you call `SaveChanges`/`SaveChangesAsync`:

```csharp
db.Products.Add(newProduct);
existingProduct.Price = 12.99m;        // tracked change → UPDATE
db.Orders.Remove(oldOrder);
await db.SaveChangesAsync();            // INSERT + UPDATE + DELETE, all in ONE transaction
```

`SaveChanges` is atomic by default (wraps the statements in a transaction — [08-Transactions.md](08-Transactions.md)) and returns the number of affected rows. This batching is the "unit of work": you make many changes in memory, then commit them as one consistent operation. Change tracking is covered in [03-ChangeTracking.md](03-ChangeTracking.md).

---

## DbContext pooling

Creating a `DbContext` has some overhead (model setup, internal services). For high-throughput apps, **context pooling** reuses context instances (resetting their state between uses) instead of constructing new ones per request:

```csharp
builder.Services.AddDbContextPool<AppDbContext>(o =>
    o.UseNpgsql(connectionString), poolSize: 128);   // reuse up to 128 context instances
```

`AddDbContextPool` reduces allocation/setup cost in hot paths. Caveats: don't store request-specific state in the context (it's reused/reset), and avoid it if your context's constructor does per-request work. For most apps, plain `AddDbContext` is fine; pool when profiling shows context creation is a bottleneck. (Performance: [09-Performance.md](09-Performance.md).)

---

## Connection management

You generally **don't** manage connections — EF Core opens a connection when needed (for a query or `SaveChanges`) and closes it immediately after, returning it to ADO.NET's connection pool. So the `DbContext` doesn't hold a connection open for its whole lifetime; connections are pooled at the ADO.NET level ([Ch06 §02](../06-DataAndCaching/README.md)). You only manage connections explicitly for advanced scenarios (sharing a connection across contexts, explicit transactions — [08-Transactions.md](08-Transactions.md)).

```csharp
// EF Core opens/closes the connection per operation; the connection STRING is what you configure:
o.UseNpgsql(connectionString);   // pooling is handled by the ADO.NET provider
```

---

## Disposal

`DbContext` implements `IDisposable`/`IAsyncDisposable` and frees the change tracker and any open connection on dispose. In DI scenarios, the **container disposes it** at the end of the scope (request) — you don't dispose it manually. When you create one yourself (via a factory or `new`), dispose it (`await using`):

```csharp
await using var ctx = await factory.CreateDbContextAsync();   // you own it → dispose it
// ... use ctx ...

// In ASP.NET Core, injected DbContext is disposed by the scope automatically — don't dispose it yourself
```

---

## Common gotchas

### Sharing a `DbContext` across threads

Not thread-safe → "second operation started on this context" / corruption. One context per unit of work; use `IDbContextFactory` for parallelism.

### Injecting scoped `DbContext` into a singleton

Captive dependency — the context lives forever and is shared across requests. Create a scope per operation in singletons/background services ([Ch03 §03](../03-HostingAndDI/03-Lifetimes.md)).

### Long-lived contexts

A `DbContext` kept alive accumulates tracked entities (memory growth, slower change detection) and a stale view. Keep it short-lived (per request/operation).

### Manual disposal of injected context

Disposing a DI-managed scoped context yourself can cause "context disposed" errors when the scope also disposes it. Let the scope dispose it; only dispose contexts you create.

### Storing state in a pooled context

`AddDbContextPool` resets and reuses contexts — request-specific state set in the constructor/fields won't behave as expected. Don't rely on per-request state in a pooled context.

---

## Summary

- **`DbContext`** is a **unit of work + repositories** (`DbSet<T>`) and the gateway to the DB — it queries (LINQ→SQL), tracks changes, and commits them atomically on **`SaveChanges`**.
- It is **not thread-safe** and is **scoped/short-lived** (one per request/operation) — never share across threads or inject into singletons (use `IDbContextFactory` for parallelism/singletons — [10](10-DbContextFactory.md), [Ch03 §03](../03-HostingAndDI/03-Lifetimes.md)).
- Register with `AddDbContext` (or **`AddDbContextPool`** for high-throughput hot paths); EF opens/closes pooled connections per operation — you configure the connection string, not connections.
- The DI **scope disposes** the context (don't dispose injected ones); dispose contexts you create yourself.
- It's already a UoW + repository — wrapping it in another repository layer is usually redundant.

→ Next: [02-Querying.md](02-Querying.md)
