# Azure Service Bus

## Microsoft's managed enterprise message broker

Azure Service Bus is a fully-managed, cloud message broker — the Azure-native choice for reliable enterprise messaging. It offers **queues** (point-to-point) and **topics/subscriptions** (pub-sub), with enterprise features: sessions (ordered/grouped processing), dead-lettering, scheduled/deferred messages, duplicate detection, and transactions. The .NET SDK is `Azure.Messaging.ServiceBus`.

```csharp
var client = new ServiceBusClient(connectionString);   // or DefaultAzureCredential (managed identity)

// Send
ServiceBusSender sender = client.CreateSender("orders");
await sender.SendMessageAsync(new ServiceBusMessage(JsonSerializer.SerializeToUtf8Bytes(order)));

// Receive (processor — recommended; handles the receive loop)
ServiceBusProcessor processor = client.CreateProcessor("orders", new ServiceBusProcessorOptions());
processor.ProcessMessageAsync += async args => {
    var order = JsonSerializer.Deserialize<Order>(args.Message.Body);
    await ProcessAsync(order);
    await args.CompleteMessageAsync(args.Message);     // ack — remove from the queue
};
processor.ProcessErrorAsync += args => { /* log */ return Task.CompletedTask; };
await processor.StartProcessingAsync();
```

> Often used via **MassTransit** ([04-MassTransit.md](04-MassTransit.md)) (`UsingAzureServiceBus`). This file covers Service Bus's distinctive concepts.

---

## Queues vs topics/subscriptions

Two messaging shapes:

```
Queue:    Producer → [Queue] → one consumer gets each message (competing consumers)

Topic:    Producer → [Topic] → Subscription A → consumer A
                              → Subscription B → consumer B   (each subscription gets a COPY)
```

- **Queue** — point-to-point; each message is consumed by **one** receiver (load-balanced across competing consumers). Like a RabbitMQ queue.
- **Topic + Subscriptions** — pub-sub; each **subscription** gets its **own copy** of every message (filtered by rules). Multiple independent consumers each process all relevant messages.

Use a **queue** for work distribution (one logical handler); a **topic** for events multiple services react to independently (subscriptions with SQL-like filter rules select which messages each cares about).

---

## Message settlement (complete / abandon / dead-letter)

Like RabbitMQ acks, Service Bus uses **peek-lock**: a received message is **locked** (invisible to others) while you process it, then you settle it:

```csharp
processor.ProcessMessageAsync += async args => {
    try {
        await ProcessAsync(args.Message);
        await args.CompleteMessageAsync(args.Message);       // success → remove
    } catch (TransientException) {
        await args.AbandonMessageAsync(args.Message);         // release the lock → redelivered
    } catch (PermanentException) {
        await args.DeadLetterMessageAsync(args.Message, "PoisonMessage", "details");  // → DLQ
    }
};
```

- **Complete** — processed successfully; remove from the queue.
- **Abandon** — release the lock so the message is redelivered (transient failure). After `MaxDeliveryCount` attempts, it auto-dead-letters.
- **Dead-letter** — move to the **dead-letter sub-queue** explicitly (poison message). 

This peek-lock + settlement model gives at-least-once delivery (lock expiry or abandon → redelivery), the same reliability principle as RabbitMQ acks. Every queue/subscription has a built-in **dead-letter queue** for messages that exceed `MaxDeliveryCount` or are dead-lettered explicitly — inspect/replay them.

---

## Sessions — ordered & grouped processing

Brokers generally don't guarantee ordering across competing consumers. **Sessions** provide FIFO ordering and stateful grouping for related messages (e.g., all events for one order processed in order, by one consumer):

```csharp
var sender = client.CreateSender("orders");
await sender.SendMessageAsync(new ServiceBusMessage(body) { SessionId = orderId.ToString() });

var sessionProcessor = client.CreateSessionProcessor("orders");
sessionProcessor.ProcessMessageAsync += async args => {
    // all messages with the same SessionId go to the SAME consumer, IN ORDER
    await ProcessAsync(args.Message);
    await args.CompleteMessageAsync(args.Message);
};
```

Messages sharing a `SessionId` are delivered in order to a single consumer (which holds the session lock) — giving **per-key ordering** without serializing the whole queue. Use sessions when related messages must be processed sequentially (per-order, per-customer, per-aggregate). (Kafka achieves similar ordering via partitions — [06-Kafka.md](06-Kafka.md).)

