# App Service (Azure Integration View)

## Hosting ASP.NET Core on managed PaaS

**Azure App Service** is the managed platform for hosting ASP.NET Core web apps/APIs ([Ch19 §10](../19-Deployment/10-AppService.md) covers deployment mechanics). This section views it through the **Azure integration** lens: how an App Service-hosted app authenticates to *other* Azure services (managed identity), pulls configuration/secrets, and fits the broader Azure ecosystem. The recurring theme of this chapter — **keyless auth via managed identity** ([01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)) — is what makes an App Service app cleanly consume Service Bus, Storage, Key Vault, Cosmos, and the rest.

> Deployment details (slots, zip/container deploy, scaling) are in [Ch19 §10](../19-Deployment/10-AppService.md). Here we focus on identity, config, and ecosystem integration.

---

## Managed identity on App Service

App Service can be assigned a **managed identity** (system- or user-assigned — [01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)) with one toggle. Once enabled, your app authenticates to other Azure services **with no credentials in config** — `DefaultAzureCredential` picks up the App Service identity automatically:

```csharp
// In an App Service-hosted app — this resolves to the App Service managed identity in production:
builder.Services.AddAzureClients(c => {
    c.AddBlobServiceClient(new Uri("https://acct.blob.core.windows.net"));
    c.AddServiceBusClient("ns.servicebus.windows.net");
    c.UseCredential(new DefaultAzureCredential());
});
```

You grant the App Service identity **RBAC roles** on each target resource (Blob Data Contributor on the storage account, Service Bus Data Sender on the namespace, Key Vault Secrets User on the vault). No connection strings, no keys — the app proves its identity via Azure AD. This is the clean, secret-free way to wire an App Service app to the Azure ecosystem.

---

## Configuration flow

App Service supplies configuration as **environment variables** that .NET reads via the config providers ([Ch13 §02](../13-Configuration/02-Providers.md)) — no App Service-specific code:

- **App Settings** → env vars (use `__` for nested keys: `ConnectionStrings__Db`); they override `appsettings.json` ([Ch13 §03](../13-Configuration/03-Layering.md)).
- **Key Vault references** — an App Setting value of the form `@Microsoft.KeyVault(SecretUri=...)` is resolved by App Service from Key Vault using the app's managed identity ([07-KeyVault.md](07-KeyVault.md)) — so secrets live in the vault, not in App Service config.
- **Azure App Configuration** ([09-AppConfig.md](09-AppConfig.md)) — for centralized config + feature flags across multiple apps.

The combination — App Settings for non-secrets, Key Vault references (via managed identity) for secrets — keeps secrets out of both the repo and the App Service config blade.

---

## Where App Service fits in the Azure picture

App Service is one of several Azure compute options; choose by app shape ([Ch19](../19-Deployment/README.md), [Ch18 §08](../18-Aspire/08-Deployment.md)):

| Option | Best for |
|---|---|
| **App Service** | a single web app/API — simplest managed hosting, slots, autoscale |
| **Azure Functions** | event-driven/serverless, pay-per-execution ([03-Functions.md](03-Functions.md)) |
| **Azure Container Apps** | containerized microservices, the default Aspire target ([Ch18 §08](../18-Aspire/08-Deployment.md)) |
| **AKS** (Kubernetes) | full control / complex orchestration ([Ch19 §04](../19-Deployment/04-Kubernetes.md)) |

App Service is the pragmatic choice for a straightforward web app; multi-service systems lean toward Container Apps/AKS (often via Aspire — [Ch18](../18-Aspire/README.md)).

---

## Observability

App Service integrates with **Application Insights** ([10-AppInsights.md](10-AppInsights.md)) — your OpenTelemetry/`ILogger` telemetry ([Ch12](../12-Observability/README.md)) flows to a managed APM with traces, metrics, and live metrics. Configure the app's `/health` endpoint as the App Service **health check path** ([Ch04 §16](../04-AspNetCore/16-HealthChecks.md)) so unhealthy instances are removed from rotation. Together these give production visibility without managing monitoring infrastructure.

---

## Common gotchas

### Storing secrets in App Settings plaintext

Better than the repo, but still visible in the config blade. Use **Key Vault references + managed identity** so the real secret stays in the vault ([07-KeyVault.md](07-KeyVault.md), [Ch13 §07](../13-Configuration/07-Secrets.md)).

### Managed identity without RBAC roles

Enabling the identity isn't enough — assign the right **RBAC role** on each target resource, or calls get **403** ([01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)).

### Wrong nested-config separator

App Settings use `__` (not `:`) for nested keys. Wrong separator → the key isn't bound ([Ch13 §03](../13-Configuration/03-Layering.md)).

### Choosing App Service for a multi-service system

App Service suits a single web app; a distributed system fits **Container Apps/AKS** (often via Aspire). Match the platform to the app shape ([Ch18 §08](../18-Aspire/08-Deployment.md)).

### Stateful app + scale-out

Scaled-out instances are interchangeable — externalize state (cache/DB), don't rely on in-memory state/sticky sessions ([Ch06](../06-DataAndCaching/README.md), [Ch19 §10](../19-Deployment/10-AppService.md)).

---

## Summary

- **App Service** hosts ASP.NET Core as managed PaaS (deployment mechanics in [Ch19 §10](../19-Deployment/10-AppService.md)); the Azure-integration value is **keyless access to other Azure services** via a **managed identity** (system/user-assigned) that `DefaultAzureCredential` picks up — grant it **RBAC roles** per resource, no secrets in config.
- **Configuration** flows from **App Settings** (env vars, `__` for nesting, override `appsettings.json`), **Key Vault references** (secrets resolved via managed identity — secrets stay in the vault), and **Azure App Configuration** (centralized config/flags — [09-AppConfig.md](09-AppConfig.md)).
- Choose App Service for a **single web app**; **Functions** (serverless), **Container Apps** (microservices, Aspire), or **AKS** (full orchestration) for other shapes.
- Integrate **Application Insights** ([10-AppInsights.md](10-AppInsights.md)) and a **health-check path** for production observability; keep secrets out of plaintext settings and apps **stateless** for scale-out.

→ Next: [03-Functions.md](03-Functions.md)
