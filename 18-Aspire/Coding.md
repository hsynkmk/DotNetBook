# Chapter 18 — .NET Aspire — Coding Problems

Build an AppHost wiring an API + worker + Postgres + Redis, consume integrations, and write an Aspire integration test. Each problem has a hidden solution — attempt it first.

---

### Problem 1 — A minimal AppHost with two services

Write an AppHost that runs an API project and a worker project.

<details>
<summary>Solution</summary>

```csharp
var builder = DistributedApplication.CreateBuilder(args);

builder.AddProject<Projects.OrderApi>("orderapi");
builder.AddProject<Projects.OrderWorker>("orderworker");

builder.Build().Run();
```

`DistributedApplication.CreateBuilder` → add resources → `Build().Run()`. `Projects.X` are generated strongly-typed references to projects the AppHost references.
</details>

---

### Problem 2 — Add Postgres and Redis and wire them to the API

Extend the AppHost: a Postgres database `ordersdb` and a Redis cache, both referenced by the API; the worker references only the database.

<details>
<summary>Solution</summary>

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var db    = builder.AddPostgres("pg").WithDataVolume().AddDatabase("ordersdb");
var cache = builder.AddRedis("cache");

builder.AddProject<Projects.OrderApi>("orderapi")
    .WithReference(db)
    .WithReference(cache)
    .WaitFor(db);

builder.AddProject<Projects.OrderWorker>("orderworker")
    .WithReference(db)
    .WaitFor(db);

builder.Build().Run();
```

`WithReference` injects connection strings; `WaitFor(db)` orders startup so services don't start before Postgres is ready; `WithDataVolume` persists data across runs.
</details>

---

### Problem 3 — Consume the database with a client integration

In the API, register an EF Core `DbContext` connected to the injected `ordersdb` connection string.

<details>
<summary>Solution</summary>

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.AddServiceDefaults();                       // telemetry, health, resilience, discovery
builder.AddNpgsqlDbContext<OrdersDbContext>("ordersdb");   // reads injected ConnectionStrings:ordersdb

var app = builder.Build();
app.MapDefaultEndpoints();
app.MapGet("/orders", (OrdersDbContext db) => db.Orders.ToListAsync());
app.Run();
```

`AddNpgsqlDbContext<T>("ordersdb")` wires the DbContext to the AppHost-injected connection string, with health checks + tracing — no manual connection string in `appsettings.json`. The name must match the AppHost resource.
</details>

---

### Problem 4 — Consume Redis

Register a Redis client in the API and use it.

<details>
<summary>Solution</summary>

```csharp
builder.AddRedisClient("cache");     // registers IConnectionMultiplexer, instrumented + health-checked
```
```csharp
app.MapGet("/cached/{key}", async (string key, IConnectionMultiplexer redis) => {
    var db = redis.GetDatabase();
    return await db.StringGetAsync(key);
});
```

`AddRedisClient("cache")` matches the AppHost's `AddRedis("cache")` by name; the client is connected via the injected config, traced, and health-checked.
</details>

---

### Problem 5 — Add ServiceDefaults to every service

Show the two lines every service needs and what they provide.

<details>
<summary>Solution</summary>

```csharp
builder.AddServiceDefaults();   // OpenTelemetry + health checks + HTTP resilience + service discovery
// ...
app.MapDefaultEndpoints();      // /health (readiness) and /alive (liveness)
```

`AddServiceDefaults()` wires the four cross-cutting concerns by convention; `MapDefaultEndpoints()` exposes the health endpoints the orchestrator uses. Both belong in every service.
</details>

---

### Problem 6 — Service-to-service call via discovery

The worker calls the API over HTTP using the logical name, not a URL.

<details>
<summary>Solution</summary>

```csharp
// AppHost: let the worker discover the API
var api = builder.AddProject<Projects.OrderApi>("orderapi").WithReference(db);
builder.AddProject<Projects.OrderWorker>("orderworker")
    .WithReference(api);                         // injects orderapi's endpoint for discovery
```
```csharp
// Worker:
builder.Services.AddHttpClient<ApiClient>(c => c.BaseAddress = new("https+http://orderapi"));
```

`WithReference(api)` injects the API's endpoint; the discovery client (from ServiceDefaults) resolves `https+http://orderapi` to the real address — no hardcoded URL/port.
</details>

---

### Problem 7 — A secret parameter

Add a secret `payment-api-key` parameter and pass it to the API as an env var.

<details>
<summary>Solution</summary>

```csharp
var apiKey = builder.AddParameter("payment-api-key", secret: true);

builder.AddProject<Projects.OrderApi>("orderapi")
    .WithReference(db)
    .WithEnvironment("PaymentApiKey", apiKey);
```
```bash
# Local value (AppHost user secrets — never committed):
dotnet user-secrets set "Parameters:payment-api-key" "sk_live_xxx"
```

`secret: true` masks it in the dashboard and maps it to a secret store in deployment. The value comes from user secrets locally, a secret store in prod.
</details>

---

### Problem 8 — Environment-conditional resource

Run a local Redis container in development, but reference a managed cache in production.

<details>
<summary>Solution</summary>

```csharp
var cache = builder.ExecutionContext.IsRunMode && builder.Environment.IsDevelopment()
    ? builder.AddRedis("cache")                  // local container in dev
    : builder.AddConnectionString("cache");      // externally-provided managed cache in prod

builder.AddProject<Projects.OrderApi>("orderapi").WithReference(cache);
```

`AddConnectionString("cache")` references an external connection string (from config/deployment) instead of provisioning a container — keeping the app model portable. The consumer code is unchanged either way.
</details>

---

### Problem 9 — pgAdmin / Redis Commander for inspection

