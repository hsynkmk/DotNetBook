# Service Lifetimes

## The three lifetimes

A lifetime answers "how many instances, and how long do they live?" Choosing wrong causes either subtle state-sharing bugs or the dreaded **captive dependency**. There are three:

| Lifetime | One instance per | Lives until | Use for |
|---|---|---|---|
| **Singleton** | the whole app | app shutdown | shared, **thread-safe**, stateless-or-immutable state (caches, config, clients) |
| **Scoped** | a scope (e.g., one web request) | scope disposed | per-request state (`DbContext`, unit of work) |
| **Transient** | every resolution | tracked by creating scope | lightweight, stateless services |

```csharp
services.AddSingleton<IClock, SystemClock>();        // one for the app
services.AddScoped<IOrderRepository, SqlRepository>(); // one per request
services.AddTransient<IEmailBuilder, EmailBuilder>();  // a fresh one each time
```

---

## What a "scope" is

A **scope** is a bounded region with its own set of scoped instances. In ASP.NET Core, the framework creates **one scope per HTTP request** — so a `Scoped` service is effectively "one per request." When the request ends, the scope is disposed (disposing its scoped + transient-disposable services).

```csharp
// You create scopes manually outside web requests (e.g., in a worker):
using IServiceScope scope = serviceProvider.CreateScope();
var repo = scope.ServiceProvider.GetRequiredService<IOrderRepository>();  // a scoped instance
// ... use repo ...
// scope disposed here → repo (and any scoped/transient-disposables) disposed
```

The root provider (the host) is itself the "root scope" for singletons. Resolving a **scoped** service from the **root** provider is an error in development (the container detects it) — scoped services need a real scope.

---

## The captive dependency — the #1 DI bug

**A service may only depend on services with an equal or longer lifetime.** Violating this "captures" a shorter-lived service inside a longer-lived one, freezing it at the wrong lifetime:

```
Singleton  (longest)
   ↑ may depend on
Scoped
   ↑ may depend on
Transient  (shortest)
```

```csharp
// ✗ — Singleton capturing a Scoped DbContext: the DbContext lives FOREVER (app lifetime),
//      shared across all requests, breaking per-request semantics AND thread-safety.
public class CacheService(AppDbContext db) { }          // db is Scoped
services.AddSingleton<CacheService>();                   // captures the scoped db → BUG
```

Symptoms of a captive dependency:
- A `DbContext` (scoped, **not thread-safe**) shared across concurrent requests → corruption, "a second operation started on this context" exceptions.
- "Stale" per-request state leaking between requests.
- Disposed-object errors (the captured service was disposed but the singleton still holds it).

The container's **scope validation** (on by default in Development) catches many of these at `Build()`/resolution time, throwing "Cannot consume scoped service X from singleton Y."

```csharp
// Enable scope validation explicitly (it's on in Development by default):
var host = builder.Build();   // CreateApplicationBuilder validates scopes in Development
```

---

## Resolving shorter-lived services correctly

When a singleton genuinely needs a scoped service (common in hosted services and background workers), **create a scope per unit of work** and resolve from it:

```csharp
public class QueueWorker(IServiceProvider services, ILogger<QueueWorker> log) : BackgroundService {
    protected override async Task ExecuteAsync(CancellationToken ct) {
        while (!ct.IsCancellationRequested) {
            var item = await DequeueAsync(ct);

            // A fresh scope per work item — like a "request" for background work
            await using AsyncServiceScope scope = services.CreateAsyncScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();   // scoped, correct
            await Process(item, db, ct);
            // scope disposed → db disposed, exactly like end-of-request
        }
    }
}
```

`IServiceScopeFactory` / `IServiceProvider.CreateScope()` / `CreateAsyncScope()` give you a scope on demand. This is the **correct pattern** for using `DbContext` (and other scoped services) inside a singleton hosted service — covered again in [Ch08 Background Processing](../08-BackgroundProcessing/README.md). Inject `IServiceProvider`/`IServiceScopeFactory` here (a legitimate framework-edge use), not the scoped service directly.

