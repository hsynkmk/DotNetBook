# 20 — Azure Integration

> Breadth file. Depth here is worth it only if the role is Azure-facing — but "how do you handle
> secrets and identity in the cloud?" gets asked almost everywhere.

## ⚡ 30-second answer

The modern **`Azure.*` SDKs** share one pattern: a **client** built from an **endpoint URI + a
credential**, registered in DI as a **singleton**. The credential is
**`DefaultAzureCredential`** — a chained credential that resolves to a **managed identity** in
Azure and to your developer login locally, so **the same code works in both**. That plus **RBAC**
is the keyless gold standard: **no secret is stored anywhere**, so there's nothing to leak or
rotate. Everything else follows from picking the right service: **App Service** for a single web
app, **Functions** for event-driven serverless, **Container Apps** for microservices (and Aspire's
default target), **AKS** for full orchestration. **Key Vault** holds secrets and plugs in as a
configuration provider; **App Configuration** holds non-secret settings and feature flags with
dynamic refresh; **Application Insights** is the APM, and the modern path into it is
**OpenTelemetry**.

---

## Core mechanics

### Identity — the part that generalizes

```csharp
builder.Services.AddAzureClients(b =>
{
    b.AddBlobServiceClient(new Uri("https://acct.blob.core.windows.net"));
    b.AddServiceBusClientWithNamespace("ns.servicebus.windows.net");
    b.UseCredential(new DefaultAzureCredential());     // one credential, all clients
});
```

`DefaultAzureCredential` tries a chain: environment variables → workload identity → **managed
identity** → Azure CLI / Visual Studio / Azure Developer CLI. So in Azure it's the app's managed
identity, and on your laptop it's you — **write once, works in both**.

**Managed identity + RBAC** removes the bootstrapping problem entirely: there's no secret to store
in order to fetch your secrets. Access is governed by **role assignments** (least privilege), and
it's audited and instantly revocable.

Clients are **thread-safe and hold connections — register them as singletons.**

### Compute — choosing the shape

| Service | For | Watch out for |
|---|---|---|
| **App Service** | a single web app or API, managed PaaS | slots for zero-downtime; keep it stateless |
| **Functions** | event-driven, serverless | **cold starts** on Consumption |
| **Container Apps** | microservices, Aspire's default target | scale-to-zero also means cold start |
| **AKS** | full orchestration, you want the control | you now own Kubernetes |

**Azure Functions** run on **triggers** (HTTP, queue/Service Bus, blob, timer, Event Grid, Cosmos
change feed). The modern model is the **isolated worker** — your function runs in its own process
with full DI and middleware, decoupled from the host. **Bindings** declaratively wire inputs and
outputs. Plans: Consumption (scale-to-zero, cheap, **cold starts**), Premium/Flex (pre-warmed,
VNet), Dedicated. Mitigate cold starts with Premium/Flex or **Native AOT**
([18–19](18-19-Aspire-Deployment.md)).

### The three messaging services — don't mix them up

| | Semantics | Analogy |
|---|---|---|
| **Service Bus** | reliable enterprise **messaging** — commands, work, ordering (sessions), dead-lettering | a queue |
| **Event Grid** | lightweight push **notifications** with filtered subscriptions | a doorbell |
| **Event Hubs** | high-volume **streaming** log, partitioned, replayable, consumer groups | Kafka |

Service Bus is at-least-once with **peek-lock settlement** — Complete, Abandon, or Dead-letter,
with automatic DLQ after `MaxDeliveryCount`. Watch **lock expiry** on long processing. Full
patterns (idempotency, outbox, ordering) are in [07](07-Messaging.md).

### Storage and data

**Blob Storage** — account → container → blob. **Stream** uploads and downloads; never buffer a
large blob in memory. For client access, use a **user-delegation SAS** (keyless, time-limited,
scoped) so bytes bypass your app entirely. **Access tiers** (Hot/Cool/Cold/Archive) trade storage
cost against access cost; lifecycle policies auto-tier aging data, and Archive needs
**rehydration** before you can read it.

**Cosmos DB** — the critical design choice is the **partition key**: high-cardinality, evenly
distributed, and **query-aligned** so your common queries hit a single partition. A poor choice
gives you hot partitions and expensive cross-partition fan-out, and **you cannot change it without
migrating**. Prefer **point reads** (id + partition key — cheapest in RUs). Five consistency
levels from Strong to Eventual, with **Session** the default sweet spot. The **change feed** is an
ordered log of changes powering Functions triggers and materialized views.

### Configuration and secrets

