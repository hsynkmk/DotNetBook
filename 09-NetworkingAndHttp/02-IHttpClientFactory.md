# IHttpClientFactory

## The right way to create HttpClients

`IHttpClientFactory` is the recommended way to obtain `HttpClient` instances. It solves two problems that plague naive `HttpClient` usage: **socket exhaustion** (from creating clients per request) and **stale DNS** (from a single never-rotated static client). It manages the underlying handler pool, integrates with DI, and is the foundation for resilience and cross-cutting handlers.

```csharp
builder.Services.AddHttpClient();   // register the factory

public class ProductClient(IHttpClientFactory factory) {
    public async Task<Product?> GetAsync(int id, CancellationToken ct) {
        HttpClient client = factory.CreateClient();        // get a client (cheap)
        return await client.GetFromJsonAsync<Product>($"/products/{id}", ct);
    }
}
```

---

## The two problems it solves

### Socket exhaustion (don't `new HttpClient()` per request)

```csharp
// ✗ — a new HttpClient per request leaks sockets
using (var client = new HttpClient()) {        // disposes → socket lingers in TIME_WAIT
    await client.GetAsync(url);                  // thousands of requests → port exhaustion → failures
}
```

Each `HttpClient` (with its own handler) opens connections; disposing it leaves sockets in `TIME_WAIT` for a while. Under load, you exhaust the ephemeral port range → "Only one usage of each socket address" / connection failures. This is the **classic .NET HTTP bug**.

### Stale DNS (the static-singleton fix has its own flaw)

```csharp
// ✗ — a single static HttpClient never rotates connections, so it doesn't pick up DNS changes
private static readonly HttpClient Client = new();   // a service's IP changes → you keep hitting the old one
```

The "fix" of one long-lived static client avoids socket exhaustion but **caches connections forever**, so it ignores DNS changes (a problem in cloud environments where service IPs move).

`IHttpClientFactory` solves **both**: it pools and reuses handlers (no socket exhaustion) **and** rotates them periodically (`PooledConnectionLifetime` / handler lifetime) so DNS changes are picked up. You get a fresh, lightweight `HttpClient` per `CreateClient()` call, but the expensive **handler** underneath is pooled and rotated.

---

## How it works: client vs handler lifetime

The key insight: a `HttpClient` is cheap; its **`HttpMessageHandler`** (which holds the connection pool) is expensive. The factory separates their lifetimes:

```
CreateClient()  → a NEW lightweight HttpClient (cheap, short-lived, don't pool it)
                  wrapping a POOLED HttpMessageHandler (expensive, reused across clients)
                  → handlers are ROTATED every ~2 minutes (HandlerLifetime) to honor DNS
```

So you call `CreateClient()` freely (it's cheap), and the factory reuses the underlying handler (no socket exhaustion) while rotating it periodically (no stale DNS). You **don't** dispose or cache the returned `HttpClient` — get a fresh one when you need it; the factory manages the handler.

```csharp
builder.Services.AddHttpClient("api")
    .SetHandlerLifetime(TimeSpan.FromMinutes(5));   // how often the underlying handler rotates
```

---

## Named clients

Configure clients by name with a base address, default headers, etc.:

```csharp
builder.Services.AddHttpClient("github", client => {
    client.BaseAddress = new Uri("https://api.github.com/");
    client.DefaultRequestHeaders.Add("User-Agent", "MyApp");
    client.Timeout = TimeSpan.FromSeconds(30);
});

public class GitHubService(IHttpClientFactory factory) {
    public async Task<Repo[]> GetReposAsync(string org, CancellationToken ct) {
        var client = factory.CreateClient("github");        // pre-configured client
        return await client.GetFromJsonAsync<Repo[]>($"orgs/{org}/repos", ct) ?? [];
    }
}
```

Named clients centralize per-endpoint configuration (base URL, headers, timeout, handlers). Each name has its own handler pool and configuration.

---

## Typed clients (the recommended style)

A **typed client** binds a configured `HttpClient` to a strongly-typed service class — the cleanest, most testable approach:

```csharp
public class GitHubClient(HttpClient http) {                 // HttpClient injected by the factory
    public async Task<Repo[]> GetReposAsync(string org, CancellationToken ct) =>
        await http.GetFromJsonAsync<Repo[]>($"orgs/{org}/repos", ct) ?? [];
}

builder.Services.AddHttpClient<GitHubClient>(client => {       // configures + registers the typed client
    client.BaseAddress = new Uri("https://api.github.com/");
    client.DefaultRequestHeaders.Add("User-Agent", "MyApp");
});

// Inject GitHubClient directly — its HttpClient is factory-managed
public class ReposController(GitHubClient github) { ... }
```

Typed clients give you: a clean API surface (methods like `GetReposAsync`, not raw `HttpClient` calls scattered around), encapsulated configuration, easy unit testing (mock the typed client or its `HttpClient`), and all the factory's pooling/rotation. **Prefer typed clients** for code that talks to a specific API.

---

## Integrating handlers and resilience

The factory is the integration point for **delegating handlers** (cross-cutting concerns — [03-DelegatingHandlers.md](03-DelegatingHandlers.md)) and **resilience** (retry/circuit breaker — [04-Resilience.md](04-Resilience.md)):

```csharp
builder.Services.AddHttpClient<GitHubClient>(...)
    .AddHttpMessageHandler<AuthHandler>()                 // add auth (a DelegatingHandler)
    .AddStandardResilienceHandler();                       // .NET 8+: retry, circuit breaker, timeout (Polly)
```

This is why the factory matters beyond lifetime management: it composes the handler pipeline (auth, logging, retry, metrics) per client. Configuring resilience/handlers requires the factory — you can't cleanly add them to a raw `new HttpClient()`.

---

## Common gotchas

### `new HttpClient()` per request

Socket exhaustion under load. Use the factory (`CreateClient`/typed clients). The reason `IHttpClientFactory` exists.

### Caching the `CreateClient()` result long-term

The returned `HttpClient` is meant to be short-lived (the factory manages the pooled handler). Caching it for the app's lifetime can defeat handler rotation. Get a fresh one per operation (or use a typed client, injected fresh).

### Disposing the factory-provided `HttpClient`

Don't — the factory owns the handler lifecycle. Disposing the client doesn't dispose the pooled handler (and isn't necessary). Just let it go.

### Manually setting up handler lifetime on a static client

Reinventing what the factory does (handler rotation for DNS). Use `IHttpClientFactory`, which handles it.

### Mixing typed client config across names

Each typed/named client has its own configuration and handler pool. Configuring one doesn't affect another. Register each deliberately.

### Forgetting `User-Agent` / required headers

Some APIs reject requests without certain headers. Set defaults on the named/typed client so every request includes them.

---

## Summary

- **`IHttpClientFactory`** (`AddHttpClient`) is the recommended way to obtain `HttpClient`s — it fixes **socket exhaustion** (don't `new` per request) **and** **stale DNS** (the static-singleton flaw) by pooling and **rotating** the underlying handler.
- It separates lifetimes: `CreateClient()` returns a **cheap, short-lived `HttpClient`** wrapping a **pooled, rotated handler** — call it freely, don't cache or dispose the client.
- **Named clients** centralize per-endpoint config (base URL, headers, timeout); **typed clients** (`AddHttpClient<T>`) bind config to a strongly-typed service class — the cleanest, most testable style (prefer these).
- The factory is the integration point for **delegating handlers** ([03](03-DelegatingHandlers.md)) and **resilience** ([04](04-Resilience.md)) — composing auth/logging/retry/metrics per client.

→ Next: [03-DelegatingHandlers.md](03-DelegatingHandlers.md)
