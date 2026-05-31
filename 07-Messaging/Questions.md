# Chapter 07 — Messaging — Q & A

---

### Q1. What is `Channel<T>` and when do you use it?

The BCL's async, in-process producer-consumer queue. Use it to decouple work within one app (a handler enqueues and returns; a background consumer processes). It's the in-process step before reaching for a broker — no durability or cross-process delivery.

---

### Q2. Bounded vs unbounded channel?

**Bounded** has a capacity cap; when full, `WriteAsync` awaits (async back-pressure) so a fast producer can't exhaust memory — the safe default. **Unbounded** never blocks the producer but risks unbounded memory growth if the producer outpaces the consumer.

---

### Q3. Why `Channel<T>` over `BlockingCollection<T>`?

`Channel<T>` is async (`await WriteAsync`/`ReadAsync`, no thread blocked); `BlockingCollection<T>` blocks threads. In async server code, blocking risks thread-pool starvation, so use `Channel<T>`.

---

### Q4. What's the scope-per-item rule in a background queue worker?

A `BackgroundService` is a singleton, but work items often need scoped services (`DbContext`). Create a DI scope per item (`CreateAsyncScope`) and resolve scoped services from it — injecting them into the singleton worker is a captive dependency, and sharing one `DbContext` across items corrupts it.

---

### Q5. Why wrap each work item in try/catch?

An unhandled exception in `BackgroundService.ExecuteAsync` stops the worker by default. Wrapping each item isolates failures (log/retry/dead-letter) so one bad item doesn't tear down the whole consumer.

---

### Q6. When do you graduate from an in-process queue to a broker?

When you need **durability** (don't lose queued work on restart/crash), **cross-service** delivery (consumer is a different app), **independent scaling** of consumers, or built-in **delivery guarantees/retries/dead-lettering**. In-memory `Channel<T>` provides none of these.

---

### Q7. Explain RabbitMQ's exchange/queue/binding model.

Producers publish to an **exchange** (not directly to a queue); **bindings** route messages from the exchange to **queues** (often by routing key); consumers read from queues. This indirection enables flexible routing (one message to many queues, filtered by key) without the producer knowing consumers.

---

### Q8. RabbitMQ exchange types?

**Direct** (exact routing-key match → specific queue), **fanout** (ignores key, broadcasts to all bound queues — pub/sub), **topic** (routing-key pattern like `order.*.created` — flexible filtered routing), **headers** (route on message headers).

---

### Q9. Why use manual acknowledgment in RabbitMQ?

With `autoAck: true`, a message is removed on delivery — a crash mid-processing loses it. **Manual ack** (`autoAck: false`, ack only after successful processing) means an unacked message is redelivered on crash → at-least-once delivery. The basis of reliability.

---

### Q10. How do you handle poison messages (always-failing)?

Nack with `requeue: false` (don't loop forever) and route to a **dead-letter queue** (via a dead-letter exchange), optionally after retry-with-backoff. Inspect/replay dead-lettered messages after fixing the cause — don't requeue infinitely or drop silently.

---

### Q11. What does MassTransit add over a raw broker client?

A higher-level model: typed messages, `IConsumer<T>`, automatic routing/serialization, configurable retry, automatic dead-lettering (`_error` queues), sagas, the outbox pattern, and broker portability (swap RabbitMQ/Service Bus/SQS via config). Far less plumbing than the raw client.

---

### Q12. Send vs Publish in MassTransit?

**Send** delivers a **command** to one specific consumer ("do X" — one logical handler). **Publish** broadcasts an **event** to all subscribers ("X happened" — zero-to-many reactors). Match commands to Send, events to Publish.

---

### Q13. What is a saga and what problem does it solve?

A long-running, stateful workflow coordinating multiple messages/services (e.g., payment → inventory → shipping) where a single transaction is impossible. It tracks state and triggers **compensating actions** on failure — giving eventual consistency without distributed transactions. MassTransit models them as state machines.

---

### Q14. What is the dual-write problem and how does the outbox solve it?

You can't atomically write to your DB and publish to a broker (separate systems). The **outbox pattern** writes the message to an outbox table **in the same DB transaction** as the business change; a relay then publishes it. So both happen or neither — no lost or phantom messages, no distributed transaction.

---

### Q15. Azure Service Bus queues vs topics/subscriptions?

A **queue** is point-to-point — each message consumed by one receiver (competing consumers). A **topic** with **subscriptions** is pub-sub — each subscription gets its own copy of every message (filtered by rules). Queue for work distribution; topic for events multiple services react to.

---

### Q16. What is peek-lock in Service Bus?

A received message is **locked** (invisible to others) while you process it, then settled: **Complete** (success → remove), **Abandon** (release lock → redeliver), or **Dead-letter** (poison → DLQ). Lock expiry or abandon causes redelivery → at-least-once. After `MaxDeliveryCount`, it auto-dead-letters.

---

### Q17. What are Service Bus sessions for?

Per-key FIFO ordering: messages sharing a `SessionId` are delivered in order to a single consumer (holding the session lock). Use them when related messages (per-order, per-customer) must be processed sequentially — without serializing the entire queue.

---

### Q18. How is Kafka fundamentally different from a queue?

Kafka is a retained, append-only **log** split into partitions; consumers read by **offset** and messages are **not deleted on consumption** — they're retained, so consumers can **replay** and multiple consumer groups read independently. Traditional queues delete messages once consumed.

---

### Q19. How does Kafka achieve ordering and parallelism?

The message **key** determines the partition (same key → same partition → ordered within it). Ordering is guaranteed **per-partition**, not globally. Different keys spread across partitions for parallelism; max consumers per group = partition count.

---

### Q20. Kafka consumer groups — load-balance vs fan-out?

Within **one** consumer group, partitions are distributed across consumers (each message processed once — scaling). **Different** consumer groups each read the whole stream independently (fan-out / pub-sub). So one topic serves both work distribution and event broadcasting.

---

### Q21. When choose Kafka vs RabbitMQ/Service Bus?

**Kafka**: high-throughput event streaming, event sourcing/replay, multiple independent consumers of the same stream. **RabbitMQ/Service Bus**: task queues, request/response, complex per-message routing, scheduled/priority messages — simpler with richer per-message features. Kafka trades those for throughput, retention, and replay.

---

### Q22. Why must consumers be idempotent?

Brokers deliver **at-least-once** — the same message can arrive more than once (redelivery after a crash/blip). A non-idempotent consumer double-charges, double-sends, etc. Make processing the same message twice equivalent to once (dedup by id, natural idempotency, or conditional writes).

---

### Q23. Three ways to make a consumer idempotent?

(1) **Dedup by message id** — track processed ids and skip duplicates (ideally atomically with the work, via an inbox). (2) **Natural idempotency** — design repeatable operations ("set status to Shipped," not "increment"). (3) **Conditional writes** — upserts (`ON CONFLICT DO NOTHING`), optimistic concurrency.

---

### Q24. Why avoid distributed transactions for cross-service consistency?

2PC across a DB and a broker (or multiple services) is complex, slow, and poorly supported in .NET. Use **outbox + sagas + idempotency** for eventual consistency instead — keep each transaction within one database/resource.

---

### Q25. Orchestration vs choreography sagas?

**Orchestration** — a central saga directs each step (easier to reason about, debug, and monitor for complex flows). **Choreography** — services react to each other's events with no central coordinator (looser coupling but harder to trace). Prefer orchestration for complex multi-step workflows.

---

→ Next: [Coding.md](Coding.md)
