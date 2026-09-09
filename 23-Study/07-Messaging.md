# 07 — Messaging

## ⚡ 30-second answer

Messaging decouples producers from consumers. **In-process**, that's **`Channel<T>`** — an async
producer/consumer queue; use a **bounded** channel so a slow consumer applies **back-pressure**
instead of exhausting memory. **Cross-process**, you need a **broker**: **RabbitMQ** (flexible
routing via exchanges), **Azure Service Bus** (managed, enterprise features), or **Kafka** (a
replayable, partitioned **event log**, not a queue). Every broker delivers **at-least-once**, so
the non-negotiable rule is **make every consumer idempotent**. The other non-negotiable is the
**outbox pattern**: you cannot atomically write your database *and* publish to a broker, so write
the message to an outbox table in the same transaction and relay it afterwards. Ordering is
per-key, not global — Kafka **partitions**, Service Bus **sessions**. Failures go **retry →
dead-letter**, never a poison-message loop.

---

## Core mechanics

### `Channel<T>` — in-process

```csharp
var channel = Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(1000)
{
    FullMode = BoundedChannelFullMode.Wait      // producer awaits when full → back-pressure
});

await channel.Writer.WriteAsync(item, ct);      // producer
await foreach (var item in channel.Reader.ReadAllAsync(ct))   // consumer
    await ProcessAsync(item, ct);

channel.Writer.Complete();                      // ← without this, ReadAllAsync never finishes
```

Bounded with `Wait` gives back-pressure; `DropOldest`/`DropWrite` shed load under overload;
unbounded risks unbounded memory. Prefer `Channel<T>` over `BlockingCollection<T>` in async code —
no blocked threads, no starvation.

It is **in-process only**: no durability, contents lost on restart, single process. Graduate to a
broker when you need durability, cross-service delivery, or independent scaling.

### RabbitMQ

```text
producer → exchange → (bindings) → queue → consumer
```

| Exchange type | Routes by | Use |
|---|---|---|
| **direct** | exact routing-key match | targeted work queues |
| **fanout** | ignores the key — broadcasts | pub/sub to everyone |
| **topic** | pattern (`order.*.created`) | flexible subscription |
| headers | header values | rare |

- **Manual acknowledgment** (`autoAck: false`, ack *after* success) gives at-least-once: a crash
  before the ack means redelivery.
- **Durable queues + persistent messages** survive a broker restart. Both halves are required.
- **Dead-letter queue**: nack with `requeue: false` plus a dead-letter exchange, so a poison
  message is quarantined rather than looping forever.
- Scale with **competing consumers** (one queue, many consumers) and tune **prefetch (QoS)** — an
  unbounded prefetch lets one consumer hoard the queue.

### Azure Service Bus

**Queues** (point-to-point, one consumer per message) and **topics/subscriptions** (each
subscription gets a copy, filtered by rules). **Peek-lock** with explicit settlement —
**Complete / Abandon / Dead-letter** — and a built-in DLQ after `MaxDeliveryCount`.

**Sessions** give per-key FIFO ordering to a single consumer without serializing the whole queue.
Plus scheduled and deferred messages, duplicate detection, and transactions. Prefer **managed
identity** over connection-string keys ([20](20-Azure.md)).

### Kafka

Not a queue — a distributed, append-only **event log**.

- Producers append to **topics** split into **partitions**; consumers read by **offset**.
- Messages are **retained** and **replayable** — the defining difference from RabbitMQ and Service
  Bus, where a consumed message is gone.
- The **partition key** routes related events to the same partition, which is the *only* ordering
  guarantee you get. Partitions bound parallelism: **max useful consumers per group = partition
  count**.
- **Consumer groups** load-balance within a group (each message processed once) and fan out across
  groups (each group independently reads everything).
- **Commit offsets after processing** for at-least-once. Auto-commit risks both loss and
  duplication.

### MassTransit

An abstraction over brokers: strongly-typed messages, `IConsumer<T>`, automatic routing,
serialization, retry and dead-lettering, and broker portability by configuration.

- **Send** = a **command** to one consumer ("do X"). **Publish** = an **event** to all subscribers
  ("X happened"). Getting this distinction right is most of good message design.
- Resilience is declarative: **retry** (immediate, then backoff) → **redelivery** (delayed) →
  **error queue**.
- **Sagas** (state machines) coordinate multi-service workflows with **compensating actions** —
  distributed consistency without two-phase commit.
- Its **transactional outbox** integrates with EF Core to make the database write and the publish
  atomic.

### The two patterns that always come up

**Idempotency.** Delivery is at-least-once, so assume duplicates. Deduplicate on a message id
(an inbox table with a unique constraint), rely on natural idempotency ("set status = Shipped"),
or use conditional writes.

