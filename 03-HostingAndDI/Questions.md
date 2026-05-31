# Chapter 03 — Hosting & DI — Q & A

---

### Q1. What is the Generic Host and why does it matter?

The shared application model wiring **DI, configuration, logging, and lifetime** into one object, used by web apps, console workers, and services alike. Learn it once; the same `Services`/`Configuration`/`Logging` APIs apply everywhere (ASP.NET Core's `WebApplication.CreateBuilder` is just a host specialized for HTTP).

---

### Q2. What does `RunAsync()` do?

`StartAsync()` (calls each hosted service's `StartAsync` in registration order) + waits for a shutdown signal + `StopAsync()` (reverse order, with a shutdown timeout) + dispose. It runs the app until Ctrl+C/SIGTERM.

---

### Q3. `IHostedService` vs `BackgroundService`?

`IHostedService` is the raw start/stop interface. `BackgroundService` is a base class wrapping it with an `ExecuteAsync(stoppingToken)` long-running loop — the standard base for background work. Put long work in `ExecuteAsync` (runs after startup), not `StartAsync` (which blocks startup).

---

### Q4. Why must you create a scope inside a hosted service to use a `DbContext`?

Hosted services are **singletons**; `DbContext` is **scoped**. Injecting it directly is a captive dependency. Instead, inject `IServiceProvider`/`IServiceScopeFactory` and `CreateScope()`/`CreateAsyncScope()` per work item, resolving the `DbContext` from that scope — mimicking a per-request scope.

---

### Q5. The three service lifetimes?

**Singleton** (one per app — must be thread-safe), **Scoped** (one per scope/request — for per-request state like `DbContext`), **Transient** (one per resolution — lightweight, stateless). Default to Singleton for stateless services, Scoped for per-request state, Transient for fresh isolated instances.

---

### Q6. What is a captive dependency?

A longer-lived service depending on a shorter-lived one (e.g., a Singleton injecting a Scoped `DbContext`), which traps the scoped service for the app's lifetime — breaking per-request semantics and thread-safety. Rule: depend only on **equal-or-longer** lifetimes. Scope validation catches it in Development.

---

### Q7. What's a "scope" and when is one created automatically?

A bounded region with its own scoped instances. ASP.NET Core creates **one scope per HTTP request** automatically (so Scoped = per-request). Outside web requests you create scopes manually with `CreateScope()`/`CreateAsyncScope()`; the scope's disposal disposes its scoped + transient-disposable services.

---

### Q8. Which injection forms does the built-in container support?

**Constructor injection only** (by design — explicit). No property or method injection. Declare dependencies as constructor parameters (primary constructors are ideal); keep constructors cheap (capture deps, no I/O).

---

### Q9. `GetService` vs `GetRequiredService` vs `GetServices`?

`GetService<T>()` returns `null` if unregistered; `GetRequiredService<T>()` throws (preferred — loud, early failures); `GetServices<T>()` returns all registrations of `T` as `IEnumerable<T>`.

---

### Q10. Why keep constructors cheap?

The container constructs objects during resolution; I/O or heavy work in a constructor blocks resolution, hurts startup, and complicates testing. Capture dependencies only; do work in methods, or inject a `Func<T>`/`Lazy<T>` for on-demand creation.

---

### Q11. Who disposes services, and what's the transient-disposable trap?

The container disposes what it creates: **singletons** with the root provider, **scoped** with their scope. **Transient `IDisposable`s resolved from the root provider are rooted by it and live until app shutdown** (a leak) — resolve disposables within a scope. Instances you provide via `AddSingleton<T>(instance)` are not disposed by the container.

---

### Q12. How do you register a generic family in one line?

Open generic registration: `services.Add...(typeof(IRepository<>), typeof(EfRepository<>))`. The container closes the implementation for each requested `T` (`IRepository<Order>` → `EfRepository<Order>`). Powers generic repositories, validators, MediatR handlers, and `ILogger<T>`.

---

### Q13. How do you override one closed type while keeping an open generic default?

Register the open generic plus a specific closed registration: the closed one (`AddScoped<IRepository<AuditLog>, AuditLogRepository>()`) wins for that type; all others fall back to the open generic. Useful when most entities share a generic repo but one needs custom behavior.

---

### Q14. What are keyed services and when use them?

`.NET 8+` feature to register multiple implementations of one interface under **keys** and resolve a specific one (`AddKeyedSingleton<INotifier, EmailNotifier>("email")` + `[FromKeyedServices("email")]`). Use for genuine variants of one abstraction (channels, backends, strategies). Keyed and keyless registrations are separate.

---

### Q15. How do you decorate a service with the built-in container?

It has no `Decorate` API — use **Scrutor's** `services.Decorate<IFoo, FooDecorator>()` or a **manual factory** that builds the chain. Order matters (last decorate = outermost); match decorator lifetimes to the wrapped service.

---

### Q16. How does configuration layering work?

Multiple providers merge into one `IConfiguration`, with **later sources overriding earlier**: appsettings.json → appsettings.{Environment}.json → User Secrets (dev) → environment variables → command-line args. This enables defaults-then-overrides per environment.

---

### Q17. Where do hierarchical config keys differ between JSON and env vars?

JSON nesting uses `:` (`Worker:MaxRetries`); environment variables use `__` (double underscore: `Worker__MaxRetries`) for cross-platform compatibility.

---

### Q18. Where should secrets live?

**Never in `appsettings.json`** (it's committed). Local dev → **User Secrets** (`dotnet user-secrets`); production → **environment variables** (injected by the orchestrator) or a **secret store** (Azure Key Vault, etc.) added as a config provider.

---

### Q19. Why prefer the Options pattern over injecting `IConfiguration`?

Typed (properties not magic strings — typos are compile errors), validatable (fail fast at startup), testable (`Options.Create(new T{...})`), and encapsulated (a service depends on *its* options, not the whole config). Reserve raw `IConfiguration` for occasional reads.

---

### Q20. `IOptions<T>` vs `IOptionsSnapshot<T>` vs `IOptionsMonitor<T>`?

`IOptions<T>` — singleton, value computed once (default). `IOptionsSnapshot<T>` — scoped, recomputed per request (reflects reloads; **don't inject into singletons**). `IOptionsMonitor<T>` — singleton, live `CurrentValue` + `OnChange` (use in singletons/background services needing live reload).

---

### Q21. Why can't you inject `IOptionsSnapshot<T>` into a singleton?

`IOptionsSnapshot<T>` is **scoped**; a singleton consuming it is a captive dependency. Use `IOptionsMonitor<T>` (singleton, live value) in singletons and background services.

---

### Q22. What are named options for?

Multiple configurations of the same options type (e.g., per-tenant, per-endpoint): `Configure<EndpointOptions>("primary", ...)` and retrieve via `monitor.Get("primary")`. Useful for multi-tenant/multi-endpoint scenarios.

---

### Q23. Why use message templates instead of string interpolation in logging?

Templates (`"Order {OrderId} placed", id`) capture **structured, queryable fields** and skip formatting when the level is filtered out. Interpolation (`$"Order {id}..."`) collapses to a flat string (no structured data) and formats even when filtered. Always use named placeholders + args.

---

### Q24. What is a log scope?

`logger.BeginScope("...{OrderId}...", id)` attaches structured properties to **every log written within it** (flowing across awaits via `AsyncLocal`), so you correlate all logs of one operation without repeating IDs. ASP.NET Core auto-creates a per-request scope and correlates with the current trace.

---

### Q25. Why use `[LoggerMessage]` source generation?

It emits allocation-free, pre-compiled logging methods (no boxing of value-type args, no per-call template parsing) — ideal for hot log paths and AOT-friendly. The generic `Log*` extension methods allocate/parse per call.

---

### Q26. Why validate options at startup, and how?

To **fail fast** on misconfiguration with a clear message instead of a cryptic runtime error deep in a request. Use `AddOptions<T>().Bind(...).ValidateDataAnnotations().Validate(predicate, msg).ValidateOnStart()`. `ValidateOnStart()` makes the host refuse to start on invalid config (caught by CI/health checks before serving traffic).

---

### Q27. Difference between options validation, guard clauses, and request validation?

**Options validation** ensures the *app is configured correctly* (startup). **Guard clauses** (`ArgumentNullException.ThrowIfNull`) catch *programmer errors* (method args). **Request/model validation** (DataAnnotations/FluentValidation in ASP.NET) checks *user input* and returns 400. Different boundaries, different tools.

---

→ Next: [Coding.md](Coding.md)
