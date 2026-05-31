# DbContextFactory

## When the scoped DbContext doesn't fit

The default `AddDbContext` gives you a **scoped** `DbContext` — one per request, injected into your services. That's perfect for typical request/response apps. But some scenarios don't have a clean "scope per operation" or need **multiple contexts at once** — and there, injecting a single scoped context breaks (it's not thread-safe — [01-DbContext.md](01-DbContext.md)). For these, use **`IDbContextFactory<T>`**: you create (and own) a context on demand.

```csharp
builder.Services.AddDbContextFactory<AppDbContext>(o => o.UseNpgsql(cs));

public class ReportService(IDbContextFactory<AppDbContext> factory) {
    public async Task<Report> BuildAsync(CancellationToken ct) {
        await using var db = await factory.CreateDbContextAsync(ct);   // a fresh context you own
        // ... use db ...
        // disposed here
    }
}
```

`CreateDbContextAsync` returns a brand-new context each call; you `await using` it (you own its lifetime, unlike the DI-managed scoped one).

---

## The three scenarios that need a factory

### 1. Parallel queries

A single `DbContext` can't run queries concurrently (not thread-safe). To fan out parallel DB work, use **a context per task**:

```csharp
// ✗ — sharing one context across parallel tasks → "second operation started on this context"
await Task.WhenAll(ids.Select(id => sharedDb.Products.FindAsync(id).AsTask()));

// ✓ — a context per parallel operation, via the factory
var results = await Task.WhenAll(ids.Select(async id => {
    await using var db = await factory.CreateDbContextAsync(ct);
    return await db.Products.FindAsync([id], ct);
}));
```

Each task gets its own context (and its own connection from the pool), so they run safely in parallel.

### 2. Blazor Server

Blazor Server components are **long-lived** (a component instance lives for the user's session/circuit, far longer than a "request"), and a user may trigger overlapping operations. A scoped `DbContext` shared by a component would be long-lived *and* potentially used concurrently — both wrong. Blazor Server apps use **`IDbContextFactory`**, creating a short-lived context per operation:

```csharp
// Blazor component
@inject IDbContextFactory<AppDbContext> Factory

private async Task LoadAsync() {
    await using var db = await Factory.CreateDbContextAsync();
    products = await db.Products.AsNoTracking().ToListAsync();
}
```

This is the **recommended EF pattern for Blazor Server** ([Ch14](../14-Blazor/README.md)) — a fresh, short-lived context per data operation, not a long-lived injected one.

### 3. Singletons & background services

A singleton or `BackgroundService` ([Ch03 §01](../03-HostingAndDI/01-GenericHost.md)) can't inject a scoped `DbContext` (captive dependency — [Ch03 §03](../03-HostingAndDI/03-Lifetimes.md)). Two options: create a DI scope per work item (`IServiceScopeFactory`), or inject `IDbContextFactory` and create a context per operation:

```csharp
public class CleanupWorker(IDbContextFactory<AppDbContext> factory) : BackgroundService {
    protected override async Task ExecuteAsync(CancellationToken ct) {
        using var timer = new PeriodicTimer(TimeSpan.FromHours(1));
        while (await timer.WaitForNextTickAsync(ct)) {
            await using var db = await factory.CreateDbContextAsync(ct);   // fresh per cycle
            await db.Orders.Where(o => o.Created < DateTime.UtcNow.AddYears(-1))
                .ExecuteDeleteAsync(ct);
        }
    }
}
```

The factory is the cleaner choice when you specifically need a `DbContext` (vs. a full scope with other scoped services).

---

## Factory vs scope-per-work-item

In a singleton/background service you have two ways to get a correctly-scoped context:

```csharp
// Option A: IServiceScopeFactory — when you need OTHER scoped services too
await using var scope = scopeFactory.CreateAsyncScope();
var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
var validator = scope.ServiceProvider.GetRequiredService<IValidator>();   // also scoped

// Option B: IDbContextFactory — when you ONLY need a DbContext
await using var db = await contextFactory.CreateDbContextAsync(ct);
```

- **`IServiceScopeFactory`** — creates a full DI scope; use when the operation needs several scoped services (the context *and* a scoped validator/repository).
- **`IDbContextFactory`** — creates just a context; cleaner when the `DbContext` is all you need, and the default for parallel/Blazor scenarios.

---

## Pooling with the factory

`AddPooledDbContextFactory` combines the factory pattern with **context pooling** (reuse instances — [01-DbContext.md](01-DbContext.md)) for high-throughput scenarios:

```csharp
builder.Services.AddPooledDbContextFactory<AppDbContext>(o => o.UseNpgsql(cs), poolSize: 128);
// CreateDbContextAsync now rents from the pool and returns on dispose
```

Each `CreateDbContextAsync` rents a context from the pool; disposing returns it. This cuts context-creation overhead when you're creating many short-lived contexts (e.g., a high-traffic Blazor Server app or a busy worker). Same caveat as pooling generally: don't store per-operation state in the pooled context.

---

## You own the lifetime

The crucial difference from injected scoped contexts: with the factory, **you create the context and you dispose it** (`await using`). The DI container doesn't manage it. So:
- Always `await using` the created context (or dispose it) — leaking it leaks a connection.
- Keep it short-lived (one operation), like any context.
- Don't capture it in a field for reuse across operations (defeats the purpose; reintroduces thread-safety/lifetime issues).

---

## Common gotchas

### Sharing one context across parallel tasks

The error that drives people to the factory: "A second operation was started on this context." A `DbContext` is single-threaded — use a context per parallel task via the factory.

### Scoped context in Blazor Server

A scoped context in Blazor Server is long-lived (per circuit) and can be used concurrently — both wrong. Use `IDbContextFactory` with short-lived per-operation contexts.

### Scoped context in a singleton/background service

Captive dependency. Use `IDbContextFactory` (or `IServiceScopeFactory` if you need other scoped services).

### Forgetting to dispose factory-created contexts

You own them — `await using` (or dispose). A leaked context holds a pooled connection.

### Holding a factory-created context as a field

Reusing one across operations reintroduces the long-lived/thread-safety problems. Create per operation, dispose, repeat.

---

## Summary

- **`IDbContextFactory<T>`** (`AddDbContextFactory`) creates **fresh, short-lived contexts on demand** that **you own and dispose** (`await using`) — for scenarios the default scoped context can't serve.
- Use it for **parallel queries** (a context per task — `DbContext` isn't thread-safe), **Blazor Server** (components are long-lived; create a context per operation — the recommended pattern), and **singletons/background services** (avoid the captive-dependency bug).
- Choose **`IServiceScopeFactory`** when you need other scoped services too; **`IDbContextFactory`** when you only need a context.
- **`AddPooledDbContextFactory`** adds context pooling for high-throughput short-lived-context workloads.
- You own the lifetime: always dispose, keep it short-lived, don't hold it in a field.

→ Next: [11-Interceptors.md](11-Interceptors.md)