---

## Choosing a lifetime — decision guide

```
Is it stateless or immutable, and thread-safe?
   YES → Singleton (one instance, cheapest). e.g., IClock, HttpClient (via factory), config, caches.
   NO  → does it hold per-request/per-operation state?
            YES → Scoped. e.g., DbContext, unit of work, the current user's request context.
            NO  → Transient (lightweight, no shared state). e.g., a stateless helper/builder.
```

- **Default to Singleton** for stateless services — fewer allocations, no per-request churn — but only if thread-safe (singletons are accessed concurrently!).
- **Scoped** for anything carrying per-request state, especially `DbContext` (EF Core registers it scoped — [Ch05](../05-EFCore/README.md)).
- **Transient** when you want a fresh, isolated instance each time and there's no shared state to worry about.

---

## Thread-safety and singletons

A **singleton is accessed by many threads concurrently** (every request, every background task). So singleton services **must be thread-safe** for their mutable state:

```csharp
// ✗ — singleton with unguarded mutable state → race conditions across requests
public class Counter { public int Count; public void Inc() => Count++; }   // not thread-safe!
services.AddSingleton<Counter>();

// ✓ — thread-safe singleton
public class Counter {
    private int _count;
    public void Inc() => Interlocked.Increment(ref _count);
    public int Count => Volatile.Read(ref _count);
}
```

If a singleton has mutable state, protect it (`Interlocked`, `lock`, concurrent collections — [Ch02 §12](../02-BCL/12-Threading.md)). Stateless/immutable singletons are inherently safe. Scoped/transient services are usually used on one thread at a time (one request), so they typically don't need synchronization.

---

## Transient subtleties

- **Transient services injected into a singleton are captured** for the singleton's lifetime (a transient becomes a de-facto singleton). The captive-dependency rule applies to transients too.
- **Transient `IDisposable`s are disposed by the scope/provider that created them** — resolve them in a scope, not from the root (else they live until app shutdown — [02-DependencyInjection.md](02-DependencyInjection.md)).

---

## Common gotchas

### Singleton → Scoped captive dependency

The classic bug — a `DbContext` (scoped, not thread-safe) trapped in a singleton, shared across concurrent requests → corruption. Use `CreateScope()` per work item. Scope validation catches it in Development.

### Disabling scope validation

Don't disable the development scope validation to "make the error go away" — it's catching a real bug. Fix the lifetime mismatch.

### Mutable singleton state without synchronization

Singletons are concurrent. Unguarded mutable fields race. Make singletons thread-safe or stateless.

### Resolving scoped from the root provider

Throws (scoped needs a scope). Create a scope.

### Transient disposables from root

Leak until shutdown (rooted by the root provider). Resolve disposables within a scope.

### Over-using Scoped where Singleton would do

A stateless service registered scoped allocates one per request needlessly. If it's thread-safe and stateless, make it a singleton.

---

## Summary

- Three lifetimes: **Singleton** (one per app — must be thread-safe), **Scoped** (one per scope/request — for per-request state like `DbContext`), **Transient** (one per resolution).
- A **scope** bounds scoped instances (one per HTTP request automatically); create scopes manually with `CreateScope()`/`CreateAsyncScope()`.
- The **captive dependency** is the #1 DI bug: never let a longer-lived service depend on a shorter-lived one (Singleton → Scoped traps the scoped service forever, breaking per-request semantics and thread-safety). Scope validation catches it in Development.
- To use a scoped service in a singleton/hosted service, **create a scope per unit of work** and resolve from it.
- **Singletons are concurrent** — protect mutable state; default to Singleton for stateless services, Scoped for stateful-per-request, Transient for fresh isolated instances.

→ Next: [04-OpenGenericsInDI.md](04-OpenGenericsInDI.md)
