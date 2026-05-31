# The Generic Host

## The universal application model

The **Generic Host** is the shared skeleton of every modern .NET application — web APIs, console workers, gRPC services, desktop apps. It wires together **dependency injection**, **configuration**, **logging**, and **lifetime management** into one consistent object, so the machinery is the same whether you're building Kestrel or a background daemon. Learn it once; it applies everywhere.

```csharp
var builder = Host.CreateApplicationBuilder(args);   // the modern entry point (.NET 8+)

builder.Services.AddSingleton<IClock, SystemClock>();   // register services (DI)
builder.Services.AddHostedService<Worker>();            // a long-running background service
// builder.Configuration already loaded (appsettings, env, args)
// builder.Logging already set up

IHost host = builder.Build();
await host.RunAsync();                                  // start, run until shutdown, dispose
```

That `builder` exposes `Services` (DI), `Configuration`, `Logging`, and `Environment` — the four pillars. `Build()` produces an `IHost`; `RunAsync()` starts it, runs hosted services, and blocks until shutdown.

---

## Why a host exists

Before the Generic Host, console apps, web apps, and services each had their own ad-hoc startup. The host unifies them: **the same DI container, config system, and logging** power all app types. A web app is just `WebApplication.CreateBuilder` (a host specialized for HTTP — [Ch04](../04-AspNetCore/README.md)); a worker is `Host.CreateApplicationBuilder`. The skills transfer completely.

The host also provides:
- **Composition root** — one place to register all services ([02-DependencyInjection.md](02-DependencyInjection.md)).
- **Graceful startup/shutdown** — ordered start, ordered stop, cancellation on Ctrl+C/SIGTERM.
- **Configuration layering** — appsettings, env vars, secrets, command line, merged with precedence ([07-Configuration.md](07-Configuration.md)).
- **Logging** — `ILogger` configured and injectable everywhere ([09-LoggingPrimer.md](09-LoggingPrimer.md)).

---

## `CreateApplicationBuilder` vs the older `CreateDefaultBuilder`

```csharp
// Modern (.NET 8+) — the streamlined builder; properties exposed directly
var builder = Host.CreateApplicationBuilder(args);
builder.Services.Add...;
builder.Configuration.Add...;
var host = builder.Build();

// Older pattern (still valid) — callback-based
var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((ctx, services) => { services.Add...; })
    .Build();
```

`Host.CreateApplicationBuilder` (and `WebApplication.CreateBuilder`) is the modern, flatter API — you access `Services`/`Configuration`/`Logging` as properties rather than nesting callbacks. Both produce the same host; new code uses the builder form. It pre-configures sensible defaults (config sources, console logging, DI, environment).

---

## The host lifecycle

```
Build()        → container built, config loaded, services registered
   ↓
StartAsync()   → IHostedService.StartAsync called for EACH hosted service, in registration order
   ↓
[ running ]    → hosted services do their work; app stays alive
   ↓
shutdown signal (Ctrl+C / SIGTERM / StopApplication)
   ↓
StopAsync()    → IHostedService.StopAsync called in REVERSE order, with a shutdown timeout
   ↓
Dispose        → scopes/services disposed
```

`RunAsync()` = `StartAsync()` + wait for shutdown + `StopAsync()` + dispose. Hosted services **start in registration order and stop in reverse** — so dependencies start before dependents and shut down after them.

---

## `IHostedService` and `BackgroundService`

A **hosted service** is a component whose lifetime the host manages — started on `StartAsync`, stopped on `StopAsync`. This is how you run background work (queues, timers, message consumers) inside any host.

```csharp
// IHostedService — the raw interface (for start/stop hooks)
public class StartupTask(IServiceProvider sp) : IHostedService {
    public async Task StartAsync(CancellationToken ct) { /* warm caches, run migrations */ }
    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}

// BackgroundService — the base class for a long-running loop (the common case)
public class Worker(ILogger<Worker> logger) : BackgroundService {
    protected override async Task ExecuteAsync(CancellationToken stoppingToken) {
        while (!stoppingToken.IsCancellationRequested) {
            logger.LogInformation("working at {Time}", DateTimeOffset.UtcNow);
            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }
}

builder.Services.AddHostedService<Worker>();
```

`BackgroundService` wraps `IHostedService` with an `ExecuteAsync(stoppingToken)` long-running loop — the standard base for background work. The `stoppingToken` is signaled on shutdown; honor it to stop promptly. This chapter introduces it; **Chapter 08 (Background Processing)** covers hosted services, queued work, scheduling, and reliability in depth.

> Critical: `StartAsync` blocks the host's startup until it returns. Long work belongs in `ExecuteAsync` (which runs *after* startup), not in `StartAsync` — a slow `StartAsync` delays the whole app coming up. And an unhandled exception in `ExecuteAsync` stops the host by default (.NET 6+ `BackgroundServiceExceptionBehavior`).

