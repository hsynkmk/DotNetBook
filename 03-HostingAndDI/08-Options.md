# The Options Pattern

## Typed, injectable, validated configuration

The Options pattern binds a configuration section to a strongly-typed class and makes it injectable. Instead of reading stringly-typed `config["Worker:MaxRetries"]` everywhere, you bind once to a `WorkerOptions` POCO and inject it — typed, validated, testable.

```csharp
// 1. The options class — a plain POCO matching the config section
public class WorkerOptions {
    public int MaxRetries { get; set; } = 3;
    public TimeSpan Interval { get; set; } = TimeSpan.FromSeconds(30);
    public string QueueName { get; set; } = "";
}

// 2. Bind it to the "Worker" section (in the composition root)
builder.Services.Configure<WorkerOptions>(builder.Configuration.GetSection("Worker"));

// 3. Inject it
public class Worker(IOptions<WorkerOptions> options) {
    private readonly WorkerOptions _opts = options.Value;   // typed access
    public async Task Run() { for (int i = 0; i < _opts.MaxRetries; i++) { } }
}
```

```json
{ "Worker": { "MaxRetries": 5, "Interval": "00:00:30", "QueueName": "orders" } }
```

The section's keys map to the POCO's properties by name. This is the **recommended way** to consume configuration (over injecting `IConfiguration` — [07-Configuration.md](07-Configuration.md)).

---

## The three accessors — `IOptions`, `IOptionsSnapshot`, `IOptionsMonitor`

The key decision is *which accessor to inject*, because they differ in lifetime and reload behavior:

| Accessor | Lifetime | Reload behavior | Use for |
|---|---|---|---|
| `IOptions<T>` | **Singleton** | computed **once**, never changes | most cases; config that doesn't change at runtime |
| `IOptionsSnapshot<T>` | **Scoped** | recomputed **per scope/request** | per-request options; picks up reloads each request |
| `IOptionsMonitor<T>` | **Singleton** | **live** — current value + change notifications | singletons/background services needing live reload |

```csharp
// IOptions<T> — singleton, value fixed at first access (the common case)
public class A(IOptions<WorkerOptions> o) { var v = o.Value; }

// IOptionsSnapshot<T> — scoped, fresh per request (use in request-scoped services)
public class B(IOptionsSnapshot<WorkerOptions> o) { var v = o.Value; }   // re-read each request

// IOptionsMonitor<T> — singleton, current value + onChange callback (use in singletons/hosted services)
public class C(IOptionsMonitor<WorkerOptions> monitor) {
    public C(...) { monitor.OnChange(updated => Reconfigure(updated)); }
    void Use() { var v = monitor.CurrentValue; }   // always the latest
}
```

Rules of thumb:
- **`IOptions<T>`** — default; config read once at startup. **Don't inject `IOptionsSnapshot` into a singleton** (it's scoped → captive dependency).
- **`IOptionsSnapshot<T>`** — when you want per-request values that reflect reloads, in **scoped** services (controllers, request handlers).
- **`IOptionsMonitor<T>`** — when a **singleton** or **background service** needs the current value and to react to live config reloads (`reloadOnChange` + `OnChange`).

---

## Named options

When you need **multiple instances** of the same options type (e.g., per-tenant, per-client settings):

```csharp
builder.Services.Configure<EndpointOptions>("primary", config.GetSection("Endpoints:Primary"));
builder.Services.Configure<EndpointOptions>("replica", config.GetSection("Endpoints:Replica"));

public class Service(IOptionsMonitor<EndpointOptions> monitor) {
    void Use() {
        var primary = monitor.Get("primary");
        var replica = monitor.Get("replica");
    }
}
```

Named options let one type carry several configurations, retrieved by name via `IOptionsMonitor.Get(name)` / `IOptionsSnapshot.Get(name)`. Useful for multi-tenant or multi-endpoint scenarios.

---

## Configuring options in code

Beyond binding to config, you can configure/transform options imperatively, and post-configure (run after all other configuration):