Add management UIs for the database and cache for local debugging.

<details>
<summary>Solution</summary>

```csharp
var db    = builder.AddPostgres("pg").WithDataVolume().WithPgAdmin();
var cache = builder.AddRedis("cache").WithRedisCommander();
```

`WithPgAdmin()`/`WithRedisCommander()` start companion management containers, visible in the dashboard's Resources view — handy for inspecting data locally without separate tooling.
</details>

---

### Problem 10 — Read a distributed trace (conceptual)

A `POST /orders` is slow. Describe how you'd diagnose it using the dashboard, and what you'd look for.

<details>
<summary>Solution</summary>

Open the dashboard's **Traces** view and find the `POST /orders` trace. It shows one correlated waterfall across services:

```
POST /orders                       [=====================] 240ms
  orderapi: POST /orders           [==================]    210ms
    → ordersdb: INSERT             [===]                    18ms
    → cache: SET                   [=]                       4ms
    → broker: publish              [==]                     12ms
  orderworker: handle OrderPlaced  [=========]              95ms
    → ordersdb: UPDATE inventory   [====]                   40ms
```

Look for the **longest span** — if `orderworker`'s inventory UPDATE dominates, that's the bottleneck; if a span errored, it's flagged. Click a span for attributes and its correlated logs. This pinpoints *which* service/operation is slow across the whole chain — for free, because ServiceDefaults instruments every service identically ([06-Dashboard.md](06-Dashboard.md)).
</details>

---

### Problem 11 — Write an Aspire integration test

Spin up the whole app graph and assert `POST /orders` returns 201.

<details>
<summary>Solution</summary>

```csharp
public class OrderFlowTests {
    [Fact]
    public async Task Placing_order_returns_created() {
        var appHost = await DistributedApplicationTestingBuilder.CreateAsync<Projects.AppHost>();
        await using var app = await appHost.BuildAsync();
        await app.StartAsync();                              // brings up services + Postgres + Redis

        var client = app.CreateHttpClient("orderapi");       // discovery-wired client
        var resp = await client.PostAsJsonAsync("/orders", new { Total = 100 });

        resp.StatusCode.Should().Be(HttpStatusCode.Created);
    }
}
```

Uses the **real AppHost** and real containerized dependencies (Docker required); `CreateHttpClient("orderapi")` resolves via service discovery. Highest-fidelity cross-service test ([09-TestingAspire.md](09-TestingAspire.md)).
</details>

---

### Problem 12 — Choose the test type

For each, pick `WebApplicationFactory` or `Aspire.Hosting.Testing` and justify: (a) test the API's validation logic on `POST /orders`, (b) test that placing an order persists it AND the worker processes the resulting message.

<details>
<summary>Solution</summary>

- **(a) API validation → `WebApplicationFactory`**: it's a single-service pipeline concern (routing/binding/validation). Fast, in-memory, substitute external deps — no need to start the whole graph ([Ch17 §05](../17-Testing/05-IntegrationTests.md)).
- **(b) Order persists + worker processes message → `Aspire.Hosting.Testing`**: this is a **cross-service flow** (API → DB → broker → worker → DB) needing real dependencies. Start the full graph and assert the end-to-end effect ([09-TestingAspire.md](09-TestingAspire.md)).

Keep many fast single-service tests and fewer full-graph tests ([Ch17 §10](../17-Testing/10-TestStrategy.md)).
</details>

---

### Problem 13 — Spot the wiring bug

```csharp
// AppHost:
var cache = builder.AddRedis("redis");
builder.AddProject<Projects.Api>("api");          // no reference to cache

// Api:
builder.AddRedisClient("cache");                   // different name
```

Two problems prevent the API from reaching Redis. Identify and fix.

<details>
<summary>Solution</summary>

1. **Name mismatch**: the AppHost names it `"redis"` but the client reads `"cache"` — no connection string by that name. Make them match.
2. **Missing `WithReference`**: the API doesn't reference the cache, so no connection string is injected at all.

Fix:
```csharp
var cache = builder.AddRedis("cache");
builder.AddProject<Projects.Api>("api").WithReference(cache);   // inject the connection string
// Api: builder.AddRedisClient("cache");   // name now matches
```

The resource name is the contract, and `WithReference` is what injects the connection info ([04-ServiceDiscovery.md](04-ServiceDiscovery.md), [05-Integrations.md](05-Integrations.md)).
</details>

---

### Problem 14 — Deploy to Azure (conceptual)

Describe the steps and what happens to your local Postgres/Redis containers.

<details>
<summary>Solution</summary>

```bash
azd init     # detects the Aspire AppHost
azd up       # provisions Azure infra + builds/deploys services
```

`azd up` reads the app model and: **provisions** managed Azure services (your `AddPostgres` → **Azure Database for PostgreSQL**, `AddRedis` → **Azure Cache for Redis**), **containerizes and deploys** your projects to **Azure Container Apps**, and wires **connection strings, service discovery, and secrets** (secret parameters → Container Apps secrets / Key Vault).

The local Postgres/Redis **containers are replaced by managed Azure resources** — the same app model expresses both; your service code (logical names + injected config) is unchanged ([08-Deployment.md](08-Deployment.md)).
</details>

---

### Problem 15 — Generate the deployment manifest

Produce the platform-neutral manifest from the AppHost.

<details>
<summary>Solution</summary>

```bash
dotnet run --project AppHost -- --publisher manifest --output-path manifest.json
```

This emits a JSON description of the app model (resources, endpoints, connection strings, parameters, references). Deployment tools/publishers consume it to target Azure (`azd`), Kubernetes (e.g., Aspir8), or other platforms — the app model stays the same; only the target differs ([08-Deployment.md](08-Deployment.md)).
</details>
