# Chapter 18 — .NET Aspire — Q & A

---

### Q1. What is .NET Aspire in one sentence?

A cloud-native composition layer for .NET: you describe a multi-service app's graph (services + backing resources) in C# in an **AppHost**, and get service discovery, telemetry, health checks, resilience, and a live dashboard by convention — running locally with one command and deploying the same model to the cloud.

---

### Q2. What is Aspire *not*?

Not a runtime (your services run on the normal .NET runtime; Aspire composes/observes them), not Kubernetes/a production orchestrator (it generates deployment artifacts but doesn't run your prod cluster), and not a new framework (you still write ASP.NET Core/EF Core). It's the glue and control panel, not the engine.

---

### Q3. What are the two halves of an integration?

**Hosting integration** (in the AppHost — provisions/orchestrates a resource, e.g., `AddPostgres`/`AddRedis`, usually a container locally) and **client integration** (in a service — registers a ready-to-use, instrumented client, e.g., `AddNpgsqlDataSource`/`AddRedisClient`). The AppHost provisions; the service consumes; the resource **name** ties them together.

---

### Q4. What is the AppHost?

A console project that describes the whole distributed app as a **graph of resources** (projects, containers, integration resources, parameters) using a fluent C# API (`DistributedApplication.CreateBuilder` → add resources → `Build().Run()`), and runs the entire graph locally with one command. It's the type-safe replacement for docker-compose + launch scripts.

---

### Q5. What does `WithReference` do?

Declares a dependency and injects the dependency's connection info into the consumer: for a DB/cache/broker, the **connection string**; for another project, the **endpoint URL** (HTTP discovery). It's what wires the graph and enables service discovery with no hardcoded URLs/connection strings.

---

### Q6. What does `WaitFor` do and why use it?

Orders startup so a service doesn't start until its dependency is healthy — avoiding connection-refused races on launch (e.g., the API starting before Postgres is accepting connections).

---

### Q7. What is ServiceDefaults and what does `AddServiceDefaults()` wire?

A shared project (scaffolded as source you own) whose `AddServiceDefaults()` wires four cross-cutting concerns into every service by convention: **OpenTelemetry** (traces/metrics/logs), **health checks**, **HTTP resilience** (standard handler), and **service discovery**. Call it in each service's `Program.cs`.

---

### Q8. What does `MapDefaultEndpoints()` expose?

Health endpoints: **`/health`** (readiness — are dependencies healthy, route traffic?) and **`/alive`** (liveness — is the process up, restart?). Matches the orchestrator liveness/readiness model that Kubernetes/Container Apps consume.

---

### Q9. How does service discovery work in Aspire?

Services use **logical names** (`http://orderapi`) instead of URLs. The AppHost's `WithReference` injects endpoint/connection config per environment; ServiceDefaults' discovery client rewrites `http://name` to the real endpoint. The *same code* resolves to local ports in dev and real addresses in prod — no code change.

---

### Q10. How do services get connection strings for databases?

The AppHost injects them (e.g., `ConnectionStrings__ordersdb=...`) when it launches the service, named for the resource. The service's client integration reads it by name (`AddNpgsqlDataSource("ordersdb")`). You never write the connection string in the service's `appsettings.json` for an Aspire-managed resource.

---

### Q11. What does a client integration give you "for free"?

A client registered in DI that's **connected** (via injected connection string), **traced and metered** (OpenTelemetry), **health-checked** (dependency health on `/health`), and resilient — from one call. So a DB query appears in distributed traces and a dependency outage flips the readiness probe automatically.

---

### Q12. Why does an Aspire app have distributed tracing "for free"?

Because every service calls `AddServiceDefaults()`, which configures OpenTelemetry identically and exports via OTLP to the dashboard (and your APM in prod). A request flowing API → worker → DB shows as one correlated trace across services.

---

### Q13. What does the developer dashboard show?

The running app graph: resource state/health, console + structured logs, **distributed traces**, metrics, and the injected env vars/connection strings — live, with zero setup. Structured logs are trace-correlated; the Traces view shows a request as one waterfall across services.

---

### Q14. Is the dev dashboard your production monitoring?

No. It's the **development** observability surface (in-memory, current run). In production you export the same OpenTelemetry to a real APM (Application Insights, Grafana/Tempo/Prometheus, Datadog). Same instrumentation shape, different (durable, scalable) backend.

---

### Q15. What is a parameter (`AddParameter`)?

A named external config value (with `secret: true` for sensitive ones) threaded through the app model — used for environment-specific or secret values (e.g., a DB password, an API key). Its value comes from the AppHost's config (user secrets locally, a secret store in prod).

---

### Q16. How are secrets handled locally vs in production?

Locally, secret parameter values come from the AppHost's **user secrets** (`dotnet user-secrets`, outside the repo). In production, they map to the platform's **secret store** (Key Vault / Container Apps secrets / Kubernetes secrets) via the deployment manifest. Same declaration, different backing — never committed.

---

### Q17. How do you express per-environment differences?

Via parameters whose values differ per environment, and by branching on `builder.Environment`/`ExecutionContext` in the AppHost — e.g., run a local **container** (`AddRedis`) in dev but reference a managed resource (`AddConnectionString("cache")`) in prod.

---

### Q18. What is the deployment manifest?

A platform-neutral JSON description of the app model (resources, endpoints, connection strings, parameters, references) that Aspire emits. Deployment tools (publishers) consume it to provision and wire real infrastructure — so the same described app can target Azure, Kubernetes, etc.

---

### Q19. How do you deploy an Aspire app to Azure?

With the **Azure Developer CLI**: `azd init` then `azd up`. It reads the app model, **provisions managed Azure services** (your `AddPostgres`→Azure DB, `AddRedis`→Azure Cache), **containerizes and deploys** projects to **Azure Container Apps**, and wires connection strings, discovery, and secrets. The local container becomes a real managed resource.

---

### Q20. Is Aspire deployment Azure-only?

No. Azure (`azd`→Container Apps) is the most polished path, but the manifest is platform-neutral; Kubernetes (e.g., Aspir8/`aspirate`) and other targets are supported via publishers. The app model is unchanged — only the target differs.

---

### Q21. What maps to what between dev and prod?

Projects → containers in Container Apps; `AddPostgres`/`AddRedis` (local containers) → managed Azure DB/Cache; connection strings → injected from managed endpoints; secret parameters → Container Apps secrets/Key Vault; discovery → platform addressing. Backing differs, **code doesn't**.

---

### Q22. What is `Aspire.Hosting.Testing` for?

Spinning up the **entire Aspire app graph** (real AppHost + real containerized dependencies) in an integration test via `DistributedApplicationTestingBuilder.CreateAsync<Projects.AppHost>()`, then exercising services through discovery-wired HTTP clients — the highest-fidelity test of **cross-service flows**.

---

### Q23. How does Aspire testing relate to WebApplicationFactory?

Complementary, different scope. `WebApplicationFactory` tests **one** ASP.NET Core app in-memory (fast, deps substituted). `Aspire.Hosting.Testing` tests the **whole graph with real dependencies** (slower, highest fidelity). Use the former broadly for single-service pipelines, the latter sparingly for distributed flows.

---

### Q24. How does Aspire relate to the rest of the .NET stack?

It's a composition layer *over* Hosting & DI ([Ch03](../03-HostingAndDI/README.md)), Observability ([Ch12](../12-Observability/README.md)), Resilience ([Ch11](../11-Resilience/README.md)), Configuration ([Ch13](../13-Configuration/README.md)), and Data/Caching ([Ch06](../06-DataAndCaching/README.md)), feeding Deployment ([Ch19](../19-Deployment/README.md)). It doesn't replace them — it makes them work together with minimal ceremony.

---

### Q25. When should you adopt Aspire (and when not)?

Adopt for **multi-service apps** (API + worker + DB + cache + broker), microservices, anything juggling docker-compose + manual wiring, teams wanting consistent observability/resilience, and Azure-targeted apps — and it can be added incrementally. Less necessary for a single self-contained service with no backing resources or a satisfactory existing orchestration.
