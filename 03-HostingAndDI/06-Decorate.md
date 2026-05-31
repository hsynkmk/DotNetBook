# Decorating Services

## Wrapping a service to add behavior

The **decorator pattern** wraps a service in another implementation of the same interface to add cross-cutting behavior — logging, caching, retry, metrics, validation — **without modifying the original**. In DI, you register a decorator that takes the inner service as a dependency.

```csharp
public interface IOrderRepository { Order? Get(int id); }

// The real implementation
public class SqlOrderRepository : IOrderRepository { public Order? Get(int id) => /* db */; }

// A caching decorator — same interface, wraps an inner IOrderRepository
public class CachingOrderRepository(IOrderRepository inner, IMemoryCache cache) : IOrderRepository {
    public Order? Get(int id) => cache.GetOrCreate(id, _ => inner.Get(id));
}

// A logging decorator
public class LoggingOrderRepository(IOrderRepository inner, ILogger<IOrderRepository> log) : IOrderRepository {
    public Order? Get(int id) { log.LogDebug("Get {Id}", id); return inner.Get(id); }
}
```

Each decorator adds **one** concern (single responsibility) and is composable — you stack them. (Decorator pattern fundamentals: CSharpBook Ch17 §11.)

---

## The problem: the built-in container has no `Decorate`

`Microsoft.Extensions.DependencyInjection` doesn't have a first-class decorate API. There are two practical approaches.

### Approach 1: Scrutor (the common choice)

[Scrutor](https://github.com/khellang/Scrutor) is a small, popular NuGet package that adds `Decorate` (and assembly-scanning registration) to `IServiceCollection`:

```csharp
services.AddScoped<IOrderRepository, SqlOrderRepository>();   // the real one
services.Decorate<IOrderRepository, CachingOrderRepository>(); // wrap it
services.Decorate<IOrderRepository, LoggingOrderRepository>(); // wrap that

// Resolution yields: Logging( Caching( Sql ) )
// A consumer injecting IOrderRepository gets the outermost (Logging) decorator.
```

`Decorate` re-registers the interface so that resolving it produces the decorator, with the previous registration injected as the inner. **Order matters**: the last `Decorate` is the outermost layer. Above, a call goes Logging → Caching → Sql (logs every call; caches misses hit SQL).

### Approach 2: Manual factory registration

Without Scrutor, register a factory that constructs the chain explicitly:

```csharp
services.AddScoped<SqlOrderRepository>();   // register the concrete inner
services.AddScoped<IOrderRepository>(sp => {
    IOrderRepository inner = sp.GetRequiredService<SqlOrderRepository>();
    inner = new CachingOrderRepository(inner, sp.GetRequiredService<IMemoryCache>());
    inner = new LoggingOrderRepository(inner, sp.GetRequiredService<ILogger<IOrderRepository>>());
    return inner;
});
```

More verbose but no extra dependency, and the composition is explicit. Use this for a one-off; use Scrutor when you decorate often.

---

## Order matters

Decorator order changes behavior:

```csharp
services.Decorate<IRepo, CachingRepo>();   // then
services.Decorate<IRepo, LoggingRepo>();   // → Logging(Caching(Real)): logs EVERY call
// vs reverse:
services.Decorate<IRepo, LoggingRepo>();   // then
services.Decorate<IRepo, CachingRepo>();   // → Caching(Logging(Real)): logs only cache MISSES
```

Decide deliberately: do you want to log all calls (logging outermost) or only those that reach the real service (caching outermost)? The same applies to retry vs metrics vs validation ordering.

---

## What decorators are great for

- **Caching** — wrap a repository/gateway to cache results.
- **Logging / metrics / tracing** — observe calls without touching the implementation.
- **Retry / circuit-breaker** — though for HTTP, prefer Polly via `IHttpClientFactory` ([Ch11 Resilience](../11-Resilience/README.md)).
- **Validation / authorization** — guard before delegating.
- **Feature toggling** — switch behavior at the boundary.

The .NET platform itself uses decoration extensively: `Stream` decorators (`GZipStream`, `CryptoStream`), ASP.NET Core **middleware** (a decorator chain over the request pipeline — [Ch04 §05](../04-AspNetCore/README.md)), and `DelegatingHandler` chains in `HttpClient`.

---

## Decorators vs middleware vs interceptors

| Technique | Where | Use |
|---|---|---|
| **DI decorator** | around a service interface | cross-cutting on a specific service |
| **Middleware** | around the HTTP request pipeline | cross-cutting on every request ([Ch04](../04-AspNetCore/README.md)) |
| **`DelegatingHandler`** | around outbound HTTP | cross-cutting on `HttpClient` calls ([Ch09](../09-NetworkingAndHttp/README.md)) |
| **MediatR pipeline behaviors** | around request handlers | cross-cutting on CQRS commands/queries ([Ch22](../22-BestPractices/README.md)) |
| **Dynamic proxy / AOP** (Castle) | runtime-generated wrapper | broad interception (heavier; reflection/codegen) |

Decorators are the DI-level tool. For HTTP-wide concerns use middleware/handlers; for command/query pipelines use behaviors. Prefer these explicit, AOT-friendly mechanisms over dynamic-proxy AOP frameworks unless you specifically need them.

---

## Common gotchas

### Wrong decorator order

`Logging(Caching(Real))` logs every call; `Caching(Logging(Real))` logs only misses. Order the chain to match your intent.

### Decorator lifetime mismatch

A decorator's lifetime should generally match the inner service's. A singleton decorator wrapping a scoped inner is a captive dependency ([03-Lifetimes.md](03-Lifetimes.md)). Register decorators with the same lifetime as what they wrap.

### Decorating with the built-in container expecting `Decorate`

It doesn't exist out of the box — add Scrutor or use a factory. Calling a nonexistent `Decorate` won't compile without the package.

### Over-decorating

Five stacked decorators are hard to follow and debug. Keep the chain short; combine related concerns or use middleware/behaviors where they fit better.

### Forgetting the inner is injected, not `new`-ed

A decorator should receive the inner service via DI (so the inner's own dependencies resolve) — Scrutor/the factory handles this. Don't `new` the inner inside the decorator.

---

## Summary

- A **decorator** wraps a service in another implementation of the same interface to add cross-cutting behavior (caching, logging, retry, metrics) without changing the original.
- The built-in container has no `Decorate` — use **Scrutor's `services.Decorate<TInterface, TDecorator>()`** or a **manual factory** registration that builds the chain.
- **Order matters** (last decorate = outermost): `Logging(Caching(Real))` logs all calls vs `Caching(Logging(Real))` logs only misses — choose deliberately.
- Match decorator **lifetimes** to the wrapped service (avoid captive dependencies); keep chains short.
- Decorators are the DI-level cross-cutting tool; use **middleware** (HTTP), **`DelegatingHandler`** (outbound HTTP), or **MediatR behaviors** (CQRS) at their respective layers.

→ Next: [07-Configuration.md](07-Configuration.md)
