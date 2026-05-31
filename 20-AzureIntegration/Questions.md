# Chapter 20 — Azure Integration — Q & A

---

### Q1. What's the common pattern across the modern Azure SDKs?

Each service has a strongly-typed **client** built from an **endpoint URI + credential**, registered in DI (as a **singleton** — thread-safe, connection-pooling), called via async `Response<T>` methods with built-in retries and OpenTelemetry. Learn one and you know them all.

---

### Q2. What is `DefaultAzureCredential`?

A **chained** credential that tries multiple auth mechanisms in order (env vars → managed identity → Visual Studio/CLI login) and uses the first that works — so the **same code authenticates** in production (managed identity) and locally (your dev login), with no code changes.

---

### Q3. What is managed identity and why is it the gold standard?

An identity managed by Azure AD (Entra ID) assigned to your Azure-hosted app, letting it authenticate to other services **with no credential stored in the app**. Access is governed by **RBAC roles** (least-privilege, auditable, revocable) — nothing to leak or rotate. It solves the bootstrapping problem (no secret needed to get secrets).

---

### Q4. System-assigned vs user-assigned managed identity?

**System-assigned** is tied to one resource's lifecycle (created/deleted with it). **User-assigned** is a standalone identity that can be shared across multiple resources. Use user-assigned when several resources should share one identity/role set.

---

### Q5. How do you register Azure clients in DI cleanly?

`Microsoft.Extensions.Azure`'s `AddAzureClients` — register all clients in one place, share a single credential (`UseCredential`), get singleton lifetimes and config integration. Inject the clients (`BlobServiceClient`, `ServiceBusClient`, etc.) anywhere.

---

### Q6. What's the modern Azure Functions model?

The **isolated worker** model (.NET 8+): your function runs in **its own process** (decoupled from the Functions host), with **full DI/middleware** and any .NET version. The legacy in-process model is being retired.

---

### Q7. Triggers vs bindings in Functions?

**Triggers** define what *starts* a function (HTTP, queue/Service Bus, blob, timer, Event Grid/Hub, Cosmos change feed). **Bindings** declaratively connect inputs/outputs (read a blob, write a queue, return HTTP) without manually constructing clients.

---

### Q8. What are the Functions hosting plans and the cold-start concern?

**Consumption** (scale-to-zero, pay-per-execution, but **cold starts**), **Premium/Flex** (pre-warmed, VNet, low cold start), **Dedicated** (App Service plan, predictable, no scale-to-zero). Mitigate cold starts with Premium/Flex or **Native AOT**.

---

### Q9. Service Bus: queues vs topics?

**Queue** = point-to-point (one queue, one consumer per message; competing consumers for scale). **Topic + subscriptions** = pub-sub (each subscription gets its own filtered copy). Queues for work/commands; topics for broadcasting events to multiple consumers.

---

### Q10. What is the dead-letter queue and when do messages go there?

A separate queue holding **poison messages** — those that repeatedly fail (exceed `MaxDeliveryCount`) or are explicitly dead-lettered (invalid). It isolates bad messages so they don't block the queue; monitor it and have a replay/inspection process.

---

### Q11. Why do Service Bus (and most messaging) handlers need to be idempotent?

Delivery is **at-least-once** — a message can be redelivered (failure, lock expiry). So a handler may run more than once per message; it must be **idempotent** (dedupe by id / duplicate detection) so redelivery doesn't double-process.

---

### Q12. What are Service Bus sessions for?

**FIFO ordering within a session**: all messages with the same `SessionId` are processed in order by a single consumer at a time. Use when order matters for a logical entity (all events for one order/customer must be sequential).

---

### Q13. How should you upload/download large blobs?

**Stream** them (the SDK does parallel block transfer) — never buffer a multi-GB blob fully in memory (`byte[]`/`BinaryData`), which spikes memory/OOM. Use `UploadAsync(stream)`/`DownloadToAsync`/`OpenReadAsync`.

