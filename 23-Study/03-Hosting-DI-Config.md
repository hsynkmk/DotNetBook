# 03 — Hosting, Dependency Injection & Configuration

> Covers the book's **Chapter 03 (Hosting & DI)** and **Chapter 13 (Configuration)** — they
> describe one mechanism and interview questions move freely between them.

## ⚡ 30-second answer

The **Generic Host** is the universal .NET app model — DI, configuration, logging and lifetime —
shared by web apps, workers and services. The built-in container resolves graphs by
**constructor injection** and offers three lifetimes: **Singleton** (one per app, must be
thread-safe), **Scoped** (one per request/scope — `DbContext`), **Transient** (one per
resolution). The #1 DI bug is the **captive dependency**: a longer-lived service capturing a
shorter-lived one — inject a scoped `DbContext` into a singleton and you've pinned one context
for the app's lifetime, shared across threads. **`IConfiguration`** merges layered sources into
one flat string dictionary where **later sources win** (JSON → environment JSON → user secrets →
env vars → command line), and you consume it through the **Options pattern** — bound to typed
POCOs and **validated at startup with `ValidateOnStart()`** so misconfiguration fails the boot
rather than a request at 3am.

---

## Core mechanics

### The host

```csharp
var builder = Host.CreateApplicationBuilder(args);     // WebApplication.CreateBuilder for HTTP
builder.Services.AddSingleton<IClock, SystemClock>();
builder.Services.AddHostedService<Worker>();
var host = builder.Build();
await host.RunAsync();                                  // start → run → graceful shutdown
```

`builder` exposes **`Services`, `Configuration`, `Logging`, `Environment`**. Hosted services
**start in registration order and stop in reverse**. `StartAsync` blocks startup, so keep it
fast — long loops belong in `BackgroundService.ExecuteAsync(stoppingToken)`
([08](08-BackgroundProcessing.md)).

### Registration and resolution

```csharp
services.AddSingleton<IClock, SystemClock>();              // by type
services.AddScoped<IOrderService>(sp => new OrderService(sp.GetRequiredService<AppDb>()));
services.TryAddSingleton<IClock, SystemClock>();           // library default — don't stomp callers
services.AddSingleton<IHandler, A>();                      // multiple registrations
services.AddSingleton<IHandler, B>();                      //   → resolve as IEnumerable<IHandler>
```

- **`GetRequiredService<T>`** (throws — preferred) vs `GetService<T>` (null) vs `GetServices<T>`.
- **Constructor injection is the only kind the container does.** Keep constructors cheap — capture
  dependencies, do no work. Inject `Func<T>`/`Lazy<T>` for deferred creation.
- **The container disposes what it creates**: singletons with the root provider, scoped with
  their scope. It does *not* dispose instances you `new` up and register yourself.
- **Open generics** register a whole family in one line — this is how `ILogger<T>` works:

  ```csharp
  services.AddScoped(typeof(IRepository<>), typeof(EfRepository<>));   // note the empty <>
  ```

- **Keyed services** (.NET 8+) pick one of N variants by key — prefer enums over strings:

  ```csharp
  services.AddKeyedSingleton<INotifier, EmailNotifier>(Channel.Email);
  app.MapPost("/notify", ([FromKeyedServices(Channel.Email)] INotifier n) => …);
  ```

- **Decorators** add cross-cutting behavior around a service. The built-in container has no
  `Decorate`; use **Scrutor** or a factory registration. **Order matters** — the last decorator
  applied is outermost, so `Logging(Caching(Real))` logs every call while
  `Caching(Logging(Real))` logs only cache misses.

### Lifetimes and scopes

A **scope** bounds scoped instances — ASP.NET Core creates one per HTTP request automatically.
To use a scoped service from a singleton or hosted service, **create a scope per unit of work**:

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    while (!stoppingToken.IsCancellationRequested)
    {
        await using var scope = _scopeFactory.CreateAsyncScope();   // one scope per work item
        var db = scope.ServiceProvider.GetRequiredService<AppDb>();
        …
    }
}
```

**Scope validation** (`ValidateScopes`, on by default in Development) catches captive
dependencies at startup. `ValidateOnBuild` catches missing registrations too — turn both on.

### Configuration

`IConfiguration` is a **flat string dictionary**. Nested JSON collapses to `:`-delimited keys,
**every value is a string**, and a missing key returns **null silently**.

```jsonc
{ "Worker": { "MaxRetries": 5 } }        // → key "Worker:MaxRetries", value "5"
```

Environment variables use **`__`** for hierarchy (`Worker__MaxRetries`), because `:` is illegal
in env-var names on some platforms. Arrays are indexed keys (`:0`, `:1`) and — a sharp edge —
override **by index**, not as a whole array.

Default precedence (**later wins**):

```text
appsettings.json → appsettings.{Environment}.json → User Secrets (dev) → env vars → command line
```

`ASPNETCORE_ENVIRONMENT` selects the environment file and **defaults to Production if unset**.
Environment files should hold only what *differs* from the base, or they drift.

### Options

```csharp
builder.Services.AddOptions<SmtpOptions>()
    .Bind(builder.Configuration.GetSection("Smtp"))
    .ValidateDataAnnotations()
    .Validate(o => o.Port is > 0 and < 65536, "Port out of range")
    .ValidateOnStart();          // ← the line that turns a 3am bug into a failed deploy
