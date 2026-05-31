# Event Grid and Event Hubs

## Two event services — and Service Bus — for different jobs

Azure has **three** messaging/eventing services that are easy to confuse: **Service Bus** ([04-ServiceBus.md](04-ServiceBus.md)), **Event Grid**, and **Event Hubs**. They look similar ("things that move messages") but solve **different problems**, and picking the right one matters. The simplest framing:

| Service | Pattern | For | Scale/throughput |
|---|---|---|---|
| **Service Bus** | enterprise **messaging** (commands/work) | reliable, ordered, transactional business messages | moderate, per-message features |
| **Event Grid** | **event notification** (pub-sub routing) | reactive "X happened" events → handlers | high, lightweight |
| **Event Hubs** | **event streaming** (telemetry/big data) | massive ingest of events/telemetry streams | **millions/sec**, streaming |

The distinction: **Service Bus** = discrete messages you *process* (each matters, often a command/work item); **Event Grid** = lightweight *notifications* routed to subscribers ("react to this"); **Event Hubs** = high-volume *streams* you *ingest and analyze* (telemetry, logs, IoT, clickstreams).

---

## Event Grid — reactive event routing

**Event Grid** is a fully-managed **event routing** service: publishers emit events, and Event Grid delivers them to **subscribers** (Functions, webhooks, queues, etc.) based on **subscriptions with filters**. It's the backbone of **reactive, event-driven** architectures on Azure — "when *this* happens, run *that*":

```csharp
// A Function reacting to a Blob Storage event routed via Event Grid:
[Function("OnBlobUploaded")]
public void Run([EventGridTrigger] CloudEvent e) {
    // e.g., a blob was created — generate a thumbnail, index it, etc.
}
```

- **Source events**: many Azure services emit events to Event Grid natively — a blob uploaded ([05-BlobStorage.md](05-BlobStorage.md)), a resource created, a Cosmos change — plus your own **custom events**.
- **Push delivery** with retries; lightweight, near-real-time, pay-per-event.
- **Filtering** routes only relevant events to each subscriber.

Use Event Grid to **react to discrete things happening** (especially Azure resource events) and wire serverless handlers ([03-Functions.md](03-Functions.md)) — it's the "glue" for event-driven Azure.

---

## Event Hubs — high-throughput streaming

**Event Hubs** is a **big-data streaming** ingestion service designed for **massive volumes** — millions of events per second — like telemetry, application logs, IoT sensor data, and clickstreams. It's Azure's equivalent of **Apache Kafka** ([Ch07 §06](../07-Messaging/06-Kafka.md)) (and is Kafka-protocol compatible):

- **Partitioned log** — events are appended to partitions; **consumers read by offset** and can replay (events are retained for a window, not deleted on read — unlike a queue).
- **Consumer groups** — multiple independent readers each track their own position over the same stream.
- **Throughput units / processing units** scale ingestion.

```csharp
// Producing to an Event Hub:
await using var producer = new EventHubProducerClient("ns.servicebus.windows.net", "telemetry",
    new DefaultAzureCredential());
using var batch = await producer.CreateBatchAsync();
batch.TryAdd(new EventData(BinaryData.FromObjectAsJson(reading)));
await producer.SendAsync(batch);
```

Use Event Hubs when you're **ingesting a firehose** of events to process/analyze downstream (stream processing, analytics, storage) — *not* for discrete business commands (that's Service Bus) or simple notifications (Event Grid). It feeds analytics pipelines (Stream Analytics, Spark, Functions).

---

## Choosing between them

```
Need reliable processing of discrete business messages/commands
  (ordering, sessions, dead-letter, transactions)?            → Service Bus ([04-ServiceBus.md])
Need to react to discrete events ("X happened"), route to
  handlers, especially Azure resource events?                 → Event Grid
Need to ingest/stream HIGH-VOLUME telemetry/events for
  analytics, with replay and consumer groups?                 → Event Hubs (Kafka-like)
```

They also **compose**: e.g., Event Hubs ingests a telemetry stream; Event Grid notifies when a blob lands; Service Bus carries the resulting commands. A real system often uses more than one, each for its strength. The mistake is using one where another fits — e.g., Event Hubs for a few business commands (overkill, wrong semantics) or Service Bus for millions of telemetry events/sec (wrong scale model).

---

## Relationship to the messaging chapter

These map onto the general messaging concepts in [Ch07](../07-Messaging/README.md): **Service Bus ≈ RabbitMQ** ([Ch07 §03](../07-Messaging/03-RabbitMQ.md)) (enterprise broker), **Event Hubs ≈ Kafka** ([Ch07 §06](../07-Messaging/06-Kafka.md)) (partitioned streaming log with replay/consumer groups), and **Event Grid** is the cloud-native event-routing layer. The messaging **patterns** — idempotency, outbox, at-least-once delivery ([Ch07 §07](../07-Messaging/07-Patterns.md)) — apply to all three; choose the service by **semantics and scale**, then apply the patterns.

---

## Common gotchas

### Confusing the three services

They're not interchangeable: **Service Bus** (discrete reliable messages/commands), **Event Grid** (event notifications/routing), **Event Hubs** (high-volume streaming). Match by semantics and scale, not surface similarity.

### Event Hubs for low-volume business messages

Event Hubs is for streams (millions/sec) you analyze, not for a handful of business commands needing per-message reliability/ordering/dead-letter — use **Service Bus** for that.

### Service Bus for massive telemetry ingestion

Per-message broker semantics don't fit a firehose of telemetry. Use **Event Hubs** (streaming, partitioned, replayable) for high-volume ingestion.

### Forgetting replay semantics differ

Event Hubs **retains** events (read by offset, replayable, consumer groups); Service Bus **removes** messages on completion. Don't expect queue semantics from a stream or vice versa.

### Non-idempotent handlers

All three deliver at-least-once (or with redelivery); handlers must be **idempotent** ([Ch07 §07](../07-Messaging/07-Patterns.md)) regardless of which service.

---

## Summary

- Azure has three easily-confused messaging/eventing services for **different** jobs: **Service Bus** (enterprise **messaging** — discrete reliable commands/work, ordering/sessions/dead-letter — [04-ServiceBus.md](04-ServiceBus.md)), **Event Grid** (**event notification/routing** — react to "X happened", especially Azure resource events, → serverless handlers), and **Event Hubs** (**event streaming** — millions/sec telemetry ingestion, Kafka-like).
- **Event Grid**: lightweight push-based pub-sub with filtered subscriptions — the glue for **reactive** event-driven Azure (e.g., blob-uploaded → Function — [03-Functions.md](03-Functions.md)).
- **Event Hubs**: a partitioned, **replayable streaming log** with **consumer groups** (Kafka-equivalent — [Ch07 §06](../07-Messaging/06-Kafka.md)) for **high-volume** ingestion feeding analytics.
- **Choose by semantics and scale** (Service Bus = reliable commands, Event Grid = notifications, Event Hubs = streams); they **compose** in real systems, and messaging **patterns** (idempotency, at-least-once — [Ch07 §07](../07-Messaging/07-Patterns.md)) apply to all.

→ Next: [09-AppConfig.md](09-AppConfig.md)
