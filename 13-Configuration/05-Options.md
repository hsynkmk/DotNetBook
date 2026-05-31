# The Options Pattern (Deep)

## Typed configuration, properly

The Options pattern binds configuration sections to strongly-typed classes and injects them — the recommended way to consume configuration ([01-IConfiguration.md](01-IConfiguration.md)). It gives type safety, validation ([06-Validation.md](06-Validation.md)), testability, and encapsulation, instead of scattering stringly-typed `config["A:B"]` reads.

```csharp
public class WorkerOptions {
    public int MaxRetries { get; set; } = 3;
    public TimeSpan Interval { get; set; } = TimeSpan.FromSeconds(30);
}
builder.Services.Configure<WorkerOptions>(builder.Configuration.GetSection("Worker"));

public class Worker(IOptions<WorkerOptions> options) {
    private readonly WorkerOptions _opts = options.Value;   // typed access
}
```

> The Options pattern is introduced in [Ch03 §08](../03-HostingAndDI/08-Options.md) (the three accessors, named options). This file goes deeper: accessor selection, `OptionsBuilder`, post-configuration, and lifetime nuances.

---

## The three accessors (recap + selection)

The crucial decision is *which accessor to inject*, because they differ in lifetime and reload behavior ([04-Reload.md](04-Reload.md)):

| Accessor | Lifetime | Reload | Inject into |
|---|---|---|---|
| `IOptions<T>` | singleton | **never** (read once) | anything; config that doesn't change |
| `IOptionsSnapshot<T>` | **scoped** | per request | scoped services (controllers, handlers) |
| `IOptionsMonitor<T>` | singleton | **live** (`CurrentValue` + `OnChange`) | **singletons / background services** |

```csharp
public class A(IOptions<WorkerOptions> o) { var v = o.Value; }                 // fixed config
public class B(IOptionsSnapshot<WorkerOptions> o) { var v = o.Value; }          // per-request, reflects reload
public class C(IOptionsMonitor<WorkerOptions> m) { var v = m.CurrentValue; }    // singleton, live
```

The recurring trap: **don't inject `IOptionsSnapshot<T>` into a singleton** (it's scoped → captive dependency — [Ch03 §03](../03-HostingAndDI/03-Lifetimes.md)). Use `IOptionsMonitor<T>` in singletons/background services for live config; `IOptions<T>` for config that's fixed at startup; `IOptionsSnapshot<T>` in request-scoped services that want per-request (reload-reflecting) values.

---

## `OptionsBuilder` — the fluent configuration API

`AddOptions<T>()` returns an `OptionsBuilder<T>` for richer configuration than `Configure` alone — binding, validation, and dependency-aware configuration in one fluent chain:

```csharp
builder.Services.AddOptions<WorkerOptions>()
    .Bind(builder.Configuration.GetSection("Worker"))     // bind from config
    .Configure(o => o.MaxRetries = Math.Max(o.MaxRetries, 1))  // adjust after binding
    .ValidateDataAnnotations()                            // validate (see [06-Validation.md])
    .ValidateOnStart();                                   // fail at startup if invalid
```

`OptionsBuilder` chains binding, in-code configuration, validation, and startup validation. It's the modern, fuller way to set up options (vs the basic `Configure<T>(section)`), especially when you need validation or dependency-aware configuration.

---

## Configuration that depends on other services

A powerful feature: options can be configured using **other DI services** (e.g., compute a value from another option, or look something up):

```csharp
builder.Services.AddOptions<CacheOptions>()
    .Configure<IHostEnvironment>((o, env) =>
        o.Directory = Path.Combine(env.ContentRootPath, "cache"))   // config depends on the environment
    .Configure<IClock>((o, clock) => o.StartTime = clock.UtcNow);
```

`Configure<TDep1, ...>((options, dep) => ...)` injects services into the configuration step (up to 5 dependencies). This lets options be computed from runtime services rather than only static config — useful when a setting derives from the environment, another service, or runtime state.

---

## Post-configuration

