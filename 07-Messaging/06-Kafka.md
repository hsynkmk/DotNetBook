# Kafka

## A distributed event log, not a traditional queue

Apache Kafka is fundamentally different from RabbitMQ/Service Bus: it's a **distributed, append-only log** of events, not a queue that deletes messages on consumption. Producers append events to **topics** (split into **partitions**); consumers read by tracking their **offset** (position) in the log. Messages are **retained** (for days/forever), so multiple consumers can read independently and **replay** history. The .NET client is `Confluent.Kafka`.

```csharp
// Producer
using var producer = new ProducerBuilder<string, string>(
    new ProducerConfig { BootstrapServers = "localhost:9092" }).Build();
await producer.ProduceAsync("orders",
    new Message<string, string> { Key = orderId.ToString(), Value = json });   // key → partition

// Consumer
using var consumer = new ConsumerBuilder<string, string>(new ConsumerConfig {
    BootstrapServers = "localhost:9092",
    GroupId = "order-processors",            // consumer group
    AutoOffsetReset = AutoOffsetReset.Earliest
}).Build();
consumer.Subscribe("orders");
while (!ct.IsCancellationRequested) {
    var result = consumer.Consume(ct);
    await ProcessAsync(result.Message.Value);
    consumer.Commit(result);                  // commit the offset (we've processed up to here)
}
```

---

## The log model: topics, partitions, offsets

```
Topic "orders":
  Partition 0: [evt0][evt1][evt2][evt3]...   ← append-only; each event has an OFFSET (0,1,2,...)
  Partition 1: [evt0][evt1][evt2]...
  Partition 2: [evt0][evt1]...

Consumers track their OFFSET per partition; messages are RETAINED (not deleted on read).
```

- **Topic** — a named stream of events, split into **partitions** for parallelism/scale.
- **Partition** — an ordered, immutable sequence of events. Ordering is guaranteed **within** a partition (not across partitions).
- **Offset** — a consumer's position in a partition. Consumers commit offsets to record progress; they can also **rewind** (re-read from an earlier offset) — enabling replay.
- **Retention** — events stay for a configured time/size regardless of consumption, so new consumers can read history and existing ones can replay.

This log-not-queue model is Kafka's defining trait: messages aren't consumed-and-deleted; they're **read by offset and retained**.

---

## Partition key → ordering & parallelism

The producer's **message key** determines the partition (same key → same partition), which gives **per-key ordering** and controls parallelism:

```csharp
await producer.ProduceAsync("orders", new Message<string, string> {
    Key = orderId.ToString(),    // all events for one order → same partition → processed IN ORDER
    Value = json
});
```

- All events with the same key land in the same partition, so they're **ordered** relative to each other (e.g., all events for `order-42` in sequence).
- Different keys spread across partitions for **parallelism** (more partitions = more parallel consumers).

Choose the key for the ordering boundary you need (per-order, per-customer, per-device). This is Kafka's answer to ordered processing — analogous to Service Bus **sessions** ([05-AzureServiceBus.md](05-AzureServiceBus.md)), but built into the partition model.

---

## Consumer groups — scaling and fan-out

```
Topic (3 partitions) + Consumer Group "processors" (3 consumers):
  → each consumer is assigned partitions; the group collectively reads all partitions ONCE (load-balanced)

Two DIFFERENT groups ("processors" and "analytics") each read ALL messages independently (fan-out).
```

- Within **one consumer group**, partitions are distributed across consumers — each message is processed by **one** consumer in the group (scaling: add consumers up to the partition count).
- **Different consumer groups** each get their **own** copy of the stream (independent offsets) — so multiple services consume the same events independently (fan-out / pub-sub).

This dual behavior — load-balance within a group, fan-out across groups — is how Kafka serves both work distribution and event broadcasting. Max parallelism per group = the partition count (so plan partitions for expected scale).

---

## Offset commits & delivery semantics

When you commit offsets determines the delivery guarantee:

```csharp
// At-least-once (common): process THEN commit. Crash before commit → reprocess (duplicate possible).
await ProcessAsync(result.Message.Value);
consumer.Commit(result);

// At-most-once: commit THEN process. Crash after commit → message lost (no duplicate).
consumer.Commit(result);
await ProcessAsync(result.Message.Value);   // risk: lost if this fails
```

