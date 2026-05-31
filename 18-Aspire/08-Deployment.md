# Deployment

## From local app model to real infrastructure

The AppHost ([02-AppHost.md](02-AppHost.md)) runs your app locally, but the *same app model* also drives **deployment**: Aspire can emit a **manifest** describing every resource (services, databases, caches, connection strings, parameters) and their relationships, which deployment tools turn into real cloud infrastructure. The headline path is **Azure** via the **Azure Developer CLI (`azd`)**, but the model also targets **Kubernetes** and other platforms. Crucially, Aspire **generates deployment artifacts** — it does *not* run your production cluster ([01-WhatIsAspire.md](01-WhatIsAspire.md)); your services run on a real platform, with Aspire bridging the app model to it.

---

## The deployment manifest

Aspire produces a **manifest** (a JSON description) of the app model: each resource, its type (project/container/database/parameter), its endpoints, its connection strings/bindings, and its references. This manifest is the **interchange format** between the app model and deployment tooling — tools read it to know what to provision and how to wire it:

```bash
dotnet run --project AppHost -- --publisher manifest --output-path manifest.json
```

The manifest captures *intent* ("an API that references a Postgres database and a Redis cache, with these endpoints and parameters"). Different **publishers** consume it to target different platforms — so the same described app can deploy to Azure, Kubernetes, etc., without rewriting the AppHost.

---

## Azure with `azd` (the primary path)

The Azure Developer CLI understands Aspire natively. From the AppHost, `azd` provisions Azure resources and deploys your services in one workflow:

```bash
azd init      # detects the Aspire AppHost
azd up        # provisions Azure infra + builds/containerizes services + deploys
```

`azd up` typically:
1. Reads the Aspire app model/manifest.
2. **Provisions** the backing resources as **managed Azure services** — your `AddPostgres` becomes Azure Database for PostgreSQL, `AddRedis` becomes Azure Cache for Redis, `AddAzureServiceBus` becomes a real Service Bus, etc. ([05-Integrations.md](05-Integrations.md)).
3. **Builds and containerizes** your project resources and deploys them to **Azure Container Apps** (the default target).
4. Wires **connection strings, service discovery, and secrets** (mapping secret parameters to Container Apps secrets / Key Vault — [07-ConfigAndSecrets.md](07-ConfigAndSecrets.md)).

The big idea: the local container that backed `AddRedis` in development is replaced by a **real managed Azure Cache** in production — the *same app model* expresses both, and `azd` handles the translation. This is the smoothest Aspire deployment experience.

---

## Kubernetes and other targets

Aspire isn't Azure-only. The manifest can be consumed by tools that target **Kubernetes** — e.g., **Aspir8 (`aspirate`)**, a community tool that generates Kubernetes manifests (Deployments, Services, etc.) from the Aspire app model — and the publisher model is extensible for other platforms. So you can:

- Deploy to **any Kubernetes cluster** via generated manifests/Helm.
- Target other clouds/platforms as publishers mature.

The app model stays the same; only the publisher/target changes. This keeps you from being locked to one platform at the *description* level, even though Azure has the most polished tooling today.

---

## What maps to what

| App model (AppHost) | Local (dev) | Production (e.g., Azure via `azd`) |
|---|---|---|
| `AddProject<T>` | launched .NET process | container in Azure Container Apps |
| `AddPostgres` / `AddRedis` | Docker container | managed Azure DB / Azure Cache |
| `AddAzureServiceBus` | emulator/container | real Azure Service Bus |
| connection strings | injected from local containers | injected from managed-resource endpoints |
| `AddParameter(secret)` | user secrets | Container Apps secret / Key Vault |
| service discovery | local ports | platform addressing / env injection |

The columns differ in *backing*, not in *code* — your services are written once against logical names and injected config ([04-ServiceDiscovery.md](04-ServiceDiscovery.md)), and the deployment supplies the production bindings.

---

## Relationship to general .NET deployment

Aspire deployment composes the per-service deployment concerns from [Ch19 Deployment](../19-Deployment/README.md): each project is still **containerized** (often via the .NET SDK's built-in container publish), still benefits from **trimming/R2R/AOT** considerations, and still needs health probes (which ServiceDefaults' `/health` and `/alive` provide — [03-ServiceDefaults.md](03-ServiceDefaults.md), consumed by Container Apps/Kubernetes). Aspire adds the **composition and wiring** on top — provisioning the backing resources and connecting everything — but the underlying service packaging is standard .NET deployment.

---

## Common gotchas

### Thinking Aspire runs production

Aspire **generates** deployment artifacts and (via `azd`) drives provisioning/deploy, but your services run on **real platforms** (Container Apps, Kubernetes). It's not the production runtime/orchestrator.

### Assuming Azure-only

Azure (`azd` → Container Apps) is the most polished path, but the **manifest** is platform-neutral; Kubernetes (e.g., Aspir8) and other targets are supported. The app model isn't Azure-locked.

### Local container specifics leaking into prod

In production, `AddPostgres` becomes a *managed* database, not your local container — don't depend on local-container behavior/config. Use injected connection strings and the integration's abstraction.

### Secrets not marked as secrets

Secret parameters must be `secret: true` so deployment maps them to a **secret store** (Key Vault / Container Apps secrets) rather than emitting them as plain config ([07-ConfigAndSecrets.md](07-ConfigAndSecrets.md)).

### Skipping health probes

Container Apps/Kubernetes use `/health` and `/alive` to manage routing/restarts. ServiceDefaults provides them via `MapDefaultEndpoints()` — ensure services map them so the platform can manage them correctly ([Ch04 §16](../04-AspNetCore/16-HealthChecks.md)).

---

## Summary

- The **same app model** that runs locally drives **deployment**: Aspire emits a platform-neutral **manifest** (resources, endpoints, connection strings, parameters) that deployment tools consume — Aspire **generates artifacts**, it doesn't run production.
- The primary path is **Azure via `azd`** (`azd up`): it **provisions** managed Azure services (your `AddPostgres`/`AddRedis` → Azure DB/Cache), **containerizes and deploys** projects to **Azure Container Apps**, and wires connection strings, discovery, and **secrets** (→ Container Apps secrets / Key Vault).
- It's **not Azure-only** — the manifest targets **Kubernetes** (e.g., Aspir8) and other platforms via publishers; the app model is unchanged, only the target differs.
- Production differs from local in **backing, not code**: local containers become managed resources, logical names resolve to platform addresses, and secret parameters map to a secret store — composing standard per-service .NET deployment ([Ch19](../19-Deployment/README.md)) with health probes from ServiceDefaults.

→ Next: [09-TestingAspire.md](09-TestingAspire.md)