---

## Advanced features

Service Bus's enterprise feature set:

- **Scheduled messages** — `ScheduleMessageAsync(message, enqueueTime)` delivers a message at a future time (delayed processing, reminders).
- **Deferred messages** — set aside a message to process later by sequence number (out-of-order workflows).
- **Duplicate detection** — the broker rejects duplicate `MessageId`s within a window (helps idempotency).
- **Auto-forwarding** — chain a queue/subscription to another entity.
- **Transactions** — group send/complete operations atomically within Service Bus.
- **Message TTL & auto-expiry**.

These make Service Bus suited to complex enterprise workflows; you won't use all of them, but they're there when needed.

---

## Authentication: prefer managed identity

```csharp
// ✗ — connection string with embedded keys (a secret to manage/rotate/leak)
var client = new ServiceBusClient(connectionStringWithKey);

// ✓ — managed identity (no secrets in config; Azure RBAC controls access)
var client = new ServiceBusClient("myns.servicebus.windows.net", new DefaultAzureCredential());
```

Prefer **`DefaultAzureCredential`** (managed identity in Azure, developer credentials locally) over connection strings with shared-access keys — no secrets to store/rotate/leak, access controlled by Azure RBAC. This is the modern Azure auth pattern across all Azure SDKs ([Ch20 Azure](../20-AzureIntegration/README.md)).

---

## Service Bus vs RabbitMQ vs Kafka

| | Azure Service Bus | RabbitMQ | Kafka |
|---|---|---|---|
| Hosting | fully managed (Azure) | self-host or managed | self-host or managed |
| Model | queues + topics | exchanges/queues | partitioned logs |
| Ordering | sessions (per-key) | limited | per-partition |
| Best for | enterprise workflows on Azure | flexible routing, on-prem/cloud | high-throughput event streaming |
| Replay | no (consumed = gone) | no | **yes** (retained log) |

Choose **Service Bus** for managed enterprise messaging on Azure (sessions, dead-letter, scheduling); **RabbitMQ** for flexible routing and platform independence; **Kafka** for high-throughput event streaming and replay ([06-Kafka.md](06-Kafka.md)). Service Bus is the path of least resistance if you're already on Azure.

---

## Common gotchas

### Forgetting to complete messages

Not calling `CompleteMessageAsync` (with peek-lock) means the lock eventually expires and the message is **redelivered** → reprocessed. Complete on success; abandon/dead-letter on failure.

### Lock expiry on long processing

If processing exceeds the lock duration, the lock expires and the message is redelivered (possibly while you're still processing it). Renew the lock for long work, or keep processing under the lock timeout.

### Expecting ordering without sessions

Competing consumers don't preserve order. For ordered processing of related messages, use **sessions** (per-`SessionId` FIFO to one consumer).

### Non-idempotent consumers

At-least-once delivery + redelivery means duplicates. Make consumers idempotent (or use duplicate detection) — [07-Patterns.md](07-Patterns.md).

### Connection-string secrets

Embedding SAS keys in config is a secret-management burden and leak risk. Use **managed identity** (`DefaultAzureCredential`).

### Topic when you meant queue (or vice versa)

A topic copies each message to every subscription; a queue delivers each to one consumer. Using a topic for work distribution duplicates processing; using a queue for events means only one consumer reacts. Match the shape to the need.

---

## Summary

- **Azure Service Bus** is a fully-managed cloud broker with **queues** (point-to-point, one consumer per message) and **topics/subscriptions** (pub-sub, each subscription gets a copy, filtered by rules).
- **Peek-lock + settlement** (Complete / Abandon / Dead-letter) gives at-least-once delivery with a built-in **dead-letter queue** after `MaxDeliveryCount`.
- **Sessions** provide per-key FIFO ordering to a single consumer (ordered processing of related messages) without serializing the whole queue; plus enterprise features (scheduled/deferred messages, duplicate detection, transactions).
- Prefer **managed identity** (`DefaultAzureCredential`) over connection-string keys; make consumers **idempotent** (at-least-once delivery).
- Choose Service Bus for managed enterprise messaging on Azure; RabbitMQ for flexible/portable routing; Kafka for high-throughput streaming + replay. Often used via **MassTransit**.

→ Next: [06-Kafka.md](06-Kafka.md)
