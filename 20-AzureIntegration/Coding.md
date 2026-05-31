# Chapter 20 — Azure Integration — Coding Problems

Wire a Function to a queue, pull config from Key Vault, access Blob/Cosmos with managed identity, and emit telemetry to App Insights. Each problem has a hidden solution — attempt it first.

---

### Problem 1 — Register Azure clients with one credential

Register Blob, Service Bus, and Key Vault Secret clients sharing `DefaultAzureCredential`.

<details>
<summary>Solution</summary>

```csharp
builder.Services.AddAzureClients(clients => {
    clients.AddBlobServiceClient(new Uri("https://acct.blob.core.windows.net"));
    clients.AddServiceBusClient("ns.servicebus.windows.net");
    clients.AddSecretClient(new Uri("https://vault.vault.azure.net"));
    clients.UseCredential(new DefaultAzureCredential());     // one credential for all
});
// Inject: public class Svc(BlobServiceClient blobs, ServiceBusClient bus) { }
```

`AddAzureClients` registers thread-safe **singletons** with a shared keyless credential ([01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)). No connection strings/keys.
</details>

---

### Problem 2 — Pull configuration from Key Vault

Make vault secrets available as `IConfiguration` keys.

<details>
<summary>Solution</summary>

```csharp
builder.Configuration.AddAzureKeyVault(
    new Uri("https://myvault.vault.azure.net/"),
    new DefaultAzureCredential());

// Secret "ConnectionStrings--Db" → config key "ConnectionStrings:Db":
var conn = builder.Configuration.GetConnectionString("Db");
```

The provider maps `--` to `:`; consuming code reads config normally — no Key Vault-specific code, no secret to access the vault (managed identity) ([07-KeyVault.md](07-KeyVault.md)).
</details>

---

### Problem 3 — An Azure Function triggered by a queue

Write an isolated-worker Function that processes messages from a Service Bus queue.

<details>
<summary>Solution</summary>

```csharp
public class OrderProcessor(ILogger<OrderProcessor> logger) {
    [Function("ProcessOrder")]
    public async Task Run([ServiceBusTrigger("orders", Connection = "ServiceBus")] Order order) {
        logger.LogInformation("Processing order {Id}", order.Id);
        await HandleAsync(order);   // make idempotent — at-least-once delivery
    }
}
```
```csharp
// Program.cs (isolated worker host):
var host = new HostBuilder().ConfigureFunctionsWorkerDefaults()
    .ConfigureServices(s => s.AddSingleton<IOrderService, OrderService>())
    .Build();
host.Run();
```

Isolated worker model with full DI; the `ServiceBusTrigger` binding delivers messages. Handler must be **idempotent** (redelivery) ([03-Functions.md](03-Functions.md), [04-ServiceBus.md](04-ServiceBus.md)).
</details>

---

### Problem 4 — Send to a Service Bus queue

Publish an order message to the `orders` queue.

<details>
<summary>Solution</summary>

```csharp
public class OrderPublisher(ServiceBusClient client) {
    public async Task PublishAsync(Order order, CancellationToken ct) {
        var sender = client.CreateSender("orders");
        var message = new ServiceBusMessage(BinaryData.FromObjectAsJson(order)) {
            MessageId = order.Id.ToString()      // enables duplicate detection / idempotency
        };
        await sender.SendMessageAsync(message, ct);
    }
}
```

Setting `MessageId` aids dedup/idempotency. The client is injected (singleton, keyless) ([04-ServiceBus.md](04-ServiceBus.md)).
</details>

---

### Problem 5 — Process Service Bus messages with settlement

Consume with a processor, completing on success and dead-lettering invalid messages.

<details>
<summary>Solution</summary>

```csharp
var processor = client.CreateProcessor("orders", new ServiceBusProcessorOptions {
    MaxConcurrentCalls = 10, AutoCompleteMessages = false
});
processor.ProcessMessageAsync += async args => {
    try {
        var order = args.Message.Body.ToObjectFromJson<Order>();
        await HandleAsync(order);
        await args.CompleteMessageAsync(args.Message);                 // success
    } catch (JsonException) {
        await args.DeadLetterMessageAsync(args.Message, "BadFormat");  // poison → DLQ
    }
    // other exceptions: don't settle → lock expires → redelivered (transient)
};
processor.ProcessErrorAsync += args => { /* log */ return Task.CompletedTask; };
await processor.StartProcessingAsync();
```

