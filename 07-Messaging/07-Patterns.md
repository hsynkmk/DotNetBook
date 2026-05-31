# Messaging Patterns

## The patterns that make messaging reliable

Brokers give you durable delivery, but **reliable** messaging requires patterns on top: the **outbox** (atomic DB-write + publish), **idempotent consumers** (handle duplicates), **ordered processing** (when order matters), and **sagas** (multi-step workflows). These solve the hard distributed-systems problems that naive "publish a message" code gets wrong.

---

## At-least-once delivery → design for duplicates

The foundational fact: **brokers deliver at-least-once.** A message can be delivered **more than once** (a consumer processes it, crashes before acking, and the broker redelivers; or a network blip causes a retry). Exactly-once delivery is generally impractical across systems. So:

> **Assume every message may arrive more than once. Design consumers to handle duplicates.**

This drives the most important pattern — idempotency — and is why naive consumers (charge a card, send an email, increment a counter) cause double-charges, duplicate emails, and wrong counts in production.

---

## Idempotent consumers

A consumer is **idempotent** if processing the same message twice has the **same effect as processing it once**. Strategies:

```csharp
// 1. Dedup by message id — record processed ids; skip duplicates
public async Task Consume(ConsumeContext<OrderPlaced> ctx) {
    var msgId = ctx.MessageId!.Value;
    if (await _processedStore.ExistsAsync(msgId)) return;          // already handled → skip
    await ProcessAsync(ctx.Message);
    await _processedStore.MarkProcessedAsync(msgId);               // record (ideally in the same tx as the work)
}

// 2. Natural idempotency — design the operation to be repeatable
await db.Orders.Where(o => o.Id == id).ExecuteUpdateAsync(s => s.SetProperty(o => o.Status, "Shipped"));
//   setting status to "Shipped" twice = same result (idempotent by nature)

// 3. Upsert / conditional — INSERT ... ON CONFLICT DO NOTHING, or check-then-act in a transaction
```

Approaches: **dedup by message id** (track processed ids, skip repeats — best done atomically with the work, e.g., via the inbox pattern), **natural idempotency** (design operations that are safe to repeat — "set to X" not "add X"), or **conditional writes** (upserts, optimistic concurrency). Idempotency is non-negotiable for at-least-once messaging — every consumer needs it.

---

## The outbox pattern — atomic DB write + publish

The **dual-write problem**: you can't atomically (1) write to your database and (2) publish a message to a broker — they're separate systems with no shared transaction. Naive code fails:

```csharp
// ✗ — two systems, not atomic:
await db.SaveChangesAsync(ct);                 // committed
await bus.Publish(new OrderPlaced(...));        // if THIS fails → DB has the order but no event published (lost)
//   (or reverse the order → publish succeeds, DB commit fails → phantom event for an order that doesn't exist)
```

The **outbox pattern** fixes it: write the message to an **outbox table in the same database transaction** as the business change. A separate process then publishes outbox rows to the broker and marks them sent.

```
1. In ONE DB transaction: save the Order AND insert an "OrderPlaced" row into the Outbox table → commit (atomic)
2. A background relay reads unsent Outbox rows → publishes to the broker → marks them sent
   (if publishing fails, retry later — the row is still there; at-least-once publish)
```

```csharp
// With MassTransit's EF outbox, this is automatic ([04-MassTransit.md]):
x.AddEntityFrameworkOutbox<AppDbContext>(o => { o.UsePostgres(); o.UseBusOutbox(); });
// Publish inside a SaveChanges transaction → message goes to the outbox table atomically, relayed later
```

The outbox guarantees the DB change and the message publish are **effectively atomic** (both or neither) — no lost messages, no phantom messages. It's the standard solution to "save to the DB and reliably publish an event," and the reason you **don't** need distributed transactions ([Ch05 §08](../05-EFCore/08-Transactions.md)). (The receiving side can use an **inbox** — dedup incoming message ids in the same transaction as processing — for idempotency.)

---

## Ordered processing

Most brokers don't guarantee global order across competing consumers. When order matters (apply events to an aggregate in sequence), use the broker's ordering mechanism:

