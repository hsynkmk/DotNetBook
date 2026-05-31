# MassTransit

## A messaging abstraction over brokers

MassTransit is a .NET messaging framework that sits **above** the raw broker clients (RabbitMQ, Azure Service Bus, Amazon SQS, Kafka), giving you a consistent, higher-level programming model: strongly-typed messages, consumers, automatic retries, dead-lettering, sagas, and the outbox pattern — without hand-coding exchanges, acks, and routing. You write consumers; MassTransit handles the broker plumbing.

```csharp
builder.Services.AddMassTransit(x => {
    x.AddConsumer<OrderPlacedConsumer>();
    x.UsingRabbitMq((ctx, cfg) => {              // swap to UsingAzureServiceBus / UsingAmazonSqs easily
        cfg.Host("localhost", "/", h => { h.Username("guest"); h.Password("guest"); });
        cfg.ConfigureEndpoints(ctx);             // auto-configures queues/consumers
    });
});

// A message (a plain record)
public record OrderPlaced(Guid OrderId, decimal Total);

// A consumer
public class OrderPlacedConsumer(IEmailService email) : IConsumer<OrderPlaced> {
    public async Task Consume(ConsumeContext<OrderPlaced> ctx) {
        await email.SendConfirmationAsync(ctx.Message.OrderId);   // ctx.Message is the typed message
    }
}

// Publishing
await publishEndpoint.Publish(new OrderPlaced(orderId, total));
```

You define messages as records, consumers as `IConsumer<T>`, and MassTransit maps them to broker queues/topics, handles serialization, acks, retries, and routing.

---

## Why use it over the raw client

The raw broker client ([03-RabbitMQ.md](03-RabbitMQ.md)) makes you hand-code exchanges, bindings, acks, serialization, retries, and dead-lettering. MassTransit provides these as first-class features:

| Concern | Raw client | MassTransit |
|---|---|---|
| Message routing | manual exchanges/bindings | automatic by message type |
| Serialization | manual (`byte[]`) | built-in (JSON by default) |
| Retry | hand-coded | configurable policies |
| Dead-lettering | manual DLX | automatic `_error` queues |
| Broker portability | locked to one client | swap brokers via config |
| Sagas / orchestration | DIY | built-in state machines |
| Outbox | DIY | built-in (EF integration) |

The trade-off: an abstraction layer (and its conventions/learning curve) vs. doing it yourself. For non-trivial messaging, MassTransit's reliability features (retry, DLQ, outbox, sagas) are worth far more than the raw control — most teams should use it (or a similar library like NServiceBus) rather than the raw client.

---

## Send vs Publish

MassTransit distinguishes two messaging styles:

```csharp
// Send — a COMMAND to a specific consumer (one recipient): "do this"
await sendEndpoint.Send(new ProcessPayment(orderId, amount));

// Publish — an EVENT to all subscribers (fan-out): "this happened"
await publishEndpoint.Publish(new OrderPlaced(orderId, total));
```

- **Send (command)** — directed to one endpoint/consumer; expresses intent (`ProcessPayment`). Use for "do X."
- **Publish (event)** — broadcast to all interested consumers; expresses a fact (`OrderPlaced`). Use for "X happened" — multiple services can react independently.

This command/event distinction (imperative vs. notification) shapes your message design: commands have one logical handler; events have zero-to-many. It's the messaging analog of method calls vs. domain events.

---

## Retry, redelivery, and error handling

MassTransit makes resilience declarative:

```csharp
cfg.ReceiveEndpoint("orders", e => {
    e.UseMessageRetry(r => r.Exponential(5, TimeSpan.FromSeconds(1), TimeSpan.FromMinutes(1), TimeSpan.FromSeconds(2)));
    e.UseScheduledRedelivery(r => r.Intervals(TimeSpan.FromMinutes(5), TimeSpan.FromMinutes(15)));
    e.ConfigureConsumer<OrderPlacedConsumer>(ctx);
});
```

