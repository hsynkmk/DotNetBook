# The AppHost and the App Model

## Describing your distributed app in C#

The **AppHost** is the heart of Aspire: a small console project that describes your entire distributed application as a **graph of resources** — projects, containers, databases, caches, brokers — and their relationships, using a fluent C# API. Running the AppHost (`dotnet run` / F5) launches the whole graph locally, wired together, with the dashboard ([06-Dashboard.md](06-Dashboard.md)). The AppHost is the **single source of truth** for what your app is made of — replacing docker-compose files and manual launch scripts with type-safe, refactorable C#.

```csharp
var builder = DistributedApplication.CreateBuilder(args);

// Backing resources (containers)
var postgres = builder.AddPostgres("postgres");
var ordersDb = postgres.AddDatabase("ordersdb");
var cache    = builder.AddRedis("cache");
var broker   = builder.AddRabbitMQ("broker");

// Your services (projects), with references that wire them up
var api = builder.AddProject<Projects.OrderApi>("orderapi")
    .WithReference(ordersDb)
    .WithReference(cache);

builder.AddProject<Projects.OrderWorker>("orderworker")
    .WithReference(ordersDb)
    .WithReference(broker);

builder.Build().Run();
```

---

## `DistributedApplication.CreateBuilder`

The AppHost mirrors the familiar host-builder pattern ([Ch03 §01](../03-HostingAndDI/01-GenericHost.md)): `DistributedApplication.CreateBuilder(args)` returns a builder, you add resources to it, then `Build().Run()`. But instead of registering DI services, you're declaring **resources** in an *app model* — an in-memory graph that Aspire materializes (starting containers, launching projects, injecting configuration) when it runs.

---

## Resource types

The app model is built from resource kinds, each added with an `Add...` method:

| Resource | Method | What it is |
|---|---|---|
| **Project** | `AddProject<Projects.X>("name")` | one of *your* .NET services (API, worker) |
| **Container** | `AddContainer("name", "image")` | any Docker image (or a typed integration like `AddRedis`) |
| **Executable** | `AddExecutable(...)` | an external process (e.g., a Node app) |
| **Integration resource** | `AddPostgres`/`AddRedis`/`AddRabbitMQ`/… | a typed backing service ([05-Integrations.md](05-Integrations.md)) |
| **Parameter** | `AddParameter("name")` | a config/secret value ([07-ConfigAndSecrets.md](07-ConfigAndSecrets.md)) |

`Projects.X` is a **generated, strongly-typed reference** to a project in your solution (the AppHost references the service projects, and Aspire generates the `Projects.*` types) — so adding a project is type-safe and renames/refactors carry through.

---

## References — wiring the graph

The defining operation is **`WithReference`**: it declares that one resource *depends on* another, and Aspire uses that to (a) order startup and (b) **inject the connection information** (connection strings, service URLs) into the dependent ([04-ServiceDiscovery.md](04-ServiceDiscovery.md)):

```csharp
var api = builder.AddProject<Projects.OrderApi>("orderapi")
    .WithReference(ordersDb)    // injects the "ordersdb" connection string into orderapi
    .WithReference(cache)       // injects Redis connection info
    .WithReference(otherApi);   // injects the other service's URL for HTTP discovery
```

`WithReference(database/cache/broker)` injects the resource's **connection string** (named by the resource) into the consumer's configuration. `WithReference(anotherProject)` injects that service's **endpoint URL** for HTTP service discovery. This is how services find their dependencies **without any hardcoded URLs or connection strings** — the AppHost is the wiring diagram.

---

## Endpoints, wait-for, and lifecycle

The fluent API also expresses operational details:

```csharp
var api = builder.AddProject<Projects.OrderApi>("orderapi")
    .WithReference(ordersDb)
    .WaitFor(ordersDb)                       // don't start orderapi until Postgres is ready
    .WithHttpEndpoint(port: 5001)            // declare/customize an endpoint
    .WithReplicas(3);                        // run multiple instances locally

postgres.WithDataVolume();                   // persist DB data across runs (don't lose it on restart)
cache.WithLifetime(ContainerLifetime.Persistent);  // keep the container between runs for faster startup
```

- **`WaitFor`** orders startup so a service doesn't start before its dependency is healthy (avoiding connection-refused races on launch).
- **`WithHttpEndpoint`/`WithHttpsEndpoint`** declare/configure the ports services listen on.
- **`WithReplicas`** runs multiple instances (testing load-balanced/scaled behavior locally).
- **`WithDataVolume`/`WithBindMount`** persist container data; **`WithLifetime(Persistent)`** keeps containers between runs for faster dev loops.

---

## The dependency graph and startup

From all the `Add...`/`WithReference`/`WaitFor` calls, Aspire builds a **dependency graph**. On run, it:

1. Resolves startup order (dependencies before dependents).
2. Starts container resources (pulling images as needed), waits for readiness where `WaitFor` is set.
3. Launches your project resources, **injecting** the resolved connection strings/endpoints as configuration.
4. Starts the dashboard, attaching to every resource's logs/telemetry.

The result: one command brings up a coherent, wired, observable system — the same graph for every developer, no manual steps.

---

## Common gotchas

### Forgetting `WithReference`

Without `WithReference`, the consumer doesn't get the dependency's connection string/URL — service discovery has nothing to resolve, and the client integration ([05-Integrations.md](05-Integrations.md)) can't connect. The reference is what wires the graph.

### Startup races (no `WaitFor`)

A service may start before its database container is accepting connections, causing transient startup errors. Use `WaitFor` (and rely on resilience — [Ch11](../11-Resilience/README.md)) to order/retry.

### Losing data on restart

Database containers are ephemeral by default — data is gone on restart. Add `WithDataVolume()` to persist it across runs during development.

### Confusing the AppHost with production orchestration

The AppHost orchestrates **local development**. It generates deployment artifacts ([08-Deployment.md](08-Deployment.md)) but doesn't run your production cluster — don't treat `Run()` as your prod orchestrator.

### Hardcoding URLs/connection strings anyway

The whole point is injected, name-based wiring. Hardcoding a URL/connection string in a service bypasses service discovery and breaks portability across environments — reference resources instead.

---

## Summary

- The **AppHost** is a C# project that describes your whole distributed app as a **graph of resources** (projects, containers, integration resources, parameters) and runs the entire graph locally with one command — the type-safe replacement for docker-compose + launch scripts.
- It uses the host-builder pattern (`DistributedApplication.CreateBuilder` → add resources → `Build().Run()`), adding **resources** to an app model rather than DI services; projects are added via **strongly-typed `Projects.X`** references.
- **`WithReference`** wires the graph: it injects a resource's **connection string** (DB/cache/broker) or **endpoint URL** (another project) into the consumer — enabling **service discovery with no hardcoded URLs/connection strings** ([04-ServiceDiscovery.md](04-ServiceDiscovery.md)).
- Operational fluent calls — **`WaitFor`** (startup ordering), `WithHttpEndpoint`, `WithReplicas`, `WithDataVolume`/`WithLifetime` — control readiness, ports, scaling, and persistence; Aspire builds a **dependency graph** and starts everything in order, injecting config and attaching the dashboard.

→ Next: [03-ServiceDefaults.md](03-ServiceDefaults.md)