---

### Q14. When use a SAS token vs managed identity for blob access?

**Managed identity + RBAC** for your *app* to access storage (keyless). **SAS** (ideally **user-delegation SAS**, key-less) for time-limited, scoped **direct client** access (browser/mobile uploads/downloads) — offloading bytes from your server.

---

### Q15. What are blob access tiers?

Hot (frequent, higher storage cost/low access cost), Cool (≥30 days), Cold (≥90 days), Archive (lowest storage, **must rehydrate** for hours before reading). Lifecycle policies auto-transition aging data to cheaper tiers.

---

### Q16. What's the most important Cosmos DB design decision?

The **partition key**. It must be **high-cardinality, evenly-distributed, and query-aligned** so common queries are **single-partition** (cheap/fast). A poor key creates a **hot partition** that caps throughput; it's effectively immutable for a container.

---

### Q17. Point read vs query in Cosmos?

A **point read** (id + partition key) is the cheapest/fastest (~1 RU, no query engine) — prefer it when you know both. **Queries** cost more RUs; always include the partition key when possible (cross-partition queries fan out across all partitions — slow, expensive).

---

### Q18. What are Cosmos's consistency levels?

Five tunable levels: **Strong** (latest, highest latency), **Bounded staleness**, **Session** (default — read-your-writes per session, good balance), **Consistent prefix**, **Eventual** (fastest, weakest). Choose Session unless you need stronger; Strong constrains multi-region and adds latency.

---

### Q19. What does Key Vault store, and what's special about keys?

**Secrets** (sensitive strings), **keys** (crypto), and **certificates** (with lifecycle/renewal). For **keys**, the private material **never leaves the vault** — you send data to the vault to encrypt/sign (`CryptographyClient`), so the key is never exposed (optionally HSM-backed).

---

### Q20. How does Key Vault integrate with configuration?

As a **configuration provider** (`AddAzureKeyVault`): vault secrets become ordinary `IConfiguration` keys (secret `A--B` → key `A:B`), read with no Key Vault-specific code. App Service can also use **Key Vault references** in App Settings. Access via managed identity — no secret to access the vault.

---

### Q21. Distinguish Service Bus, Event Grid, and Event Hubs.

**Service Bus** = enterprise **messaging** (discrete reliable commands/work, ordering/sessions/dead-letter). **Event Grid** = **event notification/routing** (react to "X happened", especially Azure resource events). **Event Hubs** = **event streaming** (millions/sec telemetry ingestion, Kafka-like, replayable). Choose by semantics and scale.

---

### Q22. Event Hubs vs Service Bus on replay?

**Event Hubs** retains events in a partitioned log (read by **offset**, replayable, **consumer groups** track position) — like Kafka. **Service Bus** removes messages on completion (queue semantics, no replay). Don't expect stream semantics from a queue or vice versa.

---

### Q23. What does Azure App Configuration provide?

Centralized **non-secret** configuration and **feature flags** across services — solving duplication/drift, enabling change-without-redeploy, per-environment **labels**, and **dynamic refresh** (sentinel key + interval). Use it **with Key Vault** for secrets (App Config holds settings/flags, references Key Vault for secrets).

---

### Q24. What do feature flags enable?

Decoupling **release** from **deployment**: ship code with a feature off, then enable it (on/off, **percentage rollout**, targeting) centrally via `IFeatureManager`/`<feature>` tags without redeploying — and roll back instantly by toggling. Foundation of progressive delivery.

---

### Q25. What's the modern way to use Application Insights?

Instrument with **OpenTelemetry** and export via the **Azure Monitor exporter** (`UseAzureMonitor()`) — **not** the legacy App Insights SDK. This keeps instrumentation **vendor-neutral**; the same telemetry that feeds the Aspire dashboard locally feeds App Insights in production. Query with **KQL**; use **sampling** at scale; scrub PII.
