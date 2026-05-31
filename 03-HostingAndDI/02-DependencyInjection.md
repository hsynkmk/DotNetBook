# Dependency Injection (the Container)

## The built-in container

`Microsoft.Extensions.DependencyInjection` is .NET's built-in IoC container — the engine behind every host. You **register** services (mapping abstractions to implementations) in `IServiceCollection`, the host **builds** an `IServiceProvider`, and it **resolves** dependency graphs by constructor injection.

> CSharpBook Chapter 17 §12 covers DI as a **design principle** (why inject, composition root, constructor injection, the service-locator anti-pattern). This file covers the **container mechanics**: registration, resolution, factories, and the `IServiceCollection`/`IServiceProvider` APIs. Lifetimes get their own file ([03-Lifetimes.md](03-Lifetimes.md)).

```csharp
// Register (in the composition root)
builder.Services.AddSingleton<IClock, SystemClock>();
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.AddTransient<IEmailSender, SmtpEmailSender>();
builder.Services.AddSingleton<OrderService>();

// Resolve (the container injects the graph)
public class OrderService(IOrderRepository repo, IEmailSender email, IClock clock) {
    // repo, email, clock supplied automatically — you never call `new` for them
}
```

---

## Registration methods

```csharp
// By lifetime + abstraction → implementation
services.AddSingleton<IFoo, Foo>();
services.AddScoped<IBar, Bar>();
services.AddTransient<IBaz, Baz>();

// Concrete type (no interface)
services.AddSingleton<OrderService>();

// Instance (you provide the object — always a singleton)
services.AddSingleton<IClock>(new SystemClock());

// Factory delegate (compute the instance, access other services)
services.AddSingleton<IConnection>(sp => {
    var config = sp.GetRequiredService<IConfiguration>();
    return new Connection(config["Db:ConnectionString"]!);
});

// TryAdd — register only if not already registered (for library defaults)
services.TryAddSingleton<IClock, SystemClock>();

// Multiple implementations of one interface → resolved as IEnumerable<T>
services.AddSingleton<IValidator, EmailValidator>();
services.AddSingleton<IValidator, PhoneValidator>();
// constructor: (IEnumerable<IValidator> validators) → both injected
```