`PostConfigure` runs **after** all `Configure` actions — for defaults/overrides that must win regardless of order:

```csharp
builder.Services.Configure<WorkerOptions>(config.GetSection("Worker"));   // bind
builder.Services.PostConfigure<WorkerOptions>(o => o.QueueName ??= "default");  // ensure a default last
```

`Configure` actions run in registration order; `PostConfigure` runs last. Use it for final normalization/defaults that should apply after everything else has had its say (e.g., "if no queue name was set by any source, use 'default'").

---

## Named options

For **multiple instances** of the same options type (per-tenant, per-endpoint — [Ch03 §08](../03-HostingAndDI/08-Options.md)):

```csharp
builder.Services.Configure<EndpointOptions>("primary", config.GetSection("Endpoints:Primary"));
builder.Services.Configure<EndpointOptions>("replica", config.GetSection("Endpoints:Replica"));

public class Service(IOptionsMonitor<EndpointOptions> monitor) {
    void Use() {
        var primary = monitor.Get("primary");   // retrieve by name
        var replica = monitor.Get("replica");
    }
}
```

Named options let one type carry several configurations, retrieved via `Get(name)` on `IOptionsMonitor`/`IOptionsSnapshot`. Useful for multi-tenant or multi-endpoint scenarios where the same shape has different values per name.

---

## Why the Options pattern (recap)

```csharp
// ✗ — stringly-typed: typo compiles, no validation, scattered, hard to test
var n = config.GetValue<int>("Worker:MaxReties");   // typo → silent default

// ✓ — typed, validated, testable, encapsulated
public class Worker(IOptions<WorkerOptions> o) { var n = o.Value.MaxRetries; }

// Trivial to test — no config plumbing:
var worker = new Worker(Options.Create(new WorkerOptions { MaxRetries = 5 }));
```

Type safety (typos are compile errors), validation at startup ([06-Validation.md](06-Validation.md)), testability (`Options.Create(...)`), and encapsulation (a service depends on *its* options, not the whole config) — all reasons to prefer Options over raw `IConfiguration` reads.

---

## Common gotchas

### `IOptionsSnapshot` in a singleton

It's scoped → captive dependency. Use `IOptionsMonitor` in singletons/background services.

### Expecting `IOptions<T>` to reload

`IOptions<T>.Value` is computed once and never updates. For live config use `IOptionsMonitor<T>.CurrentValue` ([04-Reload.md](04-Reload.md)).

### Property names not matching config keys

Binding is by name (case-insensitive); a mismatch silently leaves the default. Validate at startup to catch it ([06-Validation.md](06-Validation.md)).

### Options class without settable properties / parameterless ctor

Binding needs mutable properties and a parameterless constructor. Use mutable POCOs for options.

### Not validating

An unset/invalid value silently uses the default. Add `.ValidateDataAnnotations().ValidateOnStart()` ([06-Validation.md](06-Validation.md)).

### Configuring options with services but using the wrong lifetime

`Configure<TDep>` resolves the dependency when options are built; ensure the dependency's lifetime is compatible (don't capture a scoped service into a singleton option).

---

## Summary

- The **Options pattern** binds config sections to typed POCOs (`Configure<T>` / `AddOptions<T>()`) and injects them — typed, validated, testable; prefer it over raw `IConfiguration` reads.
- Choose the accessor by lifetime/reload: **`IOptions<T>`** (singleton, read once), **`IOptionsSnapshot<T>`** (scoped, per-request, reflects reload), **`IOptionsMonitor<T>`** (singleton, live — for singletons/background services). Never inject `IOptionsSnapshot` into a singleton.
- **`OptionsBuilder`** (`AddOptions<T>()`) fluently chains bind + configure + validate + `ValidateOnStart`; **`Configure<TDep>`** configures options using other DI services; **`PostConfigure`** applies final defaults last.
- **Named options** carry multiple configs of one type (`Get("name")`); validate options at startup ([06-Validation.md](06-Validation.md)).

→ Next: [06-Validation.md](06-Validation.md)