**The outbox.** You cannot atomically commit a database transaction *and* publish to a broker —
that's the **dual-write problem**. Write the message to an outbox table **in the same
transaction** as the business change; a relay process publishes it afterwards and marks it sent.
The message goes out **if and only if** the business change committed. Pair with an **inbox** on
the consumer side for effectively-once semantics.

---

## Comparison tables

| | `Channel<T>` | RabbitMQ | Azure Service Bus | Kafka |
|---|---|---|---|---|
| Scope | in-process | cross-service | cross-service, managed | cross-service, streaming |
| Durable | ❌ | ✅ | ✅ | ✅ |
| Message after consumption | gone | gone | gone | **retained, replayable** |
| Routing | n/a | **exchanges: direct/fanout/topic** | queues + topic filters | topic + partition key |
| Ordering | FIFO | per-queue | **sessions** (per key) | **partitions** (per key) |
| Best for | in-app decoupling | task queues, rich routing | enterprise Azure messaging | high-throughput streams, replay, event sourcing |

| | Send (command) | Publish (event) |
|---|---|---|
| Means | "do this" | "this happened" |
| Consumers | exactly one | zero or more |
| Naming | imperative — `ShipOrder` | past tense — `OrderShipped` |
| Can be rejected | yes | no — it already happened |

| Delivery guarantee | Reality |
|---|---|
| At-most-once | fast, loses messages — rarely acceptable |
| **At-least-once** | **what brokers give you** → duplicates → require idempotency |
| Exactly-once | not achievable end-to-end; approximated by at-least-once + dedupe |

---

## 🪤 Traps & gotchas

- **Forgetting `Writer.Complete()`** — `ReadAllAsync` waits forever and your app won't shut down.
- **Unbounded channels** — a producer faster than the consumer grows the queue until the process
  dies. Bounded with `Wait` is the default you want.
- **Assuming a message is processed once** — it isn't. Every broker is at-least-once, so a
  non-idempotent handler ("balance += amount") will eventually double-charge someone.
- **Dual write** — `SaveChanges()` then `Publish()`. A crash in between leaves the database
  changed and no message sent (or, with the order reversed, a message about something that rolled
  back). Outbox.
- **`autoAck: true`** — RabbitMQ considers the message delivered the moment it's sent, so a crash
  mid-processing loses it silently.
- **Durable queue but non-persistent messages** (or vice versa) — you need **both** for a message
  to survive a broker restart.
- **Poison-message loop** — a message that always fails, nacked with `requeue: true`, spins
  forever consuming the whole consumer. Dead-letter after N attempts.
- **A dead-letter queue nobody monitors** is just a slower way to lose messages. Alert on DLQ
  depth.
- **Unbounded prefetch** — one consumer grabs the entire queue, so your other consumers idle and a
  crash redelivers everything it held.
- **Assuming global ordering** — no broker gives it at scale. Kafka orders within a partition,
  Service Bus within a session. Design for per-key ordering or don't depend on order.
- **More consumers than Kafka partitions** — the extras sit idle. Partition count is your
  parallelism ceiling and is awkward to raise later.
- **Kafka auto-commit** — offsets commit on a timer regardless of whether processing succeeded, so
  you get both loss and duplication depending on timing. Commit after processing.
- **Changing a Kafka partition key** re-routes related events to different partitions and silently
  breaks your ordering guarantee.
- **Publishing a command / sending an event** — publishing "ShipOrder" to three subscribers ships
  it three times; sending "OrderShipped" to one consumer means nobody else finds out.
- **Fat messages carrying entities** — brittle across versions and expensive. Send ids and the
  minimum data; the consumer re-fetches.
- **Breaking a message contract** — consumers deployed at a different time than producers, so
  every change needs to be additive and readers need to be tolerant
  ([22](22-BestPractices-Architecture.md)).
- **No correlation id on messages** — a distributed trace that stops at the broker is a debugging
  dead end ([12](12-Observability.md)).
- **Using Kafka as a task queue** (or RabbitMQ as an event store) — both work badly. Kafka has no
  per-message ack or redelivery; RabbitMQ doesn't retain or replay.

---

## ❓ Likely questions

**Q: What is `Channel<T>` and when do you use a bounded one?**
A: An async in-process producer/consumer queue. Use bounded whenever the producer can outpace the
consumer: with `FullMode.Wait` the producer awaits when the channel is full, which propagates
back-pressure instead of growing memory until the process dies.

**Q: When do you move from `Channel<T>` to a broker?**
A: When you need durability (items must survive a restart), cross-service delivery, or independent
scaling of the consumer. An in-process channel is a single process's memory — perfectly good for
"return 202 and do this shortly", useless if losing the work is unacceptable.

**Q: How does RabbitMQ route messages?**
A: Producers publish to an **exchange**, never directly to a queue. Bindings route from the
exchange to queues: direct matches the routing key exactly, fanout broadcasts to every bound
queue, topic matches patterns like `order.*.created`. Consumers read from queues.