The **factory delegate** overload (`sp => ...`) is the escape hatch when construction needs configuration or other services. **`TryAdd*`** lets a library register a default that an app can override (the app's explicit registration wins if it ran first; `TryAdd` no-ops if already present).

---

## How resolution works

When you ask for `OrderService`, the container:
1. Looks up `OrderService`'s registration and its **constructor**.
2. Recursively resolves each constructor parameter (`IOrderRepository` → `SqlOrderRepository`, whose constructor params are resolved too…).
3. Constructs the whole graph and returns the root.

```csharp
var svc = provider.GetRequiredService<OrderService>();   // builds the entire dependency tree
var maybe = provider.GetService<IOptional>();             // returns null if not registered
var all = provider.GetServices<IValidator>();             // IEnumerable of all registrations
```

- **`GetRequiredService<T>()`** — throws if not registered (use this; failures are loud and early).
- **`GetService<T>()`** — returns `null` if not registered.
- **`GetServices<T>()`** — all registrations of `T`.

The container uses the **constructor with the most parameters it can satisfy** (and errors if it's ambiguous or a parameter can't be resolved). Constructor injection is the only injection the built-in container does — there's no property/method injection (by design — explicit is better).

---

## Constructor injection only — and keep constructors cheap

The container injects via the constructor. So:
- **Declare dependencies as constructor parameters** (primary constructors are ideal — CSharpBook Ch02 §12).
- **Constructors should only capture dependencies** — no I/O, no heavy work, no blocking. The container may construct objects eagerly or lazily; expensive constructors hurt startup and resolution.

```csharp
// ✗ — work in the constructor (blocks resolution, hard to test)
public class Bad {
    private readonly Data _data;
    public Bad(IRepo repo) { _data = repo.LoadEverything(); }   // I/O at construction!
}

// ✓ — capture deps; do work in a method
public class Good(IRepo repo) {
    public Task<Data> LoadAsync() => repo.LoadEverythingAsync();
}
```

If you need lazy/expensive creation, inject a **factory** (`Func<T>`) or `Lazy<T>` rather than doing the work in a constructor.

---

## Injecting factories: `Func<T>` and `IServiceProvider`

Sometimes you need to create instances on demand (per loop iteration, conditionally):

```csharp
// Func<T> factory (register it)
services.AddTransient<Report>();
services.AddSingleton<Func<Report>>(sp => () => sp.GetRequiredService<Report>());
public class Generator(Func<Report> reportFactory) {
    public void Make() { var r = reportFactory(); /* fresh each call */ }
}
```

Avoid injecting `IServiceProvider` into your business classes to resolve things manually — that's the **service-locator anti-pattern** (hides dependencies, hard to test — CSharpBook Ch17 §09/§12). It's acceptable at framework edges and inside factories, but domain code should declare its dependencies explicitly.

---

## `IServiceCollection` is just a list

`IServiceCollection` is literally a `List<ServiceDescriptor>` — registrations are descriptors (service type, implementation/factory, lifetime). This means:
- Order can matter (`TryAdd`, last-wins for some scenarios, `IEnumerable<T>` preserves order).
- You can **inspect/modify** it before `Build()` (e.g., `services.Remove`/`Replace` to swap an implementation — used heavily in testing, [Ch17 Testing](../17-Testing/README.md)).

```csharp
// Replace a registration (e.g., a test double)
services.Replace(ServiceDescriptor.Singleton<IEmailSender, FakeEmailSender>());
services.RemoveAll<IPaymentGateway>();
```

After `Build()`, the provider is fixed — you can't register new services into a built provider.

---

## Disposal

The container **owns and disposes** the services it creates (if they implement `IDisposable`/`IAsyncDisposable`):
- **Singletons** are disposed when the **root provider** (the host) is disposed.
- **Scoped** services are disposed when their **scope** is disposed (e.g., end of a web request — [03-Lifetimes.md](03-Lifetimes.md)).
- **Transient `IDisposable`s** are tracked by the scope/provider that created them and disposed with it — which means transient disposables resolved from the root live as long as the app! (a subtle leak — prefer resolving disposables in a scope).

```csharp
// ⚠ — transient IDisposable resolved from root lives until app shutdown
var tmp = rootProvider.GetRequiredService<TransientDisposable>();   // tracked by root → never freed early
```

Exception: instances **you** provide via `AddSingleton<T>(instance)` are **not** disposed by the container (you own them).

---

## Common gotchas

### Service-locator instead of injection

Injecting `IServiceProvider` and calling `GetService` in business logic hides dependencies. Inject the specific abstractions (CSharpBook Ch17 §12).

### Forgetting to register a dependency

`GetRequiredService` throws a clear "no service registered for type X" at resolution. Use `GetRequiredService` (not `GetService`) so failures are loud. Build the host early in tests to catch missing registrations.

### Work in constructors

Blocks resolution and complicates testing. Capture deps only; do work in methods or inject a factory.

### Transient disposables from the root provider

They're rooted by the root container and never freed until shutdown → leak. Resolve disposables within a scope.

### Multiple registrations when you expected one

Registering the same interface twice means `GetRequiredService<T>` returns the **last** one, while `GetServices<T>` returns all. Use `TryAdd`/`Replace` deliberately.

---

## Summary

- The built-in container maps abstractions → implementations in **`IServiceCollection`**, builds an **`IServiceProvider`**, and resolves graphs by **constructor injection** (the only kind it does).
- Register with `Add{Singleton,Scoped,Transient}` (by type, instance, or **factory delegate**); `TryAdd*` for library defaults; multiple registrations resolve as `IEnumerable<T>`.
- Resolve with **`GetRequiredService<T>`** (throws if missing — preferred), `GetService<T>` (null), `GetServices<T>` (all).
- Keep **constructors cheap** (capture deps only); inject a **`Func<T>`/`Lazy<T>`** for on-demand creation; avoid the service-locator anti-pattern.
- The container **disposes what it creates** (singletons with the root, scoped with their scope) — beware transient disposables resolved from the root (they leak until shutdown).
- DI as a design principle: CSharpBook Ch17 §12. Lifetimes next.

→ Next: [03-Lifetimes.md](03-Lifetimes.md)