```csharp
services.Configure<WorkerOptions>(config.GetSection("Worker"));     // bind from config
services.Configure<WorkerOptions>(o => o.MaxRetries = Math.Max(o.MaxRetries, 1));  // adjust
services.PostConfigure<WorkerOptions>(o => o.QueueName ??= "default");             // final pass

// Configure using other services (e.g., a value computed from another option)
services.AddOptions<WorkerOptions>()
    .Configure<IClock>((o, clock) => o.StartTime = clock.UtcNow);
```

`Configure` actions run in registration order; `PostConfigure` runs last (for defaults/overrides that must win). `AddOptions<T>().Configure<TDep>(...)` lets configuration depend on other services.

---

## Validation (preview — full treatment in §10)

Options can be validated, ideally **at startup** so misconfiguration fails fast:

```csharp
services.AddOptions<WorkerOptions>()
    .Bind(config.GetSection("Worker"))
    .ValidateDataAnnotations()          // [Required], [Range] on the POCO
    .Validate(o => o.MaxRetries > 0, "MaxRetries must be positive")
    .ValidateOnStart();                 // fail at startup, not on first use
```

`ValidateOnStart()` runs validation during host startup — turning a runtime "bad config" error deep in a request into a clear startup failure. Covered fully in [10-Validation.md](10-Validation.md).

---

## Why the Options pattern beats raw `IConfiguration`

```csharp
// ✗ — stringly-typed, no validation, easy to mistype, scattered, hard to test
var n = config.GetValue<int>("Worker:MaxReties");   // typo → silent default

// ✓ — typed, validated, testable, intent-revealing
public class Worker(IOptions<WorkerOptions> options) { var n = options.Value.MaxRetries; }
```

- **Type safety** — properties, not magic strings; typos are compile errors.
- **Validation** — enforce constraints at startup.
- **Testability** — pass `Options.Create(new WorkerOptions { ... })` in tests; no config plumbing.
- **Encapsulation** — a service depends on *its* options class, not the whole config.

```csharp
// Trivial to test:
var svc = new Worker(Options.Create(new WorkerOptions { MaxRetries = 5 }));
```

---

## Common gotchas

### `IOptionsSnapshot` in a singleton

`IOptionsSnapshot<T>` is **scoped** — injecting it into a singleton is a captive dependency ([03-Lifetimes.md](03-Lifetimes.md)). Use `IOptionsMonitor<T>` in singletons/background services.

### Expecting `IOptions<T>` to reload

`IOptions<T>.Value` is computed once and never changes, even with `reloadOnChange`. For live reload use `IOptionsMonitor<T>.CurrentValue` (or `IOptionsSnapshot` per request).

### Property names not matching config keys

Binding is by property name. A mismatch silently leaves the default. Validate at startup to catch it.

### No validation → silent bad config

An unset/invalid value silently uses the default. Add `.ValidateDataAnnotations().ValidateOnStart()` to fail fast.

### Options class with no parameterless ctor / read-only props

Binding needs settable properties and a parameterless constructor. Use mutable POCOs (or records with init via supported binding) for options.

---

## Summary

- The **Options pattern** binds a config section to a typed POCO (`services.Configure<T>(section)`) and injects it — typed, validated, testable; **prefer it over raw `IConfiguration`**.
- Choose the accessor: **`IOptions<T>`** (singleton, read once — default), **`IOptionsSnapshot<T>`** (scoped, per-request, reflects reloads), **`IOptionsMonitor<T>`** (singleton, live value + `OnChange` — for singletons/background services).
- **Don't inject `IOptionsSnapshot` into singletons** (captive dependency); use `IOptionsMonitor` there.
- **Named options** carry multiple configs of one type (`monitor.Get("name")`); configure imperatively with `Configure`/`PostConfigure`/`AddOptions<T>().Configure<TDep>`.
- **Validate at startup** (`ValidateDataAnnotations().ValidateOnStart()`) so bad config fails fast — full coverage in §10.

→ Next: [09-LoggingPrimer.md](09-LoggingPrimer.md)
