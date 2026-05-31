# Chapter 20 — Azure Integration

> Building .NET apps that play well with Azure: Functions, App Service, Service Bus, Storage, Cosmos DB, Key Vault, Managed Identity. Patterns that scale.

**Prerequisites**: Chapter 04 (ASP.NET Core), Chapter 03 (Hosting & DI). Familiarity with cloud concepts.

**Time to read**: ~8-10 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-OverviewAndIdentity.md](01-OverviewAndIdentity.md) | Azure SDK pattern; `DefaultAzureCredential`, Managed Identity, key-less authentication. |
| [02-AppService.md](02-AppService.md) | Deploying ASP.NET Core to App Service, slots, scale settings. |
| [03-Functions.md](03-Functions.md) | Azure Functions — isolated worker model (.NET 8+), triggers, bindings. |
| [04-ServiceBus.md](04-ServiceBus.md) | Azure.Messaging.ServiceBus — queues, topics, sessions, dead-lettering. |
| [05-BlobStorage.md](05-BlobStorage.md) | Azure.Storage.Blobs — uploads, downloads, SAS, lifecycle. |
| [06-CosmosDB.md](06-CosmosDB.md) | Azure.Cosmos — SDK basics, partitioning, query, change feed. |
| [07-KeyVault.md](07-KeyVault.md) | Azure.Security.KeyVault.Secrets/Keys/Certificates — configuration provider integration. |
| [08-EventGrid-EventHubs.md](08-EventGrid-EventHubs.md) | Event-driven Azure: when each is right. |
| [09-AppConfig.md](09-AppConfig.md) | Azure App Configuration + Feature Management. |
| [10-AppInsights.md](10-AppInsights.md) | Application Insights via OpenTelemetry — the recommended modern path. |
| [Questions.md](Questions.md) | Drilling. |
| [Coding.md](Coding.md) | Wire a Function to a queue; pull config from Key Vault; emit metrics to App Insights. |

→ Begin: [01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)
