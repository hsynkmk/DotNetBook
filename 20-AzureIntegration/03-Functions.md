# Azure Functions

## Serverless event-driven compute

**Azure Functions** is Azure's serverless compute platform: you write small functions that run **in response to events** — an HTTP request, a queue message, a timer, a blob upload — and Azure handles the hosting, scaling (including scale-to-zero), and infrastructure. You pay per execution (on the Consumption plan), and the platform scales out automatically under load. For .NET, the modern model is the **isolated worker** (.NET 8+, current in .NET 10): your function runs in its **own process**, decoupled from the Functions host, giving you full control over the runtime, DI, middleware, and .NET version.

```csharp
public class OrderFunctions(ILogger<OrderFunctions> logger) {
    [Function("ProcessOrder")]
    public void Run([QueueTrigger("orders")] Order order) {     // triggered by a queue message
        logger.LogInformation("Processing order {Id}", order.Id);
        // ... handle the order ...
    }
}
```

---

## The isolated worker model

There were historically two models; the **isolated worker process** is now the standard (the older in-process model is being retired):

| | In-process (legacy) | **Isolated worker (.NET 8+)** |
|---|---|---|
| Where it runs | inside the Functions host | **your own process** |
| .NET version | tied to the host | **any supported .NET** (incl. latest) |
| DI / middleware | limited | **full** (`HostBuilder`, middleware pipeline) |
| Dependency conflicts | shared with host | **isolated** |

The isolated model means a Function app is configured like any .NET host ([Ch03 §01](../03-HostingAndDI/01-GenericHost.md)) — you build a host, register DI services, add middleware — and your code runs separately from the Functions runtime, avoiding version/dependency conflicts and giving full control:

```csharp
var host = new HostBuilder()
    .ConfigureFunctionsWorkerDefaults()
    .ConfigureServices(services => {
        services.AddSingleton<IOrderService, OrderService>();   // normal DI
        services.AddAzureClients(c => c.AddServiceBusClient("ns.servicebus.windows.net")
                                       .UseCredential(new DefaultAzureCredential()));
    })
    .Build();
host.Run();
```

---

## Triggers and bindings

The Functions programming model centers on **triggers** (what *starts* a function) and **bindings** (declarative input/output connections to other services), expressed via attributes — so you write business logic, not boilerplate plumbing:

```csharp
[Function("ResizeImage")]
[BlobOutput("thumbnails/{name}")]                          // OUTPUT binding: return value → blob
public byte[] Run(
    [BlobTrigger("uploads/{name}")] byte[] image,          // TRIGGER: runs when a blob is uploaded
    [QueueOutput("processed")] out string message)         // OUTPUT binding: write to a queue
{
    message = "done";
    return Resize(image);
}
```

| Trigger | Fires on |
|---|---|
| `HttpTrigger` | an HTTP request |
| `QueueTrigger` / `ServiceBusTrigger` | a message arrives ([04-ServiceBus.md](04-ServiceBus.md)) |
| `BlobTrigger` | a blob is created/updated ([05-BlobStorage.md](05-BlobStorage.md)) |
| `TimerTrigger` | a schedule (CRON) — like a cron job |
| `EventGridTrigger` / `EventHubTrigger` | events ([08-EventGrid-EventHubs.md](08-EventGrid-EventHubs.md)) |
| `CosmosDBTrigger` | change feed ([06-CosmosDB.md](06-CosmosDB.md)) |

**Bindings** connect inputs/outputs declaratively (read a blob, write a queue message, return an HTTP response) without manually constructing clients — the platform wires them. (In the isolated model you can also use the SDK clients directly for more control.)

---

## Hosting plans and scaling

How Functions scale and what you pay depends on the plan:

- **Consumption** — true serverless: **scale to zero** (pay nothing when idle), auto-scale out per load, pay per execution. Best for spiky/event-driven workloads; downside is **cold starts**.
- **Premium / Flex Consumption** — pre-warmed instances (no/low cold start), VNet integration, longer runs — for latency-sensitive or always-on needs.
- **Dedicated (App Service plan)** — runs on a fixed App Service plan ([02-AppService.md](02-AppService.md)) — predictable cost, no scale-to-zero.

**Cold starts** matter on Consumption (an idle app must spin up before serving) — mitigate with **Premium/Flex** plans, or **Native AOT** ([Ch19 §06](../19-Deployment/06-NativeAOT.md)) which dramatically cuts startup time (Functions supports AOT for compatible apps).

---

## Identity and configuration

Functions follow the same Azure patterns: use **managed identity + `DefaultAzureCredential`** ([01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)) for keyless access to Storage/Service Bus/etc., pull config from **App Settings**/**Key Vault references** ([07-KeyVault.md](07-KeyVault.md)), and emit telemetry to **Application Insights** ([10-AppInsights.md](10-AppInsights.md)). Triggers/bindings that connect to services should authenticate via **identity-based connections** (managed identity) rather than connection strings where supported — keyless, like the rest of the chapter.

---

## Common gotchas

### Using the legacy in-process model for new apps

The in-process model is being retired. Use the **isolated worker** model (.NET 8+) for new Functions — full DI/middleware, any .NET version, no host coupling.

### Cold starts on Consumption

An idle Consumption app must warm up before serving — latency spikes for the first request. Use **Premium/Flex** plans or **Native AOT** for latency-sensitive functions.

### Long-running work in a function

Functions are meant for short, event-driven work (Consumption has execution time limits). For long/durable workflows, use **Durable Functions** or offload to a worker service ([Ch08](../08-BackgroundProcessing/README.md)); don't run hours-long jobs in a regular function.

### Connection strings instead of identity-based connections

Many triggers/bindings support **managed identity** ("identity-based connections"). Prefer them over storing connection strings/keys in App Settings ([01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)).

### Non-idempotent message handling

Queue/Service Bus triggers can deliver a message more than once (at-least-once). Make handlers **idempotent** ([Ch07 §07](../07-Messaging/07-Patterns.md)) so a redelivery doesn't double-process.

---

## Summary

- **Azure Functions** is serverless, **event-driven** compute — functions run on **triggers** (HTTP, queue/Service Bus, blob, timer, Event Grid/Hub, Cosmos change feed) with Azure handling hosting and **auto-scaling** (including scale-to-zero), billed per execution.
- The modern model is the **isolated worker** (.NET 8+): your function runs in **its own process** with **full DI/middleware** and any .NET version, decoupled from the host — configured like a normal .NET host.
- **Bindings** declaratively connect inputs/outputs (read a blob, write a queue, return HTTP) without manual client plumbing; use SDK clients directly when you need more control.
- **Plans**: Consumption (scale-to-zero, cheap, **cold starts**), Premium/Flex (pre-warmed, VNet), Dedicated (App Service plan). Mitigate cold starts with Premium/Flex or **Native AOT**. Use **managed identity** ([01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)), Key Vault, and App Insights; keep handlers **idempotent** and functions **short** (Durable Functions for long workflows).

→ Next: [04-ServiceBus.md](04-ServiceBus.md)
