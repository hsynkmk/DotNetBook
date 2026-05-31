# Azure App Service

## Managed hosting for web apps

**Azure App Service** is a fully-managed Platform-as-a-Service (PaaS) for hosting web apps and APIs — you deploy your ASP.NET Core app and Azure handles the OS, runtime patching, load balancing, scaling, and TLS, with no servers or containers to manage. It's the **lowest-friction** way to run a .NET web app in Azure: no Dockerfile required (though containers are supported), built-in CI/CD, deployment slots for zero-downtime releases, and tight integration with Azure config/secrets/monitoring. For a single web app (vs a multi-service distributed system that fits Aspire/Kubernetes — [Ch18](../18-Aspire/README.md), [04-Kubernetes.md](04-Kubernetes.md)), App Service is often the simplest, most productive choice.

---

## Deploying to App Service

Several deployment paths, from simplest to most controlled:

```bash
# Zip deploy from a published folder:
dotnet publish -c Release -o ./publish
az webapp deploy --resource-group rg --name myapp --src-path ./publish --type zip

# Or push a container image:
az webapp create --resource-group rg --plan myplan --name myapp \
    --deployment-container-image-name myregistry.azurecr.io/myapp:1.2.0
```

- **Zip/folder deploy** — publish the app and upload it; App Service runs it on its managed runtime (framework-dependent — Azure provides the .NET runtime).
- **Container deploy** — run your own image ([02-Docker.md](02-Docker.md)) on App Service (for custom dependencies/control).
- **CI/CD integration** — GitHub Actions / Azure Pipelines deploy automatically on push ([09-CICD.md](09-CICD.md)); App Service has built-in deployment center wiring.

App Service supports .NET as a first-class stack — you typically don't need a container at all for a standard app.

---

## Configuration — App Settings

App Service injects **Application Settings** as **environment variables**, which .NET reads via the configuration providers ([Ch13 §02](../13-Configuration/02-Providers.md)) — so your app's config "just works" without App Service-specific code:

```
# App Setting "ConnectionStrings__Db"  → config key "ConnectionStrings:Db"
# Use __ (double underscore) for nested keys, same as containers ([04-Kubernetes.md])
```

- **App Settings** override `appsettings.json` (they're environment variables, layered last — [Ch13 §03](../13-Configuration/03-Layering.md)).
- **Connection Strings** have a dedicated section in the portal.
- **Key Vault references** let an App Setting's value come from Azure Key Vault ([Ch13 §07](../13-Configuration/07-Secrets.md)) — `@Microsoft.KeyVault(...)` — so secrets aren't stored in App Service config directly; combine with a **managed identity** to access the vault without credentials.

---

## Deployment slots — zero-downtime releases

A standout App Service feature is **deployment slots**: separate, live instances of your app (e.g., `staging` alongside `production`) that you deploy to and then **swap**:

```bash
az webapp deployment slot create --name myapp --resource-group rg --slot staging
# deploy the new version to 'staging', warm it up, verify, then:
az webapp deployment slot swap --name myapp --resource-group rg --slot staging --target-slot production
```

The **swap** is near-instant and **warms up** the staging instance before routing production traffic to it (so users never hit a cold start), and you can **swap back** to roll back immediately. Slots give you blue/green-style zero-downtime deploys and instant rollback **without** Kubernetes — a major reason teams pick App Service. (Slot-specific settings can stay with the slot or swap with it, configurable per setting.)

---

## Scaling

- **Scale up** (vertical) — a bigger App Service Plan (more CPU/memory) for a single instance.
- **Scale out** (horizontal) — more instances, with **autoscale** rules by CPU/memory/schedule/metrics. App Service load-balances across instances.
- **Stateless requirement**: like any scaled-out platform, instances are interchangeable — externalize state (distributed cache/DB — [Ch06](../06-DataAndCaching/README.md)); don't rely on in-memory session affinity (App Service offers ARR affinity/sticky sessions, but stateless is better for scale).

---

## Monitoring and health

- **Application Insights** integrates for telemetry — your OpenTelemetry/`ILogger` data ([Ch12](../12-Observability/README.md)) flows to a managed APM with traces, metrics, and live metrics.
- **Health check path** — configure App Service with your `/health` endpoint so it removes unhealthy instances from rotation ([Ch04 §16](../04-AspNetCore/16-HealthChecks.md)).
- **Log streaming / diagnostics** — live logs and diagnostic settings without SSHing into a box.

---

## Common gotchas

### Secrets in App Settings plaintext

Putting raw secrets in App Settings is better than in the repo but still exposes them in the portal/config. Use **Key Vault references** + **managed identity** so the actual secret lives in the vault ([Ch13 §07](../13-Configuration/07-Secrets.md)).

### Cold swaps without warmup

Swapping without warming the staging slot (or without proper health checks) can route traffic to a not-yet-ready instance. App Service warms up on swap — ensure your app's startup/health endpoints support it.

### Stateful app + scale-out

In-memory state breaks across multiple instances. Externalize state; don't depend on sticky sessions for correctness ([Ch06](../06-DataAndCaching/README.md)).

### Wrong nested-config separator

App Settings use `__` for nested keys (`ConnectionStrings__Db`), not `:`. Using the wrong separator means the config key isn't bound ([Ch13 §03](../13-Configuration/03-Layering.md)).

### Choosing App Service for a multi-service distributed app

App Service shines for a web app/API; a multi-service system with its own orchestration fits **Aspire + Container Apps/Kubernetes** better ([Ch18 §08](../18-Aspire/08-Deployment.md), [04-Kubernetes.md](04-Kubernetes.md)). Match the platform to the app's shape.

---

## Summary

- **Azure App Service** is managed PaaS for web apps/APIs — Azure handles OS, runtime patching, load balancing, scaling, and TLS; deploy via **zip/folder**, **container**, or **CI/CD** with **no Dockerfile required** for standard apps — the lowest-friction Azure hosting for a single web app.
- **Configuration** comes from **App Settings** injected as env vars (use `__` for nested keys, they override `appsettings.json`); secrets should use **Key Vault references + managed identity** rather than plaintext settings ([Ch13](../13-Configuration/README.md)).
- **Deployment slots** give **zero-downtime** blue/green deploys: deploy to `staging`, warm up, **swap** into production (near-instant, warmed), and **swap back** to roll back — without Kubernetes.
- **Scale** up (bigger plan) or out (autoscale instances — needs **stateless** apps); integrate **Application Insights** ([Ch12](../12-Observability/README.md)) and a **health-check path** ([Ch04 §16](../04-AspNetCore/16-HealthChecks.md)). Pick App Service for single web apps; use **Aspire + Container Apps/Kubernetes** for multi-service systems.

→ Next: [Questions.md](Questions.md)
