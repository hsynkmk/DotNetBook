# Hosted Services

## The lifecycle hook for background work

`IHostedService` is the interface the Generic Host uses to start and stop components alongside the application. Anything that needs to run *outside* a request — a queue consumer, a cache warmer, a scheduled job, a startup migration — is a hosted service. The host calls `StartAsync` on each at startup and `StopAsync` at shutdown.

```csharp
public class StartupTasks(IServiceProvider services, ILogger<StartupTasks> log) : IHostedService {
    public async Task StartAsync(CancellationToken ct) {
        log.LogInformation("Warming caches...");
        await using var scope = services.CreateAsyncScope();
        // run migrations, warm caches, validate config, etc.
    }
    public Task StopAsync(CancellationToken ct) {
        log.LogInformation("Shutting down");
        return Task.CompletedTask;
    }
}

builder.Services.AddHostedService<StartupTasks>();
```

> Introduced in [Ch03 §01](../03-HostingAndDI/01-GenericHost.md); this chapter goes deep on background work patterns. `IHostedService` is the foundation everything else here builds on.

---

## `StartAsync` / `StopAsync` semantics

The two methods have precise, easy-to-misuse semantics:

```csharp
public Task StartAsync(CancellationToken ct);   // called at host startup, in registration order
public Task StopAsync(CancellationToken ct);    // called at shutdown, in REVERSE registration order
```

- **`StartAsync` blocks host startup until it returns.** The app isn't "started" (won't serve requests, for a web app) until *every* hosted service's `StartAsync` completes. So `StartAsync` must be **fast** — kick off background work, don't *do* long-running work here.
- **`StopAsync` is called on shutdown** with a cancellation token tied to the shutdown timeout. Use it to stop gracefully (drain, flush, dispose).
- **Order**: services start in **registration order** and stop in **reverse** — so dependencies start before dependents and shut down after them.

```csharp
// ✗ — long work in StartAsync BLOCKS the app from starting
public async Task StartAsync(CancellationToken ct) {
    while (true) { await PollAsync(ct); }   // never returns → host never finishes starting!
}

// ✓ — StartAsync returns quickly; long work runs separately (see BackgroundService, §02)
```

A long-running loop belongs in a background task started by `StartAsync` (or, far better, use `BackgroundService` — [02-BackgroundService.md](02-BackgroundService.md), which handles this correctly).

---

## When to use raw `IHostedService`

`BackgroundService` ([02](02-BackgroundService.md)) is the right base for long-running loops. Use raw `IHostedService` when you need **explicit start/stop hooks** rather than a continuous loop:

- **One-time startup work** — run database migrations, warm caches, validate configuration, register with a service registry. Do it in `StartAsync` (kept fast) and nothing in `StopAsync`.
- **Resource setup/teardown** — open a connection pool / subscription on start, close it on stop.
- **Wrapping a third-party component** whose lifecycle you must tie to the host's.

For a loop that runs continuously (poll, consume, schedule), use `BackgroundService`.

---

## Startup migration — a common (and subtle) example

A frequent use is running EF Core migrations at startup. It's a good `IHostedService` example *and* a cautionary tale:

```csharp
public class MigrationHostedService(IServiceProvider services) : IHostedService {
    public async Task StartAsync(CancellationToken ct) {
        await using var scope = services.CreateAsyncScope();      // scope to resolve scoped DbContext
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync(ct);
    }
    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}
```

This works for a **single instance**, but auto-migrating on startup across **multiple instances** is dangerous (instances race to apply migrations, and a bad migration takes down all instances) — see [Ch05 §05](../05-EFCore/05-Migrations.md). For production, prefer applying migrations as a controlled deploy step. The pattern illustrates `StartAsync` for startup work, but choose where you run migrations carefully.

Note the **scope-per-operation**: a hosted service is a singleton, so it can't inject a scoped `DbContext` — create a scope (`CreateAsyncScope`) and resolve from it ([03-WorkerServices.md](03-WorkerServices.md), [Ch03 §03](../03-HostingAndDI/03-Lifetimes.md)).

---

## Hosted services are singletons

`AddHostedService<T>` registers `T` as a **singleton** managed by the host. Consequences:
- It can inject other **singletons** directly, but **not scoped services** — create a scope per unit of work for those ([04-QueuedBackgroundWork.md](04-QueuedBackgroundWork.md)).
- It exists once per process for the app's lifetime.
- Multiple hosted services run concurrently (they're started in sequence but then run independently).

---

## Exception behavior

What happens if `StartAsync` throws? By default, **an exception in `StartAsync` stops the host** — the app fails to start. This is often desirable (fail-fast on a broken startup task), but means a flaky startup task can prevent the whole app from launching. Handle expected failures within `StartAsync` if the app should start anyway:

```csharp
public async Task StartAsync(CancellationToken ct) {
    try { await WarmCacheAsync(ct); }
    catch (Exception ex) { log.LogWarning(ex, "Cache warm failed; continuing"); }  // non-fatal
}
```

(Exceptions in `BackgroundService.ExecuteAsync` have their own behavior — [02-BackgroundService.md](02-BackgroundService.md).)

---

## Common gotchas

### Long-running work in `StartAsync`

It blocks the host from finishing startup (the app never becomes ready). Kick off background work and return; use `BackgroundService` for loops.

### Injecting scoped services into a hosted service

It's a singleton — injecting a scoped `DbContext` is a captive dependency. Create a scope per operation (`CreateAsyncScope`).

### Auto-migrating across multiple instances

Instances race; a bad migration kills them all. Apply migrations as a controlled deploy step, not in `StartAsync` across replicas ([Ch05 §05](../05-EFCore/05-Migrations.md)).

### Not handling `StartAsync` exceptions

An unhandled exception stops the host (app fails to start). Fine for fail-fast; wrap in try/catch if the app should start regardless.

### Slow `StopAsync` exceeding the shutdown timeout

`StopAsync` gets a bounded shutdown window (default 30s); exceeding it means forced termination (dropped work). Keep shutdown prompt or extend the timeout deliberately ([08-ReliabilityAndScale.md](08-ReliabilityAndScale.md)).

---

## Summary

- **`IHostedService`** (`StartAsync`/`StopAsync`) is the host's hook for components that run outside requests; services **start in registration order, stop in reverse**.
- **`StartAsync` blocks startup until it returns** — keep it fast (kick off work, don't do long-running work there); long loops belong in **`BackgroundService`** ([02](02-BackgroundService.md)).
- Use raw `IHostedService` for **explicit start/stop hooks** (one-time startup work, resource setup/teardown); it's a **singleton** (create a scope for scoped services).
- An exception in `StartAsync` **stops the host** (fail-fast) — handle it if the app should start anyway.
- Startup migrations illustrate the pattern but need care across multiple instances (apply as a controlled deploy step instead — [Ch05 §05](../05-EFCore/05-Migrations.md)).

→ Next: [02-BackgroundService.md](02-BackgroundService.md)
