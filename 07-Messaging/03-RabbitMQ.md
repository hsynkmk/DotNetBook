# RabbitMQ

## A message broker for cross-service messaging

RabbitMQ is a popular, mature **message broker** — an out-of-process server that durably holds messages between producers and consumers. Unlike an in-process `Channel<T>`, it persists messages (survives restarts), delivers across services/machines, and provides acknowledgments, routing, and retries. It implements the AMQP protocol; the .NET client is `RabbitMQ.Client`.

```csharp
// (RabbitMQ.Client — modern async API)
var factory = new ConnectionFactory { HostName = "localhost" };
await using var connection = await factory.CreateConnectionAsync();
await using var channel = await connection.CreateChannelAsync();

await channel.QueueDeclareAsync(queue: "orders", durable: true, exclusive: false, autoDelete: false);

// Publish
var body = JsonSerializer.SerializeToUtf8Bytes(order);
await channel.BasicPublishAsync(exchange: "", routingKey: "orders", body: body);
```

> RabbitMQ is often used through a higher-level abstraction (**MassTransit** — [04-MassTransit.md](04-MassTransit.md)) rather than the raw client. This file covers the **concepts** (exchanges, queues, bindings, acks) you need regardless of which library you use.

---

## The core model: exchanges, queues, bindings

RabbitMQ's routing model has three pieces — understanding it is the key to using RabbitMQ:

```
Producer → publishes to an EXCHANGE → (routed via BINDINGS) → QUEUE(s) → Consumer
```

- **Exchange** — receives published messages and routes them. Producers publish to an *exchange*, not directly to a queue.
- **Queue** — holds messages until a consumer processes them. Consumers read from *queues*.
- **Binding** — a rule linking an exchange to a queue (often with a routing key/pattern) that determines which messages go where.

This indirection (publish to exchange → bindings route to queues) is what enables flexible routing (one message to many queues, or filtered by routing key) without the producer knowing the consumers.

---

## Exchange types

The exchange type determines routing behavior:

| Type | Routes by | Use |
|---|---|---|
| **Direct** | exact routing-key match | route to a specific queue by key |
| **Fanout** | ignores key — to **all** bound queues | broadcast / pub-sub (every consumer gets it) |
| **Topic** | routing-key pattern (`order.*.created`) | flexible, filtered routing |
| **Headers** | message headers | routing on attributes (rarer) |

```csharp
// Topic exchange — consumers subscribe to patterns
await channel.ExchangeDeclareAsync("events", ExchangeType.Topic, durable: true);
await channel.QueueBindAsync(queue: "order-emails", exchange: "events", routingKey: "order.*.created");
await channel.BasicPublishAsync(exchange: "events", routingKey: "order.us.created", body: body);
//   → routed to "order-emails" (matches order.*.created)
```

- **Direct** for point-to-point (work queues).
- **Fanout** for pub-sub (an event every consumer should see).
- **Topic** for filtered routing (the most flexible — subscribe to `order.*`, `*.created`, etc.).

---

## Acknowledgments — at-least-once delivery

The core reliability mechanism: a consumer **acknowledges** a message after successfully processing it. Until acked, RabbitMQ considers the message unprocessed and will **redeliver** it if the consumer crashes — giving **at-least-once** delivery:

```csharp
var consumer = new AsyncEventingBasicConsumer(channel);
consumer.ReceivedAsync += async (sender, ea) => {
    try {
        var order = JsonSerializer.Deserialize<Order>(ea.Body.Span);
        await ProcessAsync(order);
        await channel.BasicAckAsync(ea.DeliveryTag, multiple: false);   // success → ack (remove from queue)
    } catch (Exception) {
        await channel.BasicNackAsync(ea.DeliveryTag, multiple: false, requeue: false);  // fail → nack
        // requeue: false → don't loop forever; route to a dead-letter queue instead
    }
};
await channel.BasicConsumeAsync("orders", autoAck: false, consumer);   // manual ack (autoAck: false!)
```

Critical: use **manual ack** (`autoAck: false`) and ack **only after successful processing**. With `autoAck: true`, a message is removed the moment it's delivered — so a crash mid-processing **loses** it. Manual ack + redelivery is what makes RabbitMQ reliable. On failure, `BasicNack` with `requeue: false` to avoid infinite redelivery loops — route failures to a **dead-letter queue** (below).

---

## Durability & persistence