- **At-least-once** (process then commit) — the common, safe choice; duplicates possible on crash → consumers must be **idempotent** ([07-Patterns.md](07-Patterns.md)).
- **At-most-once** (commit then process) — no duplicates but possible loss; rarely what you want.
- **Exactly-once** — Kafka supports it via transactions/idempotent producers for stream-processing scenarios, but it's complex; most apps use **at-least-once + idempotent consumers**.

Auto-commit (`EnableAutoCommit`) is convenient but commits on a timer (can lose/duplicate around crashes) — for control, commit manually after processing.

---

## When to use Kafka

Kafka shines for:
- **High-throughput event streaming** (millions of events/sec) — its design goal.
- **Event sourcing / replay** — retained log lets you rebuild state by replaying events.
- **Multiple independent consumers** of the same stream (analytics + processing + audit, each its own group).
- **Stream processing** (Kafka Streams, ksqlDB) — transforming streams.
- **Decoupling via durable event history**, not just transient queuing.

When **not** Kafka: simple task queues, request/response, complex per-message routing, or low-volume messaging — RabbitMQ/Service Bus are simpler and offer richer per-message features (priorities, scheduled delivery, flexible routing). Kafka trades those for throughput, retention, and replay.

---

## Kafka vs RabbitMQ/Service Bus

| | Kafka | RabbitMQ / Service Bus |
|---|---|---|
| Model | retained partitioned **log** | **queue** (consume = remove) |
| Replay | **yes** (re-read by offset) | no |
| Throughput | very high | high |
| Ordering | per-partition (by key) | limited / sessions |
| Per-message features | minimal | rich (priority, TTL, scheduling, routing) |
| Best for | event streaming, replay, analytics | task queues, workflows, flexible routing |

The mental shift: Kafka is a **durable event log you read by position**, not a **queue you drain**. Pick it for streaming/replay/high-throughput; pick a traditional broker for queuing/workflows/rich routing.

---

## Common gotchas

### Treating Kafka like a queue

Messages aren't deleted on consumption — they're retained and read by offset. Thinking "consume = remove" leads to confusion (e.g., expecting a message to be gone, or not understanding consumer groups/replay).

### Expecting global ordering

Ordering is **per-partition**, not across the whole topic. If you need related events ordered, give them the same **key** (same partition). Don't assume topic-wide order.

### Non-idempotent consumers with at-least-once

Process-then-commit means duplicates on crash. Consumers must be **idempotent** ([07-Patterns.md](07-Patterns.md)) — Kafka's high-throughput retry/replay makes duplicates routine.

### Too few partitions

Max parallelism per consumer group = partition count. Under-partitioning caps your scaling (you can't add more useful consumers than partitions). Plan partitions for expected throughput (and they're hard to reduce later).

### Auto-commit losing/duplicating

`EnableAutoCommit` commits on a timer regardless of processing success. For precise semantics, disable it and commit manually after processing.

### Using Kafka for simple queuing

Kafka's operational complexity (partitions, offsets, brokers, ZooKeeper/KRaft) isn't worth it for a simple task queue. Use RabbitMQ/Service Bus (or an in-process channel) for that.

---

## Summary

- **Kafka** is a distributed, append-only **event log** (not a queue): producers append to **topics** split into **partitions**; consumers read by **offset**; messages are **retained** and **replayable** — the defining difference from RabbitMQ/Service Bus.
- The **partition key** routes related events to the same partition for **per-key ordering**; partitions enable parallelism (max consumers per group = partition count).
- **Consumer groups** load-balance within a group (each message once) and fan out across groups (each group reads everything independently).
- Commit offsets **after** processing for **at-least-once** delivery → make consumers **idempotent**; auto-commit risks loss/duplication.
- Use Kafka for **high-throughput streaming, event sourcing/replay, and multiple independent consumers**; use RabbitMQ/Service Bus for task queues, workflows, and rich per-message routing.

→ Next: [07-Patterns.md](07-Patterns.md)
