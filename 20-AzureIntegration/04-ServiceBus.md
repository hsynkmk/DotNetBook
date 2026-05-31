# Azure Service Bus

## Enterprise messaging on Azure

**Azure Service Bus** is Azure's fully-managed enterprise message broker — the cloud equivalent of RabbitMQ ([Ch07 §03](../07-Messaging/03-RabbitMQ.md)) — for reliable, asynchronous, decoupled communication between services. It offers **queues** (point-to-point) and **topics/subscriptions** (publish-subscribe), with enterprise features: **sessions** (ordered/grouped processing), **dead-lettering** (poison-message handling), scheduled/deferred delivery, duplicate detection, and transactions. The .NET client is `Azure.Messaging.ServiceBus`, following the standard Azure SDK pattern ([01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)) — construct with a namespace + `DefaultAzureCredential`, register in DI.

```csharp
builder.Services.AddAzureClients(c =>
    c.AddServiceBusClient("myns.servicebus.windows.net")     // namespace, not a connection string
     .UseCredential(new DefaultAzureCredential()));           // keyless

// Send:
var sender = client.CreateSender("orders");
await sender.SendMessageAsync(new ServiceBusMessage(BinaryData.FromObjectAsJson(order)));
```

---

## Queues vs topics

Service Bus has two messaging shapes ([Ch07 §01](../07-Messaging/01-ChannelOfT.md) covers the general concepts):

- **Queue** — **point-to-point**: one queue, messages consumed by *one* receiver (competing consumers for scale). Use for work distribution / commands where exactly one handler should process each message.
- **Topic + subscriptions** — **publish-subscribe**: a publisher sends to a topic; *each* subscription gets its own copy, optionally filtered. Use for broadcasting an event to multiple independent consumers (each subscription is an independent queue with its own cursor).

```csharp
// Topic publish; each subscription receives a copy (filterable by rules):
var sender = client.CreateSender("order-events");
await sender.SendMessageAsync(new ServiceBusMessage(BinaryData.FromObjectAsJson(orderPlaced)));
```

Topics with **subscription filters** (SQL/correlation rules) let each subscriber receive only the messages it cares about — routing at the broker.

---

## Receiving and the processor

The recommended way to consume is the **`ServiceBusProcessor`** — an event-driven, auto-managed receiver that handles concurrency, message completion, and error callbacks:

```csharp
var processor = client.CreateProcessor("orders", new ServiceBusProcessorOptions {
    MaxConcurrentCalls = 10,
    AutoCompleteMessages = false               // complete explicitly after success
});
processor.ProcessMessageAsync += async args => {
    var order = args.Message.Body.ToObjectFromJson<Order>();
    await HandleAsync(order);
    await args.CompleteMessageAsync(args.Message);   // remove from the queue on success
};
processor.ProcessErrorAsync += args => { /* log */ return Task.CompletedTask; };
await processor.StartProcessingAsync();
```

Service Bus is **at-least-once**: a message is locked while being processed and removed on **complete**; if processing fails (or the lock expires), it's redelivered. So handlers must be **idempotent** ([Ch07 §07](../07-Messaging/07-Patterns.md)) — a redelivery shouldn't double-process.

---

## Settlement: complete, abandon, dead-letter

After receiving (in PeekLock mode), you **settle** the message:

- **Complete** — success; remove it from the queue.
- **Abandon** — release the lock so it's redelivered (transient failure, try again).
- **Dead-letter** — move it to the **dead-letter queue (DLQ)** — a poison message that repeatedly fails or is invalid, set aside for inspection rather than blocking the queue.

```csharp
await args.DeadLetterMessageAsync(args.Message, "ParseError", "Body was not valid JSON");
```

Messages also auto-dead-letter after exceeding **`MaxDeliveryCount`** (too many failed attempts). The **DLQ** is essential operationally — it isolates poison messages so one bad message doesn't stall processing, and gives you a place to investigate/replay failures ([Ch07 §07](../07-Messaging/07-Patterns.md)).

---

## Sessions — ordered/grouped processing

By default Service Bus doesn't guarantee ordering (competing consumers process in parallel). **Sessions** provide **FIFO ordering within a session** and group related messages: all messages with the same `SessionId` are processed **in order by a single consumer** at a time:

```csharp
var msg = new ServiceBusMessage(body) { SessionId = order.CustomerId };   // group by customer
// A session processor handles one session's messages in order:
var sessionProcessor = client.CreateSessionProcessor("orders", new ServiceBusSessionProcessorOptions { ... });
```

Use sessions when **order matters** for a logical entity (all events for one order/customer must be processed sequentially) — e.g., a state machine where out-of-order messages would corrupt state. Without sessions, you get higher throughput but no ordering guarantee.

---

## Other enterprise features

- **Scheduled delivery** — send a message to appear at a future time (`ScheduledEnqueueTime`) — for delays/reminders.
- **Deferral** — set a message aside to process later by sequence number.
- **Duplicate detection** — the broker drops duplicate `MessageId`s within a window — helps idempotency.
- **Transactions** — send/complete multiple messages atomically.
- **Message TTL** and **auto-forwarding** between entities.

These are why Service Bus is "enterprise" messaging vs a simple queue — it supports complex, reliable workflows.

---

## Common gotchas

### Non-idempotent handlers (at-least-once)

Service Bus redelivers on failure/lock-expiry, so a handler can run more than once per message. Make handlers **idempotent** (dedupe by id, or use duplicate detection) — never assume exactly-once ([Ch07 §07](../07-Messaging/07-Patterns.md)).

### Ignoring the dead-letter queue

Poison messages auto-dead-letter after `MaxDeliveryCount`; if you never monitor the DLQ, failures vanish silently. Monitor/alert on the DLQ and have a replay/inspection process.

### Expecting ordering without sessions

Competing consumers process in parallel — no global order. Use **sessions** (group by `SessionId`) when ordering within an entity matters.

### Lock expiry on long processing

A message's lock has a duration; if handling exceeds it, the lock is lost and the message is redelivered (possibly causing duplicate work). Renew the lock for long operations or keep handlers short.

### Connection strings over managed identity

Prefer **managed identity + `DefaultAzureCredential`** over namespace connection strings/SAS keys ([01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)) — keyless, no secret to leak.

---

## Summary

- **Azure Service Bus** is Azure's managed enterprise broker — **queues** (point-to-point, one consumer per message) and **topics/subscriptions** (pub-sub, each subscription gets a filtered copy) — for reliable, decoupled async messaging; client is `Azure.Messaging.ServiceBus` with **managed identity** ([01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)).
- Consume with **`ServiceBusProcessor`** (concurrency, error callbacks); it's **at-least-once**, so **settle** messages — **Complete** (success), **Abandon** (retry), **Dead-letter** (poison) — and make handlers **idempotent** ([Ch07 §07](../07-Messaging/07-Patterns.md)).
- The **dead-letter queue** isolates poison/failed messages (auto after `MaxDeliveryCount`) so they don't block the queue — monitor it. **Sessions** give **FIFO ordering within a `SessionId`** for entities where order matters.
- Enterprise features — scheduled/deferred delivery, **duplicate detection**, transactions, TTL, auto-forward — support complex reliable workflows; watch **lock expiry** on long processing and prefer keyless auth.

→ Next: [05-BlobStorage.md](05-BlobStorage.md)