```

Pick the accessor by lifetime and reload behavior:

| Accessor | Lifetime | Reflects reloads | Use for |
|---|---|---|---|
| **`IOptions<T>`** | singleton | **no** — read once | the default; static config |
| **`IOptionsSnapshot<T>`** | scoped | yes, per request | per-request config in a web app |
| **`IOptionsMonitor<T>`** | singleton | yes, live + `OnChange` | **singletons & background services** |

**Named options** carry several configurations of one type (`monitor.Get("primary")`).
`PostConfigure` applies final defaults last. `AddOptions<T>().Configure<TDep>(…)` configures
options using other DI services.

### Reload and secrets

Reload is built on **`IChangeToken`**. **File providers reload** (`reloadOnChange: true`);
**environment variables and command-line args are read once at startup and never change** — so a
setting meant to change at runtime must come from a reloadable source. In containers, prefer
immutable config per deployment for most settings and reload only for genuinely dynamic ones
(log levels, feature flags).

Secrets are configuration that must never be in the repo. **Dev**: `dotnet user-secrets` (stored
in your user profile, auto-loaded in Development, *not encrypted*). **Production**: a secret
store like Key Vault wired in as a configuration provider, so consumers still read ordinary
keys. **Managed identity** (`DefaultAzureCredential`) is the gold standard — the app holds no
secret at all, and the same code falls back to your developer login locally.

### Logging primer

```csharp
_logger.LogInformation("Order {OrderId} shipped to {Region}", id, region);   // ✅ template
using var scope = _logger.BeginScope(new Dictionary<string, object> { ["TenantId"] = tenant });
```

Message **templates with named placeholders**, never interpolation — that's what preserves
structured, queryable fields ([12](12-Observability.md)).

---

## Comparison tables

| Lifetime | Instances | Thread-safety | Typical use |
|---|---|---|---|
| **Singleton** | one per app | **you must ensure it** | stateless services, caches, `HttpClient` factories |
| **Scoped** | one per scope/request | one request at a time | `DbContext`, per-request context |
| **Transient** | one per resolution | n/a | lightweight, stateless, short-lived |

| Depends on ↓ / Lifetime → | Singleton | Scoped | Transient |
|---|---|---|---|
| **Singleton** | ✅ | ❌ **captive** | ⚠️ captured for app lifetime |
| **Scoped** | ✅ | ✅ | ⚠️ captured for the scope |
| **Transient** | ✅ | ✅ | ✅ |

| Cross-cutting tool | Layer | Sees |
|---|---|---|
| **Middleware** | HTTP pipeline | every request, raw `HttpContext` |
| **Filter / endpoint filter** | endpoint | bound arguments + endpoint metadata |
| **Decorator** | DI | one service's method calls |
| **`DelegatingHandler`** | outbound HTTP | requests your app *makes* |

| Validation kind | When it runs | Failure means |
|---|---|---|
| **Options validation** | startup (`ValidateOnStart`) | the deploy fails — good |
| **Argument guards** | call time | programmer error → exception |
| **Request/model validation** | per request | user error → 400 ([04](04-AspNetCore.md)) |

---

## 🪤 Traps & gotchas

- **Captive dependency** — the canonical DI interview trap. A singleton that takes a scoped
  dependency pins one instance forever:

  ```csharp
  services.AddSingleton<CacheWarmer>();      // 🐛 CacheWarmer(AppDb db) — one DbContext,
  services.AddDbContext<AppDb>();            //    for the whole app, used from many threads
  // ✅ inject IServiceScopeFactory and create a scope per unit of work
  ```

  Development-mode scope validation catches this at startup — don't disable it.
- **Injecting `IOptionsSnapshot<T>` into a singleton** — same bug in a different hat.
  `IOptionsSnapshot` is scoped; singletons need **`IOptionsMonitor<T>`**.
- **Mutable state in a singleton** — singletons are hit concurrently by every request. A plain
  `Dictionary` field is a race; use `ConcurrentDictionary` or a lock.
- **Transient `IDisposable` resolved from the root provider** — the root container holds it until
  shutdown, so it never gets collected. A slow, silent leak.
- **Disposing an injected service yourself** — the container owns it; disposing early breaks
  everyone else holding the same instance.
- **Service locator** (`serviceProvider.GetService<T>()` sprinkled through business logic) hides
  dependencies from the constructor, so nothing fails at startup and everything fails at runtime.
- **Work in constructors** — DI builds whole object graphs eagerly. I/O or blocking calls in a
  constructor turn a resolution into a timeout.
- **Async in a constructor** — you can't `await` there, and `.Result` deadlocks. Use an async
  factory or an `IHostedService` that does the init.
- **`Add` vs `TryAdd` in a library** — plain `Add` on a shared abstraction silently produces two
  registrations and `IEnumerable<T>` resolution surprises. Libraries use `TryAdd*`.
- **Registering an open generic with the wrong arity** or forgetting the empty `<>` —
  `typeof(IRepository<>)`, not `typeof(IRepository<Order>)`.
- **Keyed and keyless registrations are separate.** Registering only keyed services and then
  resolving `INotifier` plainly throws.
- **Decorator order reversed** — `Caching(Logging(Real))` logs only what misses the cache. State
  which you want and why.
- **Expecting env vars to reload** — they're read once at startup. Same for command-line args.
  A "hot-reloadable" setting sourced from an env var never changes.
- **`IOptions<T>` never reloads** — if a value must track config changes, it's `IOptionsMonitor`.
- **Forgetting `ValidateOnStart()`** — without it validation is *lazy*, running on first `.Value`
  access. Your bad connection string surfaces on the first request in production instead of at
  boot.
- **`ASPNETCORE_ENVIRONMENT` unset defaults to Production** — so a forgotten variable silently
  skips your Development config and User Secrets.
- **Env-var hierarchy needs `__`, not `:`** — `Worker:MaxRetries` as an env var simply doesn't
  bind on Linux.
- **Configuration arrays override by index** — setting `Hosts:0` replaces the first element and
  leaves the rest of the base array intact, which is almost never what you meant.
- **A missing configuration key returns null, silently** — no exception, no warning. This is
  exactly why options validation exists.
- **Environment JSON files that are full copies** of the base drift apart within a sprint. Only
  the differences belong there.
- **User Secrets are not encrypted** and are Development-only — never a production mechanism.
- **A committed secret must be rotated, not deleted** — git history keeps it forever.
- **Logging with string interpolation** destroys structured logging:

  ```csharp
  _logger.LogInformation($"Order {id} shipped");   // 🐛 one opaque string, formatted even if filtered out
  _logger.LogInformation("Order {OrderId} shipped", id);   // ✅ queryable field
  ```

---

## ❓ Likely questions

**Q: What are the three DI lifetimes and when do you use each?**
A: **Singleton** — one instance for the app; stateless services and caches, and it must be
thread-safe. **Scoped** — one per scope, which in a web app means one per request; `DbContext`
and per-request state. **Transient** — a new one every resolution; lightweight stateless
helpers.

**Q: What is a captive dependency?**
A: A longer-lived service capturing a shorter-lived one — classically a singleton holding a
scoped `DbContext`. The scoped service is never released, so per-request semantics break and a
non-thread-safe object gets shared across threads. Fix by injecting `IServiceScopeFactory` and
creating a scope per unit of work. Development scope validation catches it at startup.

**Q: How do you use a scoped service inside a singleton or `BackgroundService`?**
A: Inject `IServiceScopeFactory`, create a scope per work item (`CreateAsyncScope`), resolve from
that scope, and dispose it when the item is done.

**Q: How does the container decide which constructor to use?**
A: It picks the constructor whose parameters it can all resolve — and throws if that's ambiguous.
Constructor injection is the only form it supports; there's no property or method injection.

**Q: Who disposes services, and where does that go wrong?**
A: The container disposes what it *created* — singletons with the root provider, scoped
instances with their scope. It doesn't dispose instances you registered as pre-built objects. The
leak to know: a transient `IDisposable` resolved from the **root** provider is held until the app
shuts down.

**Q: `GetService` vs `GetRequiredService`?**
A: `GetService<T>` returns null when unregistered; `GetRequiredService<T>` throws. Prefer the
throwing one — a missing registration is a bug you want loudly, not a `NullReferenceException`
three frames later.

**Q: What are keyed services for?**
A: Registering several implementations of one interface under keys and resolving a specific one —
notification channels, storage backends, strategies. It replaces hand-rolled factories and
`IEnumerable<T>` filtering. Keyed and keyless registrations are separate namespaces.

**Q: How do you add cross-cutting behavior to a service?**
A: A decorator — wrap the interface in another implementation that adds caching, logging or
retry. The built-in container has no `Decorate`, so use Scrutor or a factory registration, and
be deliberate about order because the last decorator applied is the outermost.

**Q: How does `IConfiguration` merge sources?**
A: Every provider contributes flat `string→string` pairs, and they merge in registration order
with **later sources overriding earlier** — but only for the keys they actually specify; the rest
fall through. Default order is appsettings.json, appsettings.{Environment}.json, User Secrets in
Development, environment variables, then command line.

**Q: Why do environment variables use `__` instead of `:`?**
A: `:` is not a legal environment-variable name character on all platforms, so the double
underscore is the hierarchy separator. `Worker__MaxRetries` binds to `Worker:MaxRetries`.

**Q: Why prefer the Options pattern over reading `IConfiguration` directly?**
A: Typed, bound once, validated at startup, and injectable as a plain POCO — so consuming code
doesn't know configuration exists and is trivial to unit test. Raw string-key reads are untyped,
silently null when misspelled, and scatter config knowledge everywhere.

**Q: `IOptions` vs `IOptionsSnapshot` vs `IOptionsMonitor`?**
A: `IOptions<T>` is a singleton read once — no reload. `IOptionsSnapshot<T>` is scoped and
re-binds per request, reflecting reloads. `IOptionsMonitor<T>` is a singleton with a live
`CurrentValue` and an `OnChange` callback — the one to use in singletons and background services.
Never inject the snapshot into a singleton.

**Q: What does `ValidateOnStart()` do and why does it matter?**
A: Without it, options validation runs lazily on first `.Value` access — so bad configuration
surfaces as a runtime failure in production. With it, the host fails to start with a clear
message, which CI and your deployment gate catch first.

**Q: Which configuration sources reload, and which don't?**
A: File-based providers reload when `reloadOnChange: true` (and some cloud providers do too).
Environment variables and command-line arguments are read **once at startup**. So a setting you
intend to change at runtime must not come from an env var.

**Q: How do you handle secrets in dev vs production?**
A: Dev: User Secrets, stored outside the project in your user profile and auto-loaded in
Development — convenient, not encrypted, never for production. Production: a secret store such as
Key Vault registered as a configuration provider, authenticated with a **managed identity** so
the app holds no secret at all.

**Q: A secret got committed. What do you do?**
A: Rotate it. Deleting the line doesn't help — git history keeps the old blob, and anyone with a
clone already has it.

**Q: Why message templates instead of interpolated strings in logs?**
A: The template keeps `OrderId` as a structured, queryable field for your log backend, and the
arguments are only formatted if the level is actually enabled. Interpolation collapses everything
to one opaque string and pays the formatting cost even when the log is filtered out.

---

## 🎓 Senior Extra

- **Turn on `ValidateOnBuild` and `ValidateScopes` in every environment**, not just Development.
  The cost is a slightly slower startup; the payoff is that captive dependencies and missing
  registrations fail the deploy instead of the request.
- **The container is deliberately minimal.** It has no property injection, no interception, no
  auto-registration by convention — because it's the *lowest common denominator* every library can
  target. Reach for Autofac/Scrutor when you genuinely need decoration, assembly scanning, or
  interception, and know that's why.
- **`IHostedService` vs `BackgroundService`**: `StartAsync` blocks the host's startup sequence, so
  anything long-running belongs in `ExecuteAsync`. An unhandled exception in `ExecuteAsync` stops
  the host by default in .NET 6+ (`BackgroundServiceExceptionBehavior`) — which is usually what
  you want, but is surprising the first time.
- **Shutdown order is reverse of registration**, which matters when a consumer must drain before
  the thing it depends on stops.
- **`IOptionsMonitor.OnChange` can fire more than once per file write** (editors write in
  multiple steps) — debounce anything expensive hanging off it.
- **Options validation on reload throws on access rather than crashing the app** — deliberate, so
  a bad config push doesn't take down a running service, but it means you can't rely on
  validation alone for hot-reloaded safety-critical values.
- **Configuration binding is reflection-based**, so it's a Native AOT consideration; the
  configuration source generator handles the trim-safe case
  ([18–19](18-19-Aspire-Deployment.md)).
- **`[LoggerMessage]` source generator** gives allocation-free, AOT-friendly logging for hot
  paths, and turns the template into a compile-time-checked signature.
- **A composition root is a design choice, not a framework one.** Registration lives in one place
  (Program.cs plus per-module `AddX` extension methods); the rest of the codebase never sees
  `IServiceProvider`. That is the actual line between DI and service location
  ([22](22-BestPractices-Architecture.md)).
- **Azure App Configuration** centralizes settings and feature flags with dynamic refresh across
  many services, complementing (not replacing) file/env providers plus Key Vault for secrets
  ([20](20-Azure.md)).

→ Deeper: [`../03-HostingAndDI/`](../03-HostingAndDI/README.md) ·
[`../13-Configuration/`](../13-Configuration/README.md)