---

## Graceful shutdown

The host listens for shutdown signals (Ctrl+C, SIGTERM from a container orchestrator) and triggers ordered shutdown. You can hook the lifetime:

```csharp
public class GracefulWorker(IHostApplicationLifetime lifetime, ILogger<GracefulWorker> log)
    : BackgroundService {
    protected override async Task ExecuteAsync(CancellationToken ct) {
        lifetime.ApplicationStopping.Register(() => log.LogInformation("draining..."));
        // ... work, honoring ct ...
        // To request shutdown from code:
        // lifetime.StopApplication();
    }
}
```

`IHostApplicationLifetime` exposes `ApplicationStarted`, `ApplicationStopping`, `ApplicationStopped` tokens and `StopApplication()`. The host gives hosted services a **shutdown timeout** (default 30s, configurable via `HostOptions.ShutdownTimeout`) to finish — honor cancellation so you drain in-flight work cleanly (vital for zero-downtime deploys and Kubernetes pod termination). Covered fully in [Ch08 §08](../08-BackgroundProcessing/README.md).

---

## Environments

```csharp
if (builder.Environment.IsDevelopment()) { /* dev-only setup */ }
builder.Environment.EnvironmentName;     // "Development" / "Staging" / "Production"
builder.Environment.ContentRootPath;
```

The host reads `DOTNET_ENVIRONMENT` (or `ASPNETCORE_ENVIRONMENT` for web) to set the environment, which drives config layering (`appsettings.{Environment}.json` overrides `appsettings.json`) and conditional behavior. Default is `Production` if unset.

---

## A complete worker

```csharp
var builder = Host.CreateApplicationBuilder(args);

builder.Services.AddHttpClient();                          // typed HttpClient via factory
builder.Services.Configure<WorkerOptions>(builder.Configuration.GetSection("Worker"));
builder.Services.AddSingleton<IProcessor, Processor>();
builder.Services.AddHostedService<PollingWorker>();

var host = builder.Build();
await host.RunAsync();                                      // runs until SIGTERM/Ctrl+C
```

```csharp
public class PollingWorker(IProcessor processor, IOptions<WorkerOptions> options,
    ILogger<PollingWorker> logger) : BackgroundService {
    protected override async Task ExecuteAsync(CancellationToken ct) {
        using var timer = new PeriodicTimer(options.Value.Interval);
        while (await timer.WaitForNextTickAsync(ct))
            await processor.RunOnceAsync(ct);
    }
}
```

This is the canonical .NET worker: host + DI + config + options + a `BackgroundService` with a `PeriodicTimer`. The same pattern scales from a tiny daemon to a fleet of microservices.

---

## Common gotchas

### Heavy work in `StartAsync`

Blocks host startup (the app isn't "up" until all `StartAsync` complete). Put long-running work in `BackgroundService.ExecuteAsync`, which runs after startup.

### Not honoring the stopping token

A `BackgroundService` that ignores `stoppingToken` won't shut down promptly, causing forced termination and dropped work. Pass and check it everywhere.

### Forgetting `AddHostedService` registers a singleton

Hosted services are singletons. To use scoped services (e.g., a `DbContext`) inside one, **create a scope per work item** (`IServiceScopeFactory`) — see [03-Lifetimes.md](03-Lifetimes.md). Injecting a scoped service directly into a hosted service is a captive-dependency bug.

### Exceptions in `ExecuteAsync`

By default (.NET 6+) an unhandled exception in `ExecuteAsync` **stops the host**. Wrap your loop body in try/catch (and log) if a single iteration's failure shouldn't kill the app.

### Confusing the host with ASP.NET Core

`WebApplication.CreateBuilder` *is* a Generic Host specialized for HTTP. The DI/config/logging APIs are identical — don't relearn them per app type.

---

## Summary

- The **Generic Host** (`Host.CreateApplicationBuilder`) is the universal app model — DI, configuration, logging, and lifetime — shared by web apps, workers, and services.
- `builder` exposes **`Services`, `Configuration`, `Logging`, `Environment`**; `Build()` → `IHost`; `RunAsync()` starts, runs, and shuts down gracefully.
- **Hosted services** start in registration order, stop in **reverse**; **`BackgroundService.ExecuteAsync(stoppingToken)`** is the standard long-running-loop base (honor the token).
- Keep `StartAsync` fast (it blocks startup); handle `ExecuteAsync` exceptions; create a **scope per work item** to use scoped services in a (singleton) hosted service.
- **`IHostApplicationLifetime`** + a shutdown timeout enable graceful drain — essential for zero-downtime deploys. Background work in depth: [Chapter 08](../08-BackgroundProcessing/README.md).

→ Next: [02-DependencyInjection.md](02-DependencyInjection.md)