For messages to survive a broker restart, **both** the queue and the messages must be durable:

```csharp
await channel.QueueDeclareAsync("orders", durable: true, ...);   // durable QUEUE (survives restart)

var props = new BasicProperties { Persistent = true };           // persistent MESSAGES (written to disk)
await channel.BasicPublishAsync("", "orders", mandatory: false, props, body);
```

A durable queue + persistent messages = the queue and its messages survive a RabbitMQ restart. (Non-durable/transient are faster but lost on restart.) Durability has a throughput cost (disk writes) — use it for messages you can't afford to lose.

---

## Dead-letter queues & retries

Messages that repeatedly fail shouldn't loop forever or be silently dropped. Configure a **dead-letter exchange (DLX)** so nack'd/expired messages route to a dead-letter queue for inspection/reprocessing:

```csharp
await channel.QueueDeclareAsync("orders", durable: true, exclusive: false, autoDelete: false,
    arguments: new Dictionary<string, object?> {
        ["x-dead-letter-exchange"] = "orders-dlx",     // failed messages go here
        ["x-message-ttl"] = 86_400_000                  // optional TTL
    });
```

Failed messages (nack with `requeue: false`, or TTL-expired) land in the dead-letter queue, where you can inspect why they failed and replay them after a fix. Combined with **retry-with-backoff** (re-publish with a delay), this is how you handle transient vs permanent failures without poison-message loops. (Higher-level libraries automate retry/DLQ — [04-MassTransit.md](04-MassTransit.md).)

---

## Competing consumers (scaling) & prefetch

Multiple consumer instances reading the **same queue** compete — each message goes to **one** consumer, distributing load (scale by adding consumers). Control how many unacked messages a consumer holds with **prefetch (QoS)**:

```csharp
await channel.BasicQosAsync(prefetchSize: 0, prefetchCount: 10, global: false);
//   each consumer takes up to 10 unacked messages at a time → fair distribution, bounded memory
```

A low prefetch spreads work fairly across consumers (a slow consumer doesn't hog messages); a high prefetch improves throughput per consumer. Tune it. This **competing-consumers** pattern (one queue, many consumers) is the standard way to scale message processing horizontally.

---

## Common gotchas

### `autoAck: true` loses messages on crash

Auto-ack removes a message on delivery, before processing. A crash mid-processing loses it. Use **manual ack** (`autoAck: false`) and ack only after success — the basis of at-least-once delivery.

### Infinite redelivery (poison messages)

Nack with `requeue: true` on a message that always fails loops forever. Use `requeue: false` + a **dead-letter queue** (and retry-with-backoff) so poison messages are quarantined, not looped.

### Forgetting durability

A non-durable queue or non-persistent messages vanish on broker restart. Set `durable: true` + `Persistent = true` for messages you can't lose (at a throughput cost).

### Publishing directly to a queue conceptually

You publish to an **exchange** (with a routing key); bindings route to queues. (The default `""` exchange routes by queue name, which looks direct-to-queue but is still via the default exchange.) Understand exchanges for non-trivial routing.

### No prefetch limit

Without QoS, a consumer may grab the whole queue (uneven distribution, high memory). Set a sensible `prefetchCount` for fair, bounded consumption.

### Reinventing retry/saga logic on the raw client

Hand-rolling retries, dead-lettering, and orchestration on `RabbitMQ.Client` is error-prone. Consider **MassTransit** ([04](04-MassTransit.md)) which provides these out of the box.

---

## Summary

- **RabbitMQ** is a durable, out-of-process **message broker** for cross-service messaging — persisting messages, delivering at-least-once with acks, and routing flexibly (vs in-process `Channel<T>`).
- Model: producers publish to an **exchange** → **bindings** route to **queues** → consumers read. Exchange types: **direct** (key match), **fanout** (broadcast/pub-sub), **topic** (pattern routing), headers.
- **Manual acknowledgment** (`autoAck: false`, ack after success) gives at-least-once delivery (crash → redeliver); **durable queues + persistent messages** survive restarts.
- Quarantine failures with a **dead-letter queue** (nack `requeue: false` + DLX) and retry-with-backoff — avoid poison-message loops; scale with **competing consumers** (one queue, many consumers) + **prefetch (QoS)**.
- For non-trivial apps, use a higher-level library (**MassTransit** — [04](04-MassTransit.md)) over the raw client.

→ Next: [04-MassTransit.md](04-MassTransit.md)