Complete on success, dead-letter the unrecoverable, let transient failures redeliver ([04-ServiceBus.md](04-ServiceBus.md)).
</details>

---

### Problem 6 — Stream a blob upload

Upload a user's file to Blob Storage without buffering it in memory.

<details>
<summary>Solution</summary>

```csharp
public class FileService(BlobServiceClient blobs) {
    public async Task UploadAsync(Stream content, string name, string contentType, CancellationToken ct) {
        var container = blobs.GetBlobContainerClient("uploads");
        await container.CreateIfNotExistsAsync(cancellationToken: ct);
        var blob = container.GetBlobClient(name);
        await blob.UploadAsync(content, new BlobUploadOptions {
            HttpHeaders = new BlobHttpHeaders { ContentType = contentType }
        }, ct);
    }
}
```

Pass the request stream straight through — the SDK uploads in blocks, no full-file buffering ([05-BlobStorage.md](05-BlobStorage.md)).
</details>

---

### Problem 7 — Generate a SAS for direct client download

Let a browser download a private blob directly for 1 hour, without proxying through the server.

<details>
<summary>Solution</summary>

```csharp
var blob = container.GetBlobClient(name);
// User-delegation SAS (key-less, signed via managed identity) is preferred:
var sasUri = blob.GenerateSasUri(BlobSasPermissions.Read, DateTimeOffset.UtcNow.AddHours(1));
return sasUri;   // hand this URL to the client; it downloads directly from storage
```

A time-limited, read-only SAS offloads the download from your server; prefer **user-delegation SAS** (no account key) ([05-BlobStorage.md](05-BlobStorage.md)).
</details>

---

### Problem 8 — Cosmos point read and partition-aware query

Read one order by id, and query a customer's pending orders efficiently.

<details>
<summary>Solution</summary>

```csharp
// Point read — cheapest (id + partition key):
var order = await container.ReadItemAsync<Order>(id, new PartitionKey(customerId));

// Single-partition query (include the partition key):
var query = new QueryDefinition("SELECT * FROM c WHERE c.status = @s").WithParameter("@s", "Pending");
var it = container.GetItemQueryIterator<Order>(query,
    requestOptions: new QueryRequestOptions { PartitionKey = new PartitionKey(customerId) });
var results = new List<Order>();
while (it.HasMoreResults) results.AddRange(await it.ReadNextAsync());
```

Point read when you know id+partition key; scope queries to the partition key to stay single-partition (cheap RUs) ([06-CosmosDB.md](06-CosmosDB.md)).
</details>

---

### Problem 9 — Choose a Cosmos partition key

For an e-commerce orders container, evaluate `status`, `orderId`, and `customerId` as partition keys.

<details>
<summary>Solution</summary>

