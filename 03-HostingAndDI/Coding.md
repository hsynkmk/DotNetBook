# Chapter 03 — Hosting & DI — Coding Problems

Build the application backbone: a host, registrations with correct lifetimes, layered config, the Options pattern with validation, and a background worker that uses scoped services correctly.

---

## Problem 1: A complete console worker

Build a worker host with logging, config, DI, and a `BackgroundService`.

<details><summary>Solution</summary>

```csharp
var builder = Host.CreateApplicationBuilder(args);
builder.Services.Configure<WorkerOptions>(builder.Configuration.GetSection("Worker"));
builder.Services.AddSingleton<IProcessor, Processor>();
builder.Services.AddHostedService<PollingWorker>();

var host = builder.Build();
await host.RunAsync();

public class PollingWorker(IProcessor processor, IOptions<WorkerOptions> options,
    ILogger<PollingWorker> logger) : BackgroundService {
    protected override async Task ExecuteAsync(CancellationToken ct) {
        using var timer = new PeriodicTimer(options.Value.Interval);
        while (await timer.WaitForNextTickAsync(ct)) {
            try { await processor.RunOnceAsync(ct); }
            catch (Exception ex) { logger.LogError(ex, "iteration failed"); }   // don't let one failure kill the host
        }
    }
}

public class WorkerOptions { public TimeSpan Interval { get; set; } = TimeSpan.FromSeconds(30); }
```

`PeriodicTimer` prevents overlapping ticks; the try/catch keeps a single iteration's failure from stopping the host; `stoppingToken` ensures graceful shutdown.

</details>

---

## Problem 2: Register services with correct lifetimes

Given a stateless clock, a per-request `DbContext`-like unit of work, and a thread-safe singleton cache, register them correctly.

<details><summary>Solution</summary>

```csharp
services.AddSingleton<IClock, SystemClock>();          // stateless + thread-safe → singleton
services.AddScoped<IUnitOfWork, EfUnitOfWork>();        // per-request state → scoped
services.AddSingleton<ICache, ConcurrentCache>();       // shared, THREAD-SAFE → singleton

// The singleton cache MUST be thread-safe (accessed concurrently):
public class ConcurrentCache : ICache {
    private readonly ConcurrentDictionary<string, object> _store = new();
    public object? Get(string k) => _store.GetValueOrDefault(k);
    public void Set(string k, object v) => _store[k] = v;
}
```

