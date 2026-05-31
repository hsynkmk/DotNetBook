# Chapter 18 — .NET Aspire

> The cloud-native stack for .NET: an opinionated orchestration model for building, running, and observing multi-service applications. The AppHost wires your services and their backing resources (databases, caches, queues) into one runnable, observable system — locally and in the cloud.

**Prerequisites**: Chapter 03 (Hosting & DI), Chapter 09 (Networking & HTTP), Chapter 12 (Observability). Familiarity with containers helps.

**Time to read**: ~6-8 hours.

---

## Why this chapter

Modern .NET apps are rarely a single process — they're an API, a worker, a database, a Redis cache, maybe a message broker, all needing to find and talk to each other, with configuration, telemetry, and health checks wired consistently. Historically you stitched this together with docker-compose, manual connection strings, and bespoke setup. **.NET Aspire** (GA since .NET 9, matured in .NET 10) makes the composition first-class: you describe your app graph in C#, get service discovery, telemetry, and a live dashboard for free, and deploy the same model to the cloud. It's the flagship cloud-native developer experience for .NET and increasingly the default way to build distributed .NET apps.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-WhatIsAspire.md](01-WhatIsAspire.md) | What Aspire is (and isn't — not a runtime, not Kubernetes), the problems it solves, components vs orchestration, when to adopt it. |
| [02-AppHost.md](02-AppHost.md) | The AppHost project and the app model: `DistributedApplication.CreateBuilder`, `AddProject`, `AddContainer`, references, the dependency graph. |
| [03-ServiceDefaults.md](03-ServiceDefaults.md) | The `ServiceDefaults` shared project: OpenTelemetry, health checks, resilience, and service discovery wired into every service by convention. |
| [04-ServiceDiscovery.md](04-ServiceDiscovery.md) | How services find each other by name, `WithReference`, connection-string injection, endpoint resolution — no hardcoded URLs. |
| [05-Integrations.md](05-Integrations.md) | Aspire integrations (formerly "components") for Postgres, Redis, RabbitMQ, Azure Service Bus, Cosmos, etc. — client wiring + health + telemetry in one call. |
| [06-Dashboard.md](06-Dashboard.md) | The developer dashboard: live logs, distributed traces, metrics, resource state, and environment variables — observability out of the box. |
| [07-ConfigAndSecrets.md](07-ConfigAndSecrets.md) | Configuration flow from AppHost to services, parameters, connection strings, secrets, and per-environment values. |
| [08-Deployment.md](08-Deployment.md) | From local run to production: the deployment manifest, `azd` (Azure Developer CLI), Aspire-to-Kubernetes, and how the app model maps to real infrastructure. |
| [09-TestingAspire.md](09-TestingAspire.md) | `Aspire.Hosting.Testing` — spinning up the whole app graph in integration tests; relationship to `WebApplicationFactory` (Chapter 17). |
| [Questions.md](Questions.md) | ~25 drilling questions. |
| [Coding.md](Coding.md) | Build an AppHost wiring an API + worker + Postgres + Redis; add an integration; read traces in the dashboard; write an Aspire integration test. |

---

## Learning objectives

After this chapter you should be able to:
- Explain what Aspire orchestrates and what it deliberately leaves to the runtime/platform.
- Model a multi-service app in an AppHost and run it with one command.
- Get service discovery, telemetry, health checks, and resilience by convention via ServiceDefaults.
- Add backing resources (databases, caches, brokers) with integrations and consume them with injected, instrumented clients.
- Use the dashboard to debug distributed behavior locally.
- Take the same app model to production via `azd` or Kubernetes.

---

## How this relates to the rest of the book

Aspire is the **composition layer** over things covered elsewhere: it wires up Hosting & DI ([Ch03](../03-HostingAndDI/README.md)), Observability ([Ch12](../12-Observability/README.md)), Resilience ([Ch11](../11-Resilience/README.md)), Configuration ([Ch13](../13-Configuration/README.md)), and Data/Caching ([Ch06](../06-DataAndCaching/README.md)) — and feeds Deployment ([Ch19](../19-Deployment/README.md)). Read those first for depth; this chapter shows how Aspire makes them work together with minimal ceremony.

→ Begin: [01-WhatIsAspire.md](01-WhatIsAspire.md)