**Q: How do you get at-least-once delivery in RabbitMQ?**
A: Manual acknowledgment — `autoAck: false`, and ack only after processing succeeds. If the
consumer crashes first, the broker redelivers. Combine with durable queues and persistent messages
so a broker restart doesn't lose them.

**Q: What's the difference between Kafka and RabbitMQ?**
A: Kafka is an append-only log — messages are retained for a configured period and can be
**replayed**, and multiple consumer groups each read the whole stream independently. RabbitMQ is a
queue — a consumed message is gone, and its strength is flexible per-message routing. Kafka for
high-throughput streaming, event sourcing and replay; RabbitMQ for task queues and workflows.

**Q: How does ordering work in Kafka?**
A: Only within a partition. The partition key determines which partition an event lands in, so
events sharing a key (say, an order id) are ordered relative to each other. There is no global
ordering across partitions, by design — that's what makes it scale.

**Q: What limits Kafka consumer parallelism?**
A: The partition count. Within a consumer group each partition is assigned to exactly one
consumer, so consumers beyond the partition count sit idle.

**Q: Send vs Publish?**
A: Send delivers a **command** to exactly one consumer — imperative, and the consumer may reject
it. Publish broadcasts an **event** to every subscriber — past tense, a statement of fact that
can't be rejected. Getting this wrong either duplicates work or silently drops notifications.

**Q: Why must consumers be idempotent?**
A: Because every broker delivers at-least-once. A network blip on the ack, a redelivery after a
crash, or a producer retry all cause the same message to be processed twice. Deduplicate on a
message id, or make the operation naturally idempotent.

**Q: What is the outbox pattern and what problem does it solve?**
A: The dual-write problem — you can't atomically commit to the database and publish to a broker.
The outbox writes the message into a table in the *same* transaction as the business change; a
separate relay publishes it and marks it sent. So the message is published if and only if the
change committed. An inbox table on the consumer side gives you deduplication.

**Q: What is a saga?**
A: A multi-step workflow across services coordinated by a state machine, using **compensating
actions** to undo earlier steps when a later one fails — the alternative to a distributed
transaction. Orchestration (a central coordinator) is easier to reason about than choreography
for complex flows.

**Q: How do you handle a message that always fails?**
A: Retry with backoff for transient faults, then dead-letter it after a bounded number of
attempts. Never requeue indefinitely — a poison message will otherwise consume your consumer
forever. Then monitor and alert on the dead-letter queue, because a DLQ nobody watches is just
slow data loss.

**Q: Azure Service Bus sessions — what problem do they solve?**
A: Per-key FIFO ordering. All messages with the same session id go to one consumer in order, while
different sessions still process in parallel — so you get ordering where it matters without
serializing the entire queue.

**Q: Why use MassTransit rather than the raw client?**
A: You get typed messages and consumers, routing, serialization, retry/redelivery/error-queue
policies, the transactional outbox, and sagas — all of which you'd otherwise hand-roll — plus
broker portability by configuration.

---

## 🎓 Senior Extra

- **The outbox needs a relay strategy**: polling a table is simple and fine at moderate scale;
  change-data-capture (Debezium) scales further without polling. Either way, the relay must be
  idempotent, because it can crash after publishing but before marking sent.
- **Inbox + outbox = effectively-once**, which is the honest version of "exactly-once". True
  exactly-once end-to-end would require distributed transactions across your database and the
  broker, which is why nobody does it.
- **Message versioning**: additive changes only, tolerant readers, and never reuse a field name
  with new semantics. Producers and consumers deploy independently, so both old and new versions
  are in flight simultaneously — plan for it ([22](22-BestPractices-Architecture.md)).
- **Claim-check pattern** for large payloads: put the blob in storage, put the reference in the
  message. Brokers are not file transfer.
- **Propagate the trace context** (`traceparent`) through message headers, or your distributed
  trace ends at the publish and the consumer's work looks unrelated ([12](12-Observability.md)).
- **Consumer concurrency vs downstream capacity**: scaling consumers to drain a backlog faster
  just moves the bottleneck onto the database. Bound concurrency deliberately, and treat the queue
  depth as the signal for whether you actually have capacity.
- **Queue depth and consumer lag are your two key metrics** — depth rising means you're behind,
  and Kafka's consumer lag tells you by exactly how much.
- **Kafka compaction** retains the latest value per key indefinitely, which turns a topic into a
  durable materialized state — the mechanism behind event-sourced projections and the reason
  "replay to rebuild" works.
- **Dead-letter replay needs to be a designed feature**, not an ad-hoc script: fix the bug, then
  replay the DLQ through the same idempotent handler. That only works if the handler really is
  idempotent.
- **Prefer choreography for simple flows, orchestration for complex ones.** Choreography (each
  service reacts to events) has no central point of failure but no central view either — at four
  or five steps, nobody can answer "where did this order get stuck?"

→ Deeper: [`../07-Messaging/`](../07-Messaging/README.md)