Stateless+thread-safe → Singleton (cheapest). Per-request state → Scoped. A singleton cache must use a concurrent collection (it's hit by many threads). ([03-Lifetimes.md](03-Lifetimes.md).)

</details>

---

## Problem 3: Spot and fix a captive dependency

This throws "Cannot consume scoped service from singleton." Fix it.

```csharp
services.AddScoped<AppDbContext>();
services.AddSingleton<ReportCache>();   // ReportCache(AppDbContext db) — BUG
```

<details><summary>Solution</summary>

```csharp
// ReportCache is a singleton that needs the scoped DbContext per operation.
// Inject IServiceScopeFactory and create a scope per use:
public class ReportCache(IServiceScopeFactory scopeFactory) {
    public async Task<Report> BuildAsync(int id) {
        await using var scope = scopeFactory.CreateAsyncScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();   // scoped, correct
        return await BuildFromAsync(db, id);
        // scope disposed → db disposed
    }
}
services.AddScoped<AppDbContext>();
services.AddSingleton<ReportCache>();
```

A singleton can't hold a scoped `DbContext` (not thread-safe, wrong lifetime). Create a scope per operation and resolve the scoped service from it. ([03-Lifetimes.md](03-Lifetimes.md).)

</details>

---

## Problem 4: Open generic repository

Register a generic repository once for all entities, then override one entity's repo.

<details><summary>Solution</summary>

```csharp
public interface IRepository<T> where T : class { T? Get(int id); }
public class EfRepository<T>(AppDbContext db) : IRepository<T> where T : class {
    public T? Get(int id) => db.Set<T>().Find(id);
}
public class AuditLogRepository(AppDbContext db) : IRepository<AuditLog> {
    public AuditLog? Get(int id) => /* custom: maybe read-replica */;
}

services.AddScoped(typeof(IRepository<>), typeof(EfRepository<>));   // default for all T
services.AddScoped<IRepository<AuditLog>, AuditLogRepository>();      // override for AuditLog

// IRepository<Order> → EfRepository<Order>;  IRepository<AuditLog> → AuditLogRepository
```

One open registration covers every entity; a closed registration special-cases `AuditLog`. ([04-OpenGenericsInDI.md](04-OpenGenericsInDI.md).)

</details>

---

## Problem 5: Keyed services dispatcher

Register email/SMS/push notifiers and dispatch by a runtime channel string.

<details><summary>Solution</summary>

```csharp
services.AddKeyedSingleton<INotifier, EmailNotifier>("email");
services.AddKeyedSingleton<INotifier, SmsNotifier>("sms");
services.AddKeyedSingleton<INotifier, PushNotifier>("push");

public class NotificationDispatcher(IServiceProvider services) {
    public Task SendAsync(string channel, Message msg) {
        var notifier = services.GetRequiredKeyedService<INotifier>(channel);  // by runtime key
        return notifier.SendAsync(msg);
    }
}

// For a compile-time-known channel, inject directly:
public class WelcomeService([FromKeyedServices("email")] INotifier email) { }
```

Keyed services select one of N variants by name. Resolving by a *runtime* key in a focused dispatcher is fine; for a fixed key use `[FromKeyedServices]`. ([05-Keyed.md](05-Keyed.md).)

</details>

---

## Problem 6: Decorate a repository with caching + logging

Wrap a SQL repository so calls are logged and reads are cached, without touching the SQL class.

<details><summary>Solution</summary>

```csharp
// (using Scrutor)
services.AddScoped<IProductRepository, SqlProductRepository>();
services.Decorate<IProductRepository, CachingProductRepository>();
services.Decorate<IProductRepository, LoggingProductRepository>();
// Resolution: Logging( Caching( Sql ) ) — logs every call, cache misses hit SQL

public class CachingProductRepository(IProductRepository inner, IMemoryCache cache) : IProductRepository {
    public Product? Get(int id) => cache.GetOrCreate(id, _ => inner.Get(id));
}
public class LoggingProductRepository(IProductRepository inner, ILogger<IProductRepository> log) : IProductRepository {
    public Product? Get(int id) { log.LogDebug("Get {Id}", id); return inner.Get(id); }
}
```

Order matters: last `Decorate` is outermost. Here Logging wraps Caching wraps Sql (logs all calls). Reverse the two `Decorate` calls to log only cache misses. ([06-Decorate.md](06-Decorate.md).)

</details>

---

## Problem 7: Layer configuration from multiple sources

Set up config so an env var overrides appsettings, with a typed connection string.

<details><summary>Solution</summary>

```csharp
var builder = Host.CreateApplicationBuilder(args);
// Defaults already loaded: appsettings.json → appsettings.{Env}.json → user-secrets → env → args
builder.Configuration.AddJsonFile("custom.json", optional: true, reloadOnChange: true);

var cs = builder.Configuration.GetConnectionString("Default");
// appsettings.json: { "ConnectionStrings": { "Default": "Server=localhost;..." } }
// Override in production via env var:
//   export ConnectionStrings__Default="Server=prod;..."   (note __ for hierarchy)
```

Later providers win, so the env var overrides the JSON. `__` is the env-var hierarchy separator. Secrets come from env/Key Vault, never `appsettings.json`. ([07-Configuration.md](07-Configuration.md).)

</details>

---

## Problem 8: Options pattern with startup validation

Bind SMTP settings, validate them, and fail at startup if misconfigured.

<details><summary>Solution</summary>

```csharp
public class SmtpOptions {
    [Required] public string Host { get; set; } = "";
    [Range(1, 65535)] public int Port { get; set; }
    [Required, EmailAddress] public string From { get; set; } = "";
}

builder.Services.AddOptions<SmtpOptions>()
    .Bind(builder.Configuration.GetSection("Smtp"))
    .ValidateDataAnnotations()
    .Validate(o => o.Port != 25 || !o.UseSsl, "Port 25 with SSL is suspicious")
    .ValidateOnStart();                          // ← host refuses to start if invalid

public class EmailSender(IOptions<SmtpOptions> options) {
    private readonly SmtpOptions _o = options.Value;   // typed, validated
}
```

`ValidateOnStart()` turns bad config into a clear startup failure (caught by CI/health checks) instead of a cryptic runtime error. ([08-Options.md](08-Options.md), [10-Validation.md](10-Validation.md).)

</details>

---

## Problem 9: Live-reloading options in a singleton

A singleton background service must pick up config changes at runtime. Which accessor, and how?

<details><summary>Solution</summary>

```csharp
builder.Configuration.AddJsonFile("appsettings.json", optional: false, reloadOnChange: true);
builder.Services.Configure<PollingOptions>(builder.Configuration.GetSection("Polling"));

public class PollingWorker(IOptionsMonitor<PollingOptions> monitor, ILogger<PollingWorker> log)
    : BackgroundService {
    protected override async Task ExecuteAsync(CancellationToken ct) {
        monitor.OnChange(o => log.LogInformation("config reloaded: interval={Interval}", o.Interval));
        while (!ct.IsCancellationRequested) {
            var interval = monitor.CurrentValue.Interval;   // always the latest value
            await Task.Delay(interval, ct);
            await PollAsync(ct);
        }
    }
}
```

`IOptionsMonitor<T>` (singleton, live `CurrentValue` + `OnChange`) is correct for a singleton/background service. `IOptionsSnapshot<T>` would be a captive dependency (it's scoped); `IOptions<T>` never reloads. ([08-Options.md](08-Options.md).)

</details>

---

## Problem 10: Structured logging with scope and source-gen

Log an operation with correlation context and an allocation-free hot-path message.

<details><summary>Solution</summary>

```csharp
public partial class OrderService(ILogger<OrderService> logger) {
    [LoggerMessage(Level = LogLevel.Information, Message = "Order {OrderId} placed for {Amount}")]
    private partial void LogOrderPlaced(int orderId, decimal amount);   // source-gen, no boxing

    public async Task PlaceAsync(Order order, string tenantId) {
        using (logger.BeginScope("Order {OrderId} for tenant {TenantId}", order.Id, tenantId)) {
            logger.LogInformation("validating");          // carries OrderId + TenantId
            await ValidateAsync(order);
            await SaveAsync(order);
            LogOrderPlaced(order.Id, order.Total);          // hot-path, structured, allocation-free
        }
    }
}
```

`BeginScope` attaches correlation context to all logs in the operation (flows across awaits); `[LoggerMessage]` gives a fast, structured, AOT-friendly log call. Message templates (not interpolation) keep fields queryable. ([09-LoggingPrimer.md](09-LoggingPrimer.md).)

</details>

---

You can now stand up a production-shaped host: correct lifetimes, scope-per-work-item for scoped services, open generics and keyed services, decoration, layered config, validated typed options, and structured logging. This backbone underlies every chapter that follows.

→ Back to [Chapter 03 README](README.md) · Next chapter: [Chapter 04 — ASP.NET Core](../04-AspNetCore/README.md)
