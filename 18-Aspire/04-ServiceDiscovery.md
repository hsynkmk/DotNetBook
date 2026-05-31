# Service Discovery

## Finding services by name, not by URL

In a distributed app, services call each other and connect to backing resources — but hardcoding URLs (`http://localhost:5001`) and connection strings is brittle: ports differ between dev and prod, containers get dynamic addresses, and every environment change means editing config. **Service discovery** replaces hardcoded addresses with **logical names**: a service calls `http://orderapi` and the discovery system resolves that to the real endpoint at runtime. Aspire makes this automatic — the AppHost's `WithReference` ([02-AppHost.md](02-AppHost.md)) injects the resolution data, and ServiceDefaults' discovery client ([03-ServiceDefaults.md](03-ServiceDefaults.md)) resolves names on `HttpClient` calls.

```csharp
// Caller service — uses the logical name, not a URL:
builder.Services.AddHttpClient<OrderClient>(c => c.BaseAddress = new("http://orderapi"));
// "http://orderapi" resolves to the real endpoint via service discovery — no hardcoded port/host.
```

---

## How it works end to end

The flow ties together the AppHost and ServiceDefaults:

1. **AppHost declares the dependency**: `consumer.WithReference(orderApi)` records that the consumer depends on `orderapi` ([02-AppHost.md](02-AppHost.md)).
2. **Aspire injects endpoint config**: when it launches the consumer, it sets configuration like `services__orderapi__http__0 = http://localhost:5001` (the resolved address of `orderapi`).
3. **The discovery client resolves names**: ServiceDefaults registers `AddServiceDiscovery()` and applies it to `HttpClient`s, so a request to `http://orderapi` is rewritten to the injected real endpoint.

So the consumer's code only knows the **name** `orderapi`; the actual host/port comes from configuration that Aspire populates per environment. Change the environment (dev → prod), and the *same code* resolves to a different real address — no edits.

---

## Connection strings for backing resources

The same mechanism injects **connection strings** for databases/caches/brokers. When the AppHost does `api.WithReference(ordersDb)`, Aspire injects a connection string named for the resource into the consumer's configuration:

```
ConnectionStrings__ordersdb = Host=localhost;Port=5432;Database=ordersdb;Username=...;Password=...
```

The service's **client integration** ([05-Integrations.md](05-Integrations.md)) reads it by that name:

```csharp
builder.AddNpgsqlDataSource("ordersdb");   // reads ConnectionStrings:ordersdb (injected by AppHost)
builder.AddRedisClient("cache");            // reads the "cache" connection info
```

The string `"ordersdb"`/`"cache"` is the contract: it must match the resource name in the AppHost. This is why you never write a connection string in a service's `appsettings.json` for an Aspire-managed resource — the AppHost supplies it.

---

## Named endpoints

A resource can expose multiple endpoints (HTTP and HTTPS, or several named ports). Service discovery supports scheme/name selection:

```csharp
// Prefer https, fall back to http:
c.BaseAddress = new("https+http://orderapi");

// A specific named endpoint:
c.BaseAddress = new("http://_admin.orderapi");   // the "admin" endpoint of orderapi
```

`scheme://_endpointname.servicename` selects a particular named endpoint; `https+http://` expresses a scheme preference order. This lets a service target the right endpoint of a multi-endpoint dependency by name.

---

## Why no hardcoded URLs/connection strings

The payoff is **environment portability**:

- **Local dev**: names resolve to the local container/project ports Aspire assigned.
- **Production**: the deployment ([08-Deployment.md](08-Deployment.md)) maps the same names to real service addresses / managed-resource connection strings (often via platform service discovery or injected env vars).
- **Tests** ([09-TestingAspire.md](09-TestingAspire.md)): the test host resolves names to the test instances.

The *code never changes* — only the injected configuration does. This is the core benefit: services are written against logical names, and the environment supplies the bindings. It also makes refactoring safe (rename a resource in the AppHost, update references) and eliminates the classic "wrong connection string in the wrong environment" class of bugs.

---

## Relationship to platform service discovery

Aspire's `Microsoft.Extensions.ServiceDiscovery` is a **client-side** abstraction that reads endpoints from configuration. In production it composes with the platform: on Kubernetes, names can resolve via DNS/Kubernetes services; on Azure Container Apps, via the platform's service-to-service addressing. Aspire's model is a thin, consistent layer that works locally (config-injected) and defers to platform discovery in the cloud — so the same `http://orderapi` works everywhere, backed by different resolution underneath.

---

## Common gotchas

### Hardcoding a URL/connection string anyway

Writing `http://localhost:5001` or a literal connection string defeats discovery and breaks across environments. Use the **logical name** (`http://orderapi`, `ConnectionStrings:ordersdb`) and let Aspire inject the real value.

### Name mismatch between AppHost and consumer

The consumer's client integration reads by name (`AddRedisClient("cache")`); if the AppHost resource is named differently (or not referenced), resolution fails. The **names must match** and the dependency must be `WithReference`d.

### Forgetting `AddServiceDiscovery` (no ServiceDefaults)

HTTP name resolution comes from the discovery client wired by ServiceDefaults. A service that skips `AddServiceDefaults()` won't resolve `http://servicename` — calls fail.

### Expecting discovery without a reference

`WithReference` is what injects the endpoint/connection config. Without it, there's nothing for the discovery client to resolve — the name has no binding.

### Assuming a specific port

Aspire assigns ports dynamically; relying on a fixed port is fragile. Resolve by name; if you need a fixed port for an external reason, set it explicitly in the AppHost (`WithHttpEndpoint(port: ...)`).

---

## Summary

- **Service discovery** replaces hardcoded URLs/connection strings with **logical names**: a service calls `http://orderapi` (or reads `ConnectionStrings:ordersdb`) and the real address is resolved at runtime from config Aspire injects.
- The flow: AppHost **`WithReference`** records the dependency → Aspire **injects** endpoint/connection-string config per environment → ServiceDefaults' **discovery client** rewrites `http://name` to the real endpoint; **client integrations** read connection strings by the resource **name**.
- Supports **named endpoints** (`scheme://_endpoint.service`) and **scheme preference** (`https+http://`); the resource **name is the contract** between AppHost and consumer.
- The benefit is **environment portability** — the same code resolves to local ports, production addresses, or test instances with **no code change**, just different injected config; in production it composes with **platform discovery** (Kubernetes DNS, Container Apps).

→ Next: [05-Integrations.md](05-Integrations.md)