**Key Vault** holds secrets, keys and certificates. It plugs in as a **configuration provider**, so
vault secrets become ordinary `IConfiguration` keys (`A--B` → `A:B`) and consuming code doesn't
change. Access via managed identity. Handle **rotation** with provider reload plus
**`IOptionsMonitor<T>`** — not `IOptions<T>`, which never reloads
([03](03-Hosting-DI-Config.md)). For keys, do the cryptography **in the vault**
(`CryptographyClient`) rather than extracting the key.

**App Configuration** centralizes **non-secret** settings and **feature flags** across services,
with **labels** for per-environment separation and **dynamic refresh** (sentinel key + interval +
middleware). Feature Management gives declarative flags with percentage rollout and targeting —
which is what makes flags actually decouple deployment from release
([22](22-BestPractices-Architecture.md)).

**Use both**: App Configuration for settings and flags, **Key Vault for secrets** via references.

### Application Insights

Azure's APM — logs, metrics, distributed traces, an application map, dashboards and alerting. The
**modern path is OpenTelemetry** instrumentation ([12](12-Observability.md)). Correlation is via
W3C trace context (`operation_Id`), and **sampling** controls volume and cost at scale. It's the
production counterpart to the Aspire dev dashboard.

---

## 🪤 Traps & gotchas

- **Connection strings and account keys in configuration** — the whole point of managed identity is
  that there's nothing to store, leak or rotate. If you're storing a key, ask why.
- **Creating an Azure SDK client per request** — they're thread-safe and hold connections.
  Singleton, exactly like `HttpClient`'s handler ([09](09-Http-gRPC-SignalR.md)).
- **Forgetting the RBAC role assignment** — `DefaultAzureCredential` authenticates fine and every
  call returns **403**. The confusing part is that authentication succeeded; it's authorization
  that failed.
- **`DefaultAzureCredential` picking the wrong identity** — the chain finds a stale Azure CLI login
  or the wrong managed identity on a resource with several. Be explicit in production
  (`ManagedIdentityCredential` with the client id).
- **A bad Cosmos partition key** — hot partitions, expensive cross-partition queries, and a
  migration to fix it. This is the one Cosmos decision that's genuinely hard to reverse.
- **Cross-partition queries by default** — always include the partition key where you can; a fan-out
  costs RUs proportional to the number of partitions.
- **Assuming Cosmos is strongly consistent** — the default is **Session**. Read-your-writes holds
  within a session, not globally.
- **Buffering a large blob into memory** — stream it. The SDK does parallel block transfer for you.
- **Uploading through your API when a SAS would do** — you're paying compute and bandwidth to be a
  proxy. A user-delegation SAS lets the client talk to storage directly.
- **Reading an Archive-tier blob** — it isn't available until rehydrated, which takes hours.
- **Service Bus lock expiry during long processing** — the lock times out, the message is
  redelivered, and you process it twice. Renew the lock or shorten the work.
- **Not monitoring the dead-letter queue** — a DLQ nobody watches is a slow way to lose messages.
- **Confusing Event Grid with Event Hubs** — notifications versus a streaming log. Using Event Hubs
  for a doorbell, or Event Grid for high-volume telemetry, goes badly in different directions.
- **Cold starts on the Consumption plan** — a first request after idle can take seconds. Premium,
  Flex, or AOT.
- **Using `IOptions<T>` for a rotated secret** — it's read once at startup, so rotation never
  reaches your app. `IOptionsMonitor<T>` plus provider reload.
- **Secrets in App Configuration** — it's for non-secret settings and flags. Secrets go in Key
  Vault, referenced from App Configuration.
- **Stateful app on App Service with autoscale** — in-memory session, in-memory cache, and the Data
  Protection key ring all break as soon as there's a second instance
  ([10](10-Identity-Security.md)).
- **100% Application Insights sampling in production** — cost nobody budgeted. Sample, keeping
  errors and slow requests.

---

## ❓ Likely questions

**Q: How do you authenticate to Azure services from a .NET app?**
A: `DefaultAzureCredential` with **managed identity**. It's a chained credential — in Azure it
resolves to the app's managed identity, locally to your Azure CLI or Visual Studio login — so the
same code works in both without any conditional logic, and no secret is stored anywhere.

**Q: Why is managed identity better than a connection string?**
A: There's no secret to store, leak, rotate or accidentally commit — which also solves the
bootstrapping problem of "what secret do I need to fetch my secrets?" Access is governed by RBAC
role assignments, so it's least-privilege, audited and instantly revocable.

**Q: How should Azure SDK clients be registered in DI?**
A: As **singletons** — they're thread-safe and hold connections, so creating one per request
wastes connections and defeats pooling. `AddAzureClients` registers them consistently with one
shared credential.

**Q: How do secrets get into configuration?**
A: Key Vault registers as a **configuration provider**, so vault secrets appear as ordinary
`IConfiguration` keys and consuming code is unchanged (`A--B` in the vault becomes `A:B`). The app
authenticates to the vault with its managed identity.