- **`status`** ❌ — very low cardinality (a few values) and skewed → **hot partition**, caps throughput. Bad.
- **`orderId`** ⚠️ — high cardinality and even, but most queries are "orders for a customer" → those become **cross-partition** (expensive). Good for point reads, bad for the common query.
- **`customerId`** ✅ — high cardinality, reasonably even, and **aligns with the common query** (a customer's orders → single-partition). Best choice, assuming no single customer dominates traffic.

Pick the key that is high-cardinality, evenly distributed, **and** matches your common access pattern ([06-CosmosDB.md](06-CosmosDB.md)).
</details>

---

### Problem 10 — Process the Cosmos change feed in a Function

React to order changes via the Cosmos DB trigger.

<details>
<summary>Solution</summary>

```csharp
public class OrderChangeHandler {
    [Function("OnOrderChanged")]
    public void Run([CosmosDBTrigger(
        databaseName: "ShopDb", containerName: "orders",
        Connection = "Cosmos", LeaseContainerName = "leases", CreateLeaseContainerIfNotExists = true)]
        IReadOnlyList<Order> changes) {
        foreach (var order in changes) { /* update a read model / send a notification */ }
    }
}
```

The change feed delivers inserts/updates in order; the lease container tracks progress so processing is reliable and resumable ([06-CosmosDB.md](06-CosmosDB.md), [03-Functions.md](03-Functions.md)).
</details>

---

### Problem 11 — Encrypt data using a Key Vault key (key never leaves the vault)

Encrypt a value using an RSA key stored in Key Vault.

<details>
<summary>Solution</summary>

```csharp
var keyClient = new KeyClient(new Uri("https://vault.vault.azure.net"), new DefaultAzureCredential());
KeyVaultKey key = await keyClient.GetKeyAsync("data-key");
var crypto = new CryptographyClient(key.Id, new DefaultAzureCredential());

EncryptResult enc = await crypto.EncryptAsync(EncryptionAlgorithm.RsaOaep, plaintextBytes);
DecryptResult dec = await crypto.DecryptAsync(EncryptionAlgorithm.RsaOaep, enc.Ciphertext);
```

`CryptographyClient` performs the operation **in the vault** — the private key never leaves it, so it's never exposed to the app ([07-KeyVault.md](07-KeyVault.md), [Ch10 §10](../10-Identity/10-Cryptography.md)).
</details>

---

### Problem 12 — A feature flag

Gate a new checkout flow behind a centrally-controlled feature flag.

<details>
<summary>Solution</summary>

```csharp
// Program.cs
builder.Configuration.AddAzureAppConfiguration(o =>
    o.Connect(new Uri("https://cfg.azconfig.io"), new DefaultAzureCredential()).UseFeatureFlags());
builder.Services.AddFeatureManagement();
```
```csharp
public class CheckoutController(IFeatureManager features) {
    public async Task<IActionResult> Index() =>
        await features.IsEnabledAsync("NewCheckout") ? View("NewCheckout") : View("Checkout");
}
```

The flag is toggled centrally in App Configuration without redeploying — decoupling release from deployment ([09-AppConfig.md](09-AppConfig.md)).
</details>

---

### Problem 13 — Emit telemetry to Application Insights via OpenTelemetry

Wire OpenTelemetry to export to App Insights and add a custom metric.

<details>
<summary>Solution</summary>

```csharp
// Modern path — OTel + Azure Monitor exporter (not the legacy SDK):
builder.Services.AddOpenTelemetry().UseAzureMonitor();
builder.Services.AddSingleton<OrderMetrics>();

public class OrderMetrics {
    private readonly Counter<int> _placed;
    public OrderMetrics(IMeterFactory mf) =>
        _placed = mf.Create("MyApp.Orders").CreateCounter<int>("orders.placed");
    public void OrderPlaced() => _placed.Add(1);
}
```

OTel instrumentation exports to App Insights via `UseAzureMonitor()`; the custom `Meter`/`Counter` ([Ch12 §05](../12-Observability/05-Metrics.md)) flows to App Insights and queries with KQL ([10-AppInsights.md](10-AppInsights.md)).
</details>

---

### Problem 14 — Spot the security mistake

```csharp
var blobs = new BlobServiceClient(
    "DefaultEndpointsProtocol=https;AccountName=acct;AccountKey=abc123...;");
```

What's wrong and how to fix it?

<details>
<summary>Solution</summary>

It uses a **storage account key** in a connection string — a powerful, leakable, hard-to-rotate secret (and likely committed to config/repo). Fix: use **managed identity + RBAC** (keyless):

```csharp
var blobs = new BlobServiceClient(
    new Uri("https://acct.blob.core.windows.net"), new DefaultAzureCredential());
// Grant the app's managed identity the "Storage Blob Data Contributor" role on the account.
```

No secret stored, access governed by RBAC, audited and revocable ([01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)).
</details>

---

### Problem 15 — Choose the right Azure messaging service

For each, pick Service Bus / Event Grid / Event Hubs and justify: (a) processing customer orders reliably with ordering per customer, (b) running a thumbnail generator whenever a blob is uploaded, (c) ingesting 2 million IoT sensor readings/second for analytics.

<details>
<summary>Solution</summary>

- **(a) Reliable ordered order processing → Service Bus**: discrete business messages needing reliability, dead-lettering, and **sessions** for per-customer ordering ([04-ServiceBus.md](04-ServiceBus.md)).
- **(b) React to blob upload → Event Grid**: a discrete "X happened" notification routed to a handler (Function) — Event Grid is the reactive glue for Azure resource events ([08-EventGrid-EventHubs.md](08-EventGrid-EventHubs.md), [03-Functions.md](03-Functions.md)).
- **(c) 2M readings/sec → Event Hubs**: high-volume **streaming** ingestion with partitions, consumer groups, and replay (Kafka-like) feeding analytics — not Service Bus (wrong scale) or Event Grid (not a stream) ([08-EventGrid-EventHubs.md](08-EventGrid-EventHubs.md)).
</details>