- **Kafka** — same **partition key** → same partition → ordered ([06-Kafka.md](06-Kafka.md)).
- **Azure Service Bus** — **sessions** (`SessionId`) → FIFO to one consumer ([05-AzureServiceBus.md](05-AzureServiceBus.md)).
- **RabbitMQ** — a single consumer per queue, or consistent-hash routing.

Scope ordering to the **key** that needs it (per-order, per-customer) rather than serializing everything — global ordering kills parallelism. And design so order matters less where possible (idempotent, commutative operations are easier than strictly-ordered ones).

---

## Sagas — long-running, multi-step workflows

A **saga** coordinates a workflow spanning multiple services/messages where a single transaction is impossible (each service has its own database). Example: place order → charge payment → reserve inventory → arrange shipping. Each step is a local transaction + a message; the saga tracks state and triggers **compensating actions** if a step fails:

```
OrderPlaced → ChargePayment → (success) → ReserveInventory → (success) → ArrangeShipping → done
                            → (FAIL) → CancelOrder (compensate)
                                              ReserveInventory → (FAIL) → RefundPayment + CancelOrder (compensate)
```

Sagas provide **eventual consistency** without distributed transactions: instead of rolling back a 2PC, each failure triggers compensating actions that undo prior steps (refund the charge, release the inventory). MassTransit models sagas as state machines ([04-MassTransit.md](04-MassTransit.md)). Two styles: **orchestration** (a central saga directs each step) vs **choreography** (services react to each other's events with no central coordinator) — orchestration is easier to reason about and debug for complex flows.

---

## Dead-letter handling

Messages that fail repeatedly must go somewhere — not loop forever (poison messages) and not be silently dropped:

- After **retries** (immediate + delayed backoff) are exhausted, route the message to a **dead-letter queue** (built into Service Bus and RabbitMQ via DLX; MassTransit's `_error` queues).
- **Monitor** dead-letter queues (a growing DLQ is an alert) and provide a way to **inspect and replay** messages after fixing the cause.

This separates transient failures (retried) from permanent ones (dead-lettered for human/automated intervention) — covered per-broker in [03](03-RabbitMQ.md)/[05](05-AzureServiceBus.md) and via MassTransit in [04](04-MassTransit.md).

---

## Common gotchas

### Non-idempotent consumers

The #1 messaging bug — at-least-once delivery means duplicates, and a non-idempotent consumer double-charges/double-sends. Every consumer must be idempotent (dedup ids, natural idempotency, or conditional writes).

### Dual-write without the outbox

Saving to the DB then publishing (or vice versa) risks lost or phantom messages when one of the two fails. Use the **outbox pattern** for atomic DB-write + publish.

### Distributed transactions for cross-service consistency

2PC across DB + broker (or multiple services) is fragile and poorly supported ([Ch05 §08](../05-EFCore/08-Transactions.md)). Use **outbox + sagas + idempotency** for eventual consistency instead.

### Assuming ordering

Competing consumers don't preserve order. Use partition keys (Kafka) or sessions (Service Bus) for per-key ordering; don't assume global order.

### Sagas without compensation

Designing only the happy path leaves inconsistent state on failure. Define compensating actions for each step (refund, release, cancel).

### Ignoring dead-letter queues

Unmonitored DLQs hide failures (messages silently pile up). Monitor and alert on DLQ depth; provide replay.

---

## Summary

- Brokers deliver **at-least-once** → **assume duplicates** and make every consumer **idempotent** (dedup by message id, natural idempotency, or conditional writes) — the non-negotiable pattern.
- The **outbox pattern** solves the dual-write problem: write the message to an outbox table **in the same DB transaction** as the business change, then relay it to the broker — atomic, no lost/phantom messages, no distributed transaction needed (MassTransit automates it; pair with an **inbox** for receive-side dedup).
- For **ordering**, use the broker's mechanism scoped to a key (Kafka **partitions**, Service Bus **sessions**) — don't assume global order or serialize everything.
- **Sagas** coordinate multi-service workflows with **compensating actions** (eventual consistency, no 2PC); prefer orchestration for complex flows.
- Handle failures with **retry → dead-letter** (monitor and replay DLQs); don't loop poison messages or drop them silently.

→ Next: [Questions.md](Questions.md)
