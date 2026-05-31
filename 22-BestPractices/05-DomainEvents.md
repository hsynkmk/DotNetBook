# Domain Events

## Decoupling "what happened" from "what reacts"

A **domain event** records that **something meaningful happened in the domain** — "an order was placed," "a payment failed," "a user registered." Instead of an aggregate ([02-DomainPersistence.md](02-DomainPersistence.md)) directly calling all the things that should react (send an email, update inventory, notify shipping), it **raises an event**, and separate **handlers** react. This decouples the core business action from its side effects: the `Order` aggregate doesn't need to know about email or inventory — it just declares "I was placed," and interested handlers respond. The result is cohesive aggregates, single-responsibility handlers, and an extensible system (add a reaction without touching the aggregate).

```csharp
public class Order {
    private readonly List<IDomainEvent> _events = new();
    public IReadOnlyList<IDomainEvent> DomainEvents => _events;

    public static Order Place(int customerId, IEnumerable<LineDto> lines) {
        var order = new Order(customerId, lines);
        order._events.Add(new OrderPlaced(order.Id, customerId, order.Total));   // record what happened
        return order;
    }
}
public record OrderPlaced(int OrderId, int CustomerId, decimal Total) : IDomainEvent;
```

---

## Raising and dispatching

The aggregate **records** events as part of its behavior (it doesn't dispatch them itself — that would re-couple it). The events are **dispatched after the changes are persisted**, typically by the infrastructure (e.g., overriding `SaveChanges` to collect and publish events from tracked entities — [Ch05 §11](../05-EFCore/11-Interceptors.md)):

```csharp
// In the DbContext / a SaveChanges interceptor: after saving, dispatch collected events
public override async Task<int> SaveChangesAsync(CancellationToken ct = default) {
    var result = await base.SaveChangesAsync(ct);
    var events = ChangeTracker.Entries<IHasDomainEvents>()
        .SelectMany(e => e.Entity.DomainEvents).ToList();
    foreach (var e in events) await _dispatcher.DispatchAsync(e, ct);   // handlers react
    return result;
}
```

Handlers (one per concern) react:

```csharp
public class SendConfirmationOnOrderPlaced : IDomainEventHandler<OrderPlaced> {
    public Task Handle(OrderPlaced e, CancellationToken ct) => _email.SendConfirmationAsync(e.OrderId, ct);
}
public class ReserveInventoryOnOrderPlaced : IDomainEventHandler<OrderPlaced> { /* ... */ }
```

Each handler does **one** thing; adding a new reaction means adding a handler, not editing `Order` (Open/Closed — [CSharpBook Ch17 SOLID](../../CSharpBook/17-BestPractices/README.md)).

---

## Domain events vs integration events

A critical distinction:

| | Domain event | Integration event |
|---|---|---|
| Scope | **inside** one service/process | **across** services/boundaries |
| Delivery | in-process (same transaction context) | via a message broker ([Ch07](../07-Messaging/README.md)) |
| Consistency | with the aggregate's transaction | eventual, network |
| Example | `OrderPlaced` → reserve inventory (same app) | `OrderPlaced` → notify the shipping *service* |

- **Domain events** are **in-process** — handled within the same service, often within or right after the same transaction, to coordinate side effects in *this* application.
- **Integration events** cross **service boundaries** — published to a broker (RabbitMQ/Service Bus/Kafka — [Ch07](../07-Messaging/README.md), [Ch20 §04/§08](../20-AzureIntegration/04-ServiceBus.md)) for *other* services to consume, with eventual consistency.

Don't conflate them: a domain event might *cause* an integration event to be published (often via the outbox — below), but they operate at different scopes with different delivery/consistency guarantees.

---

## The timing problem and the outbox pattern

A subtle hazard: if you dispatch events that have **external side effects** (send an email, publish an integration event) *before* the transaction commits, and the commit then fails, you've fired side effects for something that didn't happen. Conversely, if you publish an integration event *after* commit but the broker call fails, you've committed locally but lost the event — an **inconsistency** between your database and downstream services (the **dual-write problem**).

The **outbox pattern** solves this for integration events ([Ch07 §07](../07-Messaging/07-Patterns.md)):

```
1. In the SAME transaction as the business change, write the integration event to an "outbox" table.
   (DB change + outbox row commit atomically — no dual-write.)
2. A separate process reads the outbox and publishes to the broker, marking rows sent.
   (At-least-once delivery; consumers are idempotent — [Ch07 §07].)
```

This guarantees that an event is published **if and only if** the business change was committed — atomically tying the local state change to the eventual message. It's the standard way to bridge a domain change to a reliable integration event without the dual-write race. (Pure in-process domain-event handlers that only touch the same database can run in the same transaction and don't need the outbox; it's the **external** publish that does.)

---

## Common gotchas

### Aggregate dispatching its own events (re-coupling)

If the aggregate calls handlers directly, it's coupled to them again — defeating the point. The aggregate **records** events; infrastructure **dispatches** them (after save).

### Side effects before commit

Dispatching events with external effects before the transaction commits means a failed commit leaves side effects for an event that "didn't happen." Dispatch after a successful save (and use the outbox for cross-service publishes).

### Dual-write (DB then broker) without an outbox

Committing the DB then publishing to a broker can lose the message if the publish fails — DB and downstream diverge. Use the **outbox** ([Ch07 §07](../07-Messaging/07-Patterns.md)) to publish atomically with the DB change.

### Conflating domain and integration events

In-process domain events ≠ cross-service integration events (different scope/delivery/consistency). Keep them distinct; a domain event may trigger an integration event (via outbox), but they're not the same thing.

### Non-idempotent integration handlers

Outbox/broker delivery is at-least-once — consumers can see an event more than once. Make integration-event handlers **idempotent** ([Ch07 §07](../07-Messaging/07-Patterns.md)).

---

## Summary

- **Domain events** record that *something meaningful happened* so an aggregate ([02-DomainPersistence.md](02-DomainPersistence.md)) doesn't directly call its side effects — the aggregate **records** events; **handlers** (one per concern, single-responsibility) **react**, making the system decoupled and extensible (add a reaction without editing the aggregate).
- The aggregate doesn't dispatch its own events; **infrastructure dispatches them after persistence** (e.g., a `SaveChanges` interceptor collecting events from tracked entities — [Ch05 §11](../05-EFCore/11-Interceptors.md)).
- Distinguish **domain events** (in-process, this service, often same transaction) from **integration events** (cross-service, via a **broker** — [Ch07](../07-Messaging/README.md), eventual consistency); a domain event may *cause* an integration event.
- For cross-service publishing, use the **outbox pattern** — write the event to an outbox table **in the same transaction** as the business change, then a separate process publishes it — solving the **dual-write** problem (publish iff committed); keep integration handlers **idempotent**.

→ Next: [06-VerticalSlices.md](06-VerticalSlices.md)
