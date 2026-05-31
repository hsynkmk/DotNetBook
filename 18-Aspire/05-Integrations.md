# Integrations

## Backing services in one call

Real apps depend on databases, caches, message brokers, and cloud services. Wiring each one means: start it (locally), inject its connection string, register a client in DI, add health checks, add telemetry, add resilience — repetitive and easy to get inconsistent. Aspire **integrations** (formerly "components") collapse all of that into a couple of method calls. There are **two halves**: a **hosting integration** used in the AppHost to *provision/orchestrate* the resource, and a **client integration** used in a service to *consume* it with a fully-wired, instrumented client.

```csharp
// AppHost (hosting integration — orchestrate Postgres + Redis):
var db    = builder.AddPostgres("pg").AddDatabase("ordersdb");
var cache = builder.AddRedis("cache");
var api   = builder.AddProject<Projects.Api>("api").WithReference(db).WithReference(cache);

// Service (client integration — consume them):
builder.AddNpgsqlDataSource("ordersdb");   // registers a health-checked, instrumented data source
builder.AddRedisClient("cache");            // registers an instrumented IConnectionMultiplexer
```

---

## Hosting integrations (AppHost side)

In the AppHost ([02-AppHost.md](02-AppHost.md)), hosting integrations add a backing resource to the app graph — usually starting a **container** locally and exposing its connection info:

```csharp
var postgres = builder.AddPostgres("pg")        // runs a postgres container
    .WithDataVolume()                            // persist data across runs
    .WithPgAdmin();                              // also start pgAdmin for inspection
var ordersDb = postgres.AddDatabase("ordersdb"); // a database within that server

var redis = builder.AddRedis("cache").WithRedisCommander();
var rabbit = builder.AddRabbitMQ("broker").WithManagementPlugin();
var blobs  = builder.AddAzureStorage("storage").AddBlobs("files");
```

Each `Add...` returns a resource you can `WithReference` into services. Many integrations add **management UIs** (`WithPgAdmin`, `WithRedisCommander`, `WithManagementPlugin`) for local inspection. The hosting integration knows how to run the resource locally (a container) *and* how to represent it for deployment ([08-Deployment.md](08-Deployment.md)) — e.g., `AddAzureStorage` runs an emulator locally but provisions real Azure Storage in the cloud.

---

## Client integrations (service side)

In a service, the client integration registers a **ready-to-use client** in DI, configured from the injected connection string ([04-ServiceDiscovery.md](04-ServiceDiscovery.md)) and pre-wired with health checks, telemetry, and resilience:

```csharp
builder.AddNpgsqlDataSource("ordersdb");        // NpgsqlDataSource (or AddNpgsqlDbContext for EF Core)
builder.AddRedisClient("cache");                 // IConnectionMultiplexer
builder.AddRabbitMQClient("broker");             // IConnection
builder.AddAzureBlobClient("files");             // BlobServiceClient
```

```csharp
public class OrderRepo(NpgsqlDataSource db) {     // injected, instrumented client
    public async Task<int> CountAsync() {
        await using var cmd = db.CreateCommand("SELECT count(*) FROM orders");
        return Convert.ToInt32(await cmd.ExecuteScalarAsync());
    }
}
```

The argument (`"ordersdb"`, `"cache"`) is the **resource name** — it must match the AppHost ([04-ServiceDiscovery.md](04-ServiceDiscovery.md)). The single call gives you a client that's already: connected (via the injected string), **traced and metered** (OpenTelemetry — [Ch12](../12-Observability/README.md)), **health-checked** (the dependency's health shows on `/health`), and often **resilient**. That's the value: consistent, instrumented client wiring without boilerplate.

---

## What you get "for free" per integration

A client integration typically bundles:

| Concern | What the integration adds |
|---|---|
| **Client registration** | the right client type in DI (`NpgsqlDataSource`, `IConnectionMultiplexer`, …) |
| **Connection** | reads the injected connection string by name |
| **Telemetry** | traces + metrics for the client's operations ([Ch12](../12-Observability/README.md)) |
| **Health checks** | a check for the dependency, surfaced on `/health` ([Ch04 §16](../04-AspNetCore/16-HealthChecks.md)) |
| **Resilience / config** | sensible defaults, configurable via options |

So a database query shows up as a span in your distributed trace, a Redis outage flips the readiness probe, and metrics flow to the dashboard — all from `AddNpgsqlDataSource("ordersdb")`.

---

## Configuring integrations

Client integrations expose options (often bound from configuration — [Ch13](../13-Configuration/README.md)) for tuning:

```csharp
builder.AddNpgsqlDataSource("ordersdb", settings => {
    settings.DisableHealthChecks = false;
    settings.DisableTracing = false;
});
builder.AddRedisClient("cache", settings => settings.ConnectTimeout = 5000);
```

You can disable any of the bundled concerns (e.g., turn off a health check) or adjust client behavior, but the defaults are designed to be production-sensible.

---

## EF Core integration

For EF Core ([Ch05](../05-EFCore/README.md)), the integration registers the `DbContext` wired to the injected connection string, with health/telemetry:

```csharp
builder.AddNpgsqlDbContext<AppDbContext>("ordersdb");   // DbContext + connection + health + tracing
```

This replaces manual `AddDbContext` + connection-string plumbing; EF Core queries then appear in distributed traces automatically.

---

## Common gotchas

### Confusing hosting vs client integration

`AddPostgres` (AppHost — runs the server) is **not** `AddNpgsqlDataSource` (service — connects a client). You need the **hosting** integration to provision and the **client** integration to consume, tied by name. Using only one half leaves the wiring broken.

### Name mismatch

The client integration reads the connection string by name; if it doesn't match the AppHost resource name (and a `WithReference`), there's no connection string to read. Keep names consistent.

### Writing the connection string manually

For Aspire-managed resources, the AppHost injects the connection string — don't also put one in `appsettings.json` (it causes confusion/override surprises). Let the integration read the injected value.

### Expecting the local container to be production

Locally `AddPostgres` runs a container; in production the deployment maps it to a managed/real instance. Don't depend on the local container's specifics; the integration abstracts the environment difference.

### Disabling telemetry/health by habit

The bundled health checks and telemetry are the point — they give observability for free. Disable them only with reason; keeping them is what makes the dashboard and probes useful.

---

## Summary

- **Integrations** wire backing services with minimal code, in **two halves**: **hosting integrations** (AppHost — provision/orchestrate the resource, e.g., `AddPostgres`/`AddRedis`, often a container locally + management UIs) and **client integrations** (service — register a ready-to-use client, e.g., `AddNpgsqlDataSource`/`AddRedisClient`).
- A **client integration** registers the client in DI **connected** (via the injected connection string), **traced/metered** (OpenTelemetry), **health-checked**, and resilient — all from one call; the **resource name** is the contract tying it to the AppHost ([04-ServiceDiscovery.md](04-ServiceDiscovery.md)).
- Integrations exist for Postgres, Redis, RabbitMQ, Azure Storage/Service Bus/Cosmos, EF Core (`AddNpgsqlDbContext<T>`), and more; they're **configurable** (and concerns like health checks can be disabled).
- The hosting integration abstracts the **environment difference** — a local container in dev, a real/managed resource in production ([08-Deployment.md](08-Deployment.md)); don't confuse the two halves or mismatch their names.

→ Next: [06-Dashboard.md](06-Dashboard.md)