- **Retry** — in-process immediate retries (exponential backoff) for transient failures.
- **Redelivery** — schedule the message for a later attempt (minutes/hours later) for failures that need time to resolve.
- **Error queue** — after retries are exhausted, the message moves to an `_error` queue automatically (MassTransit's dead-lettering) for inspection/replay.

This layered strategy (immediate retry → delayed redelivery → error queue) handles transient vs. persistent failures without poison-message loops — all configured, not hand-coded.

---

## Sagas (orchestrating multi-step workflows)

A **saga** is a long-running, stateful workflow coordinated across multiple messages/services — e.g., an order process: payment → inventory → shipping, each a step that may succeed or fail. MassTransit models sagas as **state machines**:

```csharp
public class OrderSaga : MassTransitStateMachine<OrderState> {
    public State AwaitingPayment { get; private set; } = null!;
    public State AwaitingShipment { get; private set; } = null!;

    public OrderSaga() {
        Initially(When(OrderPlaced).TransitionTo(AwaitingPayment).Then(ctx => /* request payment */));
        During(AwaitingPayment,
            When(PaymentReceived).TransitionTo(AwaitingShipment).Then(ctx => /* request shipment */),
            When(PaymentFailed).Finalize());                    // compensate / cancel
        During(AwaitingShipment, When(OrderShipped).Finalize());
    }
}
```

Sagas implement distributed-transaction-like consistency **without** a distributed transaction (which you should avoid — [Ch05 §08](../05-EFCore/08-Transactions.md)): each step is a local action + a message, and the saga tracks state and triggers **compensating actions** on failure (eventual consistency). This is the right pattern for multi-service workflows where 2PC isn't viable. (Saga concepts also in [07-Patterns.md](07-Patterns.md).)

---

## The transactional outbox

MassTransit integrates the **outbox pattern** with EF Core, solving the dual-write problem (you can't atomically write to a DB *and* publish to a broker — [Ch05 §08](../05-EFCore/08-Transactions.md)):

```csharp
x.AddEntityFrameworkOutbox<AppDbContext>(o => {
    o.UsePostgres();
    o.UseBusOutbox();                 // messages published during a transaction go to an outbox table
});
```

With the outbox, publishing a message inside a `SaveChanges` transaction writes the message to an **outbox table in the same DB transaction** as your business change. A background process then reliably publishes it to the broker. So the DB write and the message publish are effectively atomic (both happen or neither) — no lost messages, no phantom messages. This is the standard solution to "save to DB and publish an event reliably." (Outbox in depth: [07-Patterns.md](07-Patterns.md).)

---

## Broker portability

Because MassTransit abstracts the broker, switching is mostly a configuration change:

```csharp
x.UsingRabbitMq(...)        // → x.UsingAzureServiceBus(...) → x.UsingAmazonSqs(...)
```

Your consumers and messages stay the same; the transport changes. This lets you develop against RabbitMQ locally and deploy to Azure Service Bus, or migrate brokers, without rewriting messaging logic. (The transports aren't 100% identical — some features are broker-specific — but the core model ports cleanly.)

---

## Common gotchas

### Confusing Send and Publish

`Send` (command, one consumer) vs `Publish` (event, all subscribers). Publishing a command or sending an event misroutes intent. Model commands and events distinctly.

### Non-idempotent consumers with at-least-once delivery

Brokers deliver **at-least-once** — the same message can arrive twice (redelivery after a transient failure). Consumers must be **idempotent** (processing a message twice has the same effect as once — [07-Patterns.md](07-Patterns.md)). Non-idempotent consumers double-charge, double-send, etc.

### Skipping the outbox for DB + publish

Publishing a message after `SaveChanges` (not via the outbox) risks losing the message (if publish fails after commit) or publishing a phantom (if commit fails after publish). Use the **outbox** for atomic DB-write-and-publish.

### Sagas without compensation

A saga that doesn't handle failure (compensating actions for partial completion) leaves inconsistent state. Design the failure paths (cancel/refund/restock), not just the happy path.

### Hand-rolling what MassTransit provides

Reimplementing retry/DLQ/outbox/saga logic on the raw client is error-prone. Use the framework's built-in features.

---

## Summary

- **MassTransit** is a messaging abstraction over brokers (RabbitMQ/Azure Service Bus/SQS/Kafka): strongly-typed messages, `IConsumer<T>` consumers, automatic routing/serialization/retry/dead-lettering, and **broker portability** via config.
- Distinguish **Send** (a command to one consumer — "do X") from **Publish** (an event to all subscribers — "X happened").
- Resilience is declarative: **retry** (immediate, backoff) → **redelivery** (delayed) → **error queue** (after exhaustion) — handling transient vs. persistent failures without poison loops.
- **Sagas** (state machines) coordinate multi-service workflows with compensating actions — distributed consistency without distributed transactions; the **transactional outbox** (EF integration) makes DB-write + message-publish atomic.
- Brokers deliver **at-least-once** → make consumers **idempotent** ([07](07-Patterns.md)); prefer MassTransit's built-in features over hand-rolling on the raw client.

→ Next: [05-AzureServiceBus.md](05-AzureServiceBus.md)
