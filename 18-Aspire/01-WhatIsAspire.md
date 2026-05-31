# What Is .NET Aspire

## The cloud-native composition layer for .NET

A modern .NET application is rarely one process. It's an API *and* a background worker *and* a PostgreSQL database *and* a Redis cache *and* maybe a message broker — all of which must find each other, share configuration and connection strings, emit consistent telemetry, and report health. Historically you stitched this together by hand: a `docker-compose.yml`, hand-copied connection strings, ad-hoc OpenTelemetry setup in each service, and a prayer that everyone's environment matched. **.NET Aspire** makes that composition **first-class**: you describe your app's graph of services and backing resources **in C#**, and Aspire gives you service discovery, telemetry, health checks, resilience, and a live dashboard *by convention* — running the whole system locally with one command and deploying the same model to the cloud.

```csharp
// AppHost — the whole distributed app, described in C#
var builder = DistributedApplication.CreateBuilder(args);

var db    = builder.AddPostgres("pg").AddDatabase("orders");
var cache = builder.AddRedis("cache");

var api = builder.AddProject<Projects.Api>("api")
    .WithReference(db).WithReference(cache);     // api gets connection strings + discovery

builder.AddProject<Projects.Worker>("worker")
    .WithReference(db);

builder.Build().Run();   // one command runs API + worker + Postgres + Redis + dashboard
```

Aspire is GA since .NET 9 and matured in .NET 10 — increasingly the default way to build distributed .NET apps.

---

## What Aspire *is*

- **An app model / orchestrator (for development)** — you declare the resources (projects, containers, databases, caches, brokers) and their relationships in an **AppHost** project ([02-AppHost.md](02-AppHost.md)); Aspire launches and wires them together for local development (`F5`/`dotnet run` brings up the whole graph).
- **A set of conventions** — **ServiceDefaults** ([03-ServiceDefaults.md](03-ServiceDefaults.md)) wires OpenTelemetry, health checks, resilience, and service discovery into every service consistently, so you don't hand-roll them per service.
- **Integrations** ([05-Integrations.md](05-Integrations.md)) — one-call client wiring for common backing services (Postgres, Redis, RabbitMQ, Azure Service Bus, Cosmos…) that registers an instrumented, health-checked, resilient client.
- **A developer dashboard** ([06-Dashboard.md](06-Dashboard.md)) — live logs, **distributed traces**, metrics, and resource state out of the box.
- **A deployment model** ([08-Deployment.md](08-Deployment.md)) — the same app model emits a manifest that tools (`azd`, Kubernetes generators) turn into real cloud infrastructure.

---

## What Aspire is *not*

This is crucial to avoid misunderstanding it:

- **Not a runtime** — Aspire doesn't run your code; your services are ordinary ASP.NET Core/worker apps on the normal .NET runtime ([Ch01](../01-Runtime/README.md)). Aspire *composes and observes* them.
- **Not Kubernetes / not a production orchestrator** — the AppHost orchestrates your app **for local development**. In production, your services run on real platforms (Kubernetes, Azure Container Apps, App Service); Aspire *generates* the deployment artifacts but isn't the runtime orchestrator itself ([08-Deployment.md](08-Deployment.md)).
- **Not a new framework** — you still write normal ASP.NET Core, EF Core, etc. Aspire is a thin composition/convention layer *over* the stack you already know, not a replacement for it.
- **Not magic infrastructure** — it doesn't replace your database or message broker; it *wires you to* them (a real Postgres container locally, a managed Postgres in the cloud).

The mental model: **Aspire is the glue and the control panel, not the engine.** Your services and resources are real; Aspire makes them easy to compose, configure, observe, and deploy together.

---

## The problems it solves

| Pain (the old way) | Aspire's answer |
|---|---|
| docker-compose + manual wiring | the app graph in C# (AppHost) — type-safe, refactorable |
| hardcoded URLs / connection strings | **service discovery** + injected connection strings ([04-ServiceDiscovery.md](04-ServiceDiscovery.md)) |
| per-service, inconsistent telemetry | **ServiceDefaults** — OpenTelemetry/health/resilience by convention |
| "works on my machine" env drift | one command brings up the identical graph for everyone |
| no local view of distributed behavior | the **dashboard** — logs, traces, metrics live |
| dev setup ≠ prod setup | the same app model → deployment manifest ([08](08-Deployment.md)) |

---

## Integrations vs orchestration (two halves)

Aspire has two complementary parts, easy to confuse:

- **Hosting integrations (orchestration)** — used in the **AppHost** to *declare and run* a resource: `builder.AddRedis("cache")` starts a Redis container and exposes it in the app graph.
- **Client integrations (consumption)** — used in a **service** to *connect to* that resource: `builder.AddRedisClient("cache")` registers an instrumented `IConnectionMultiplexer` in that service's DI.

The AppHost side *provisions/orchestrates*; the service side *consumes*. The name (`"cache"`) ties them together via service discovery ([04-ServiceDiscovery.md](04-ServiceDiscovery.md)). [05-Integrations.md](05-Integrations.md) covers both halves.

---

## When to adopt Aspire

- **Good fit**: multi-service apps (API + worker + DB + cache + broker), microservices, anything where you currently juggle docker-compose and manual wiring, teams wanting consistent observability/resilience for free, and apps targeting Azure (best-supported deployment).
- **Less necessary**: a single self-contained service with no backing resources (Aspire adds value but less); throwaway scripts; or teams with a mature, satisfactory existing orchestration they don't want to change.

You can also **adopt incrementally** — add an AppHost and ServiceDefaults to an existing solution without rewriting services. The cost is learning the model and a small amount of project structure; the payoff grows with the number of services/resources.

---

## Common gotchas

### Thinking Aspire is a runtime or replaces Kubernetes

Aspire **composes and observes** your apps for development and **generates** deployment artifacts — it doesn't run your code or orchestrate production. Your services run on the normal runtime and deploy to real platforms.

### Confusing hosting vs client integrations

`AddRedis` (AppHost — orchestrates the container) is different from `AddRedisClient` (service — registers the client). The AppHost provisions; the service consumes. Mixing them up means the wiring doesn't connect.

### Expecting it to replace your stack

You still write ASP.NET Core, EF Core, etc. Aspire is a convention/composition layer over them — not a new programming model.

### Assuming it's Azure-only

Aspire is cloud-agnostic at the model level (and runs locally with containers); Azure has the most polished deployment path (`azd`), but you can deploy to Kubernetes or other targets ([08-Deployment.md](08-Deployment.md)).

---

## Summary

- **.NET Aspire** is the **cloud-native composition layer** for .NET: describe your multi-service app's graph (services + backing resources) **in C#** in an **AppHost**, and get **service discovery, telemetry, health checks, resilience, and a live dashboard by convention** — run the whole system locally with one command and deploy the same model to the cloud (GA since .NET 9, matured in .NET 10).
- It **is** an app model/orchestrator *for development*, a set of conventions (**ServiceDefaults**), **integrations** (one-call client wiring), a **dashboard**, and a **deployment model**.
- It is **not** a runtime, **not** Kubernetes/a production orchestrator, and **not** a replacement for your stack — it's the **glue and control panel**, not the engine; your services and resources are real.
- Two halves: **hosting integrations** (AppHost — orchestrate/provision a resource) vs **client integrations** (service — consume it), tied by name via service discovery.
- Adopt it for **multi-service apps** (best on Azure, but cloud-agnostic); it can be added **incrementally** to existing solutions.

→ Next: [02-AppHost.md](02-AppHost.md)