**Q: How do you handle secret rotation?**
A: The configuration provider reloads, and consumers use **`IOptionsMonitor<T>`** so they see the
new value. `IOptions<T>` is read once at startup and never updates, which is the usual reason
rotation appears not to work.

**Q: Key Vault or App Configuration?**
A: Both, for different things. App Configuration holds non-secret settings and feature flags,
with labels per environment and dynamic refresh. Key Vault holds secrets, keys and certificates.
App Configuration can hold **references** to Key Vault secrets, so there's one place to look and
the secrets still live in the vault.

**Q: What's the most important Cosmos DB design decision?**
A: The **partition key**. It needs high cardinality, even distribution, and alignment with your
most common queries so they're single-partition. A poor choice produces hot partitions and
expensive fan-out queries, and you can't change it without migrating the data.

**Q: What are Cosmos DB's consistency levels?**
A: Five, from Strong through Bounded Staleness, Session, Consistent Prefix, to Eventual. **Session
is the default** and the usual sweet spot — read-your-writes within a session, without the latency
and cost of global strong consistency.

**Q: Service Bus, Event Grid, or Event Hubs?**
A: Service Bus for reliable messaging — commands and work items needing ordering, sessions and
dead-lettering. Event Grid for lightweight push notifications with filtered subscriptions, like
"blob uploaded → run a Function". Event Hubs for high-volume streaming ingestion with replay and
consumer groups, which is the Kafka-shaped workload.

**Q: How do Service Bus delivery semantics work?**
A: Peek-lock: the consumer locks the message, processes it, then **settles** — Complete on
success, Abandon to retry immediately, or Dead-letter for a poison message. After
`MaxDeliveryCount` it's dead-lettered automatically. It's at-least-once, so consumers must be
idempotent.

**Q: How do you let a client upload a large file?**
A: A **user-delegation SAS** — a time-limited, scope-limited, keyless token signed via your managed
identity — so the client uploads straight to Blob Storage and your API never handles the bytes.

**Q: App Service, Container Apps, Functions or AKS?**
A: App Service for a single web app where you want PaaS to handle the OS, TLS and scaling.
Functions for event-driven work that should scale to zero. Container Apps for microservices
without owning Kubernetes — it's also Aspire's default deployment target. AKS when you genuinely
need Kubernetes' control and are prepared to operate it.

**Q: How do you get zero-downtime deploys on App Service?**
A: Deployment slots. Deploy to staging, let it warm up, then **swap** — which is near-instant
because the instance is already warm — and swap back to roll back.

**Q: How do you instrument for Application Insights today?**
A: With **OpenTelemetry** — `ILogger`, `Meter` and `ActivitySource` exported to Application
Insights. That keeps your instrumentation vendor-neutral while still getting the application map,
end-to-end transaction correlation and alerting.

---

## 🎓 Senior Extra

- **`DefaultAzureCredential` is convenient and slightly dangerous.** The chain makes local
  development seamless, but in production you want to know *exactly* which identity you're using —
  pin it explicitly so a misconfiguration fails loudly instead of silently falling back.
- **RBAC propagation is eventually consistent.** A fresh role assignment can take minutes to take
  effect, which produces a "it works now, it didn't five minutes ago" that has confused many
  deployments.
- **Cosmos RU budgeting is the real cost model.** A point read is around 1 RU, a cross-partition
  query can be hundreds. Designing for point reads and single-partition queries is a cost decision
  as much as a latency one.
- **The Cosmos change feed is an event source you already own** — materialized views, cache
  invalidation, and search index updates all fall out of it without adding a broker
  ([07](07-Messaging.md)).
- **Prefer the SAS to the proxy.** Any time your service is streaming bytes it didn't transform,
  there's usually a SAS-shaped answer that removes the bandwidth and the timeout risk.
- **Feature flags only pay off with a runtime-changeable source.** App Configuration's dynamic
  refresh is what turns a flag from a deploy-time constant into an actual release control
  ([22](22-BestPractices-Architecture.md)).
- **Application Insights sampling preserves accuracy through adaptive sampling** — it records the
  sampling ratio so counts stay correct — but it still means an individual trace may be missing.
  Know that before you promise someone you can find any specific request.
- **Aspire's Azure story is `azd up`**: your `AddPostgres` and `AddRedis` become managed Azure
  resources and your projects deploy to Container Apps, wired with connection strings and managed
  identity. That's the clean answer to "how does Aspire get to production?"
  ([18–19](18-19-Aspire-Deployment.md)).
- **Private endpoints and VNet integration** are where the Consumption plan runs out — needing
  network isolation is the usual reason a team moves from Consumption to Premium/Flex or Container
  Apps, more often than cold starts.

→ Deeper: [`../20-AzureIntegration/`](../20-AzureIntegration/README.md)
