# Chapter 07 — Messaging — Coding Problems

Build an in-process channel queue, a background consumer, then port to a broker — adding idempotency, the outbox, and dead-lettering.

---

## Problem 1: A bounded channel queue

Build a generic in-process queue with back-pressure and completion.

<details><summary>Solution</summary>

```csharp
public sealed class WorkQueue<T> {
    private readonly Channel<T> _channel = Channel.CreateBounded<T>(
        new BoundedChannelOptions(1000) { FullMode = BoundedChannelFullMode.Wait });   // back-pressure

    public ValueTask EnqueueAsync(T item, CancellationToken ct) => _channel.Writer.WriteAsync(item, ct);
    public IAsyncEnumerable<T> DequeueAllAsync(CancellationToken ct) => _channel.Reader.ReadAllAsync(ct);
    public void Complete() => _channel.Writer.Complete();   // signal end-of-input → consumers stop
}
```

Bounded → producer awaits when full (no unbounded memory). `Complete()` lets consumer `ReadAllAsync` loops finish. ([01-ChannelOfT.md](01-ChannelOfT.md).)

</details>

---

## Problem 2: Background consumer with scope-per-item

Consume the queue in a `BackgroundService`, using a scoped service per item, isolating failures.

<details><summary>Solution</summary>

```csharp
public class EmailWorker(WorkQueue<EmailJob> queue, IServiceProvider services, ILogger<EmailWorker> log)
    : BackgroundService {
    protected override async Task ExecuteAsync(CancellationToken ct) {
        await foreach (var job in queue.DequeueAllAsync(ct)) {
            await using var scope = services.CreateAsyncScope();        // fresh scope per item
            try {
                var sender = scope.ServiceProvider.GetRequiredService<IEmailSender>();  // scoped
                await sender.SendAsync(job.To, job.Body, ct);
            }
            catch (OperationCanceledException) when (ct.IsCancellationRequested) { throw; }  // shutdown
            catch (Exception ex) { log.LogError(ex, "Email job {To} failed", job.To); }      // isolate failure
        }
    }
}
```

Scope per item (so each can use scoped services), per-item try/catch (one failure ≠ kill the worker), honors the stopping token. ([02-BackgroundQueues.md](02-BackgroundQueues.md).)

</details>

---

## Problem 3: Producer endpoint

Wire a Minimal API endpoint that enqueues work and returns 202.

<details><summary>Solution</summary>

```csharp
builder.Services.AddSingleton<WorkQueue<EmailJob>>();
builder.Services.AddHostedService<EmailWorker>();

app.MapPost("/emails", async (SendEmailRequest req, WorkQueue<EmailJob> queue, CancellationToken ct) => {
    await queue.EnqueueAsync(new EmailJob(req.To, req.Body), ct);   // awaits if queue full (back-pressure)
    return Results.Accepted();                                       // 202 — work happens in the background
});
```

The request returns immediately; work is decoupled to the background consumer. Bounded queue means the endpoint awaits under overload rather than exhausting memory. ([02-BackgroundQueues.md](02-BackgroundQueues.md).)

</details>

---

## Problem 4: Port to RabbitMQ — durable publish

Replace the in-process queue with RabbitMQ for durability across restarts.

<details><summary>Solution</summary>

```csharp
// Publish (producer)
var factory = new ConnectionFactory { HostName = "localhost" };
await using var conn = await factory.CreateConnectionAsync();
await using var channel = await conn.CreateChannelAsync();
await channel.QueueDeclareAsync("emails", durable: true, exclusive: false, autoDelete: false);

var props = new BasicProperties { Persistent = true, MessageId = Guid.NewGuid().ToString() };
await channel.BasicPublishAsync("", "emails", mandatory: false, props,
    JsonSerializer.SerializeToUtf8Bytes(job));
```

Durable queue + persistent messages survive a broker restart (unlike the in-memory channel). A `MessageId` enables idempotency on the consumer side (Problem 6). ([03-RabbitMQ.md](03-RabbitMQ.md).)

</details>

---

## Problem 5: RabbitMQ consumer with manual ack + dead-letter

Consume reliably: ack on success, dead-letter on permanent failure.

<details><summary>Solution</summary>

```csharp
await channel.QueueDeclareAsync("emails", durable: true, exclusive: false, autoDelete: false,
    arguments: new Dictionary<string, object?> { ["x-dead-letter-exchange"] = "emails-dlx" });

await channel.BasicQosAsync(0, prefetchCount: 10, global: false);   // fair, bounded consumption

var consumer = new AsyncEventingBasicConsumer(channel);
consumer.ReceivedAsync += async (_, ea) => {
    try {
        var job = JsonSerializer.Deserialize<EmailJob>(ea.Body.Span)!;
        await ProcessAsync(job);
        await channel.BasicAckAsync(ea.DeliveryTag, multiple: false);          // success → ack
    } catch (TransientException) {
        await channel.BasicNackAsync(ea.DeliveryTag, multiple: false, requeue: true);   // retry
    } catch (Exception) {
        await channel.BasicNackAsync(ea.DeliveryTag, multiple: false, requeue: false);  // → dead-letter
    }
};
await channel.BasicConsumeAsync("emails", autoAck: false, consumer);   // MANUAL ack
```

Manual ack = at-least-once (crash → redeliver); transient failures requeue, permanent ones dead-letter (no poison loop); prefetch bounds in-flight messages. ([03-RabbitMQ.md](03-RabbitMQ.md).)

</details>

---

## Problem 6: Idempotent consumer

The same message may be delivered twice. Make processing idempotent.

<details><summary>Solution</summary>

```csharp
public class IdempotentEmailConsumer(IProcessedStore processed, IEmailSender sender) {
    public async Task ConsumeAsync(string messageId, EmailJob job, CancellationToken ct) {
        // Dedup by message id — skip if already processed
        if (await processed.ExistsAsync(messageId, ct)) return;        // duplicate → no-op

        await sender.SendAsync(job.To, job.Body, ct);
        await processed.MarkProcessedAsync(messageId, ct);              // record (ideally in the same tx)
    }
}
```

At-least-once delivery means duplicates; dedup by `MessageId` ensures the email is sent once. For DB work, record the processed id **in the same transaction** as the work (the inbox pattern) so dedup and processing are atomic. Alternatively, design natural idempotency. ([07-Patterns.md](07-Patterns.md).)

</details>

---

## Problem 7: The outbox pattern (atomic DB write + publish)

Save an order and publish `OrderPlaced` atomically — no lost or phantom events.

<details><summary>Solution</summary>

```csharp
// Manual outbox: write the message to an outbox table in the SAME transaction as the order
public async Task PlaceOrderAsync(Order order, CancellationToken ct) {
    db.Orders.Add(order);
    db.OutboxMessages.Add(new OutboxMessage {
        Id = Guid.NewGuid(),
        Type = nameof(OrderPlaced),
        Payload = JsonSerializer.Serialize(new OrderPlaced(order.Id, order.Total)),
        OccurredAt = DateTimeOffset.UtcNow
    });
    await db.SaveChangesAsync(ct);   // order + outbox row commit ATOMICALLY (one transaction)
}

// A background relay publishes unsent outbox rows, then marks them sent
public class OutboxRelay(IServiceProvider services, IBus bus) : BackgroundService {
    protected override async Task ExecuteAsync(CancellationToken ct) {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(1));
        while (await timer.WaitForNextTickAsync(ct)) {
            await using var scope = services.CreateAsyncScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            var pending = await db.OutboxMessages.Where(m => m.SentAt == null).Take(50).ToListAsync(ct);
            foreach (var msg in pending) {
                await bus.PublishAsync(msg.Type, msg.Payload, ct);   // at-least-once publish
                msg.SentAt = DateTimeOffset.UtcNow;
            }
            await db.SaveChangesAsync(ct);
        }
    }
}
```

The order and the outbox row commit in one transaction (atomic — no dual-write problem); the relay publishes reliably (retries naturally — unsent rows stay until published). With MassTransit, `AddEntityFrameworkOutbox` does all this for you. ([07-Patterns.md](07-Patterns.md), [04-MassTransit.md](04-MassTransit.md).)

</details>

---

## Problem 8: MassTransit consumer with retry

Define a typed consumer with retry and let MassTransit handle plumbing.

<details><summary>Solution</summary>

```csharp
builder.Services.AddMassTransit(x => {
    x.AddConsumer<OrderPlacedConsumer>();
    x.UsingRabbitMq((ctx, cfg) => {
        cfg.Host("localhost", "/", h => { h.Username("guest"); h.Password("guest"); });
        cfg.ReceiveEndpoint("order-placed", e => {
            e.UseMessageRetry(r => r.Exponential(5, TimeSpan.FromSeconds(1), TimeSpan.FromMinutes(1), TimeSpan.FromSeconds(2)));
            e.ConfigureConsumer<OrderPlacedConsumer>(ctx);
        });
    });
});

public record OrderPlaced(Guid OrderId, decimal Total);

public class OrderPlacedConsumer(IEmailService email) : IConsumer<OrderPlaced> {
    public async Task Consume(ConsumeContext<OrderPlaced> ctx) {
        // idempotent: dedup by ctx.MessageId or use a natural-idempotent operation
        await email.SendConfirmationAsync(ctx.Message.OrderId);
    }
    // After retries exhaust, MassTransit moves the message to order-placed_error automatically
}

// Publish (an event → all subscribers)
await publishEndpoint.Publish(new OrderPlaced(orderId, total));
```

Typed message + consumer; exponential retry for transients; automatic `_error` dead-lettering after exhaustion — no manual ack/exchange/DLX code. Keep the consumer idempotent. ([04-MassTransit.md](04-MassTransit.md).)

</details>

---

## Problem 9: Choose the right messaging tech

For each, pick in-process channel, RabbitMQ/Service Bus, or Kafka — and justify.
1. Offload sending a welcome email after signup, within one monolith; losing a queued email on restart is acceptable.
2. An order service publishes `OrderPlaced`; payment, inventory, and analytics services each react independently and must not miss events.
3. Ingesting 500k IoT telemetry events/sec, with analytics that may replay history.
4. Per-customer events must be processed strictly in order.

<details><summary>Solution</summary>

1. **In-process `Channel<T>`** + background consumer — single app, loss-on-restart acceptable, no broker overhead needed. ([01](01-ChannelOfT.md)/[02](02-BackgroundQueues.md).)
2. **RabbitMQ/Service Bus pub-sub** (fanout/topic, or Service Bus topic + subscriptions) — durable, multiple independent consumers each get the event; use the **outbox** so the order save and `OrderPlaced` publish are atomic. Consumers idempotent. ([03](03-RabbitMQ.md)/[05](05-AzureServiceBus.md)/[07](07-Patterns.md).)
3. **Kafka** — built for high-throughput streaming + retained log for replay; analytics is a separate consumer group reading the same stream. ([06-Kafka.md](06-Kafka.md).)
4. **Kafka partition key = customerId** (per-partition order) or **Service Bus sessions** (`SessionId = customerId`) — per-key FIFO without serializing everything. ([05](05-AzureServiceBus.md)/[06](06-Kafka.md)/[07](07-Patterns.md).)

The principle: in-process for single-app non-durable offload; a broker for durable cross-service messaging (pub-sub, work queues); Kafka for high-throughput streaming/replay; partition keys or sessions for ordering.

</details>

---

You can now build in-process channel queues with background consumers, port to a durable broker with manual ack and dead-lettering, make consumers idempotent, implement the outbox for atomic DB-write-and-publish, use MassTransit's higher-level model, and choose the right messaging technology and ordering strategy per scenario.

→ Back to [Chapter 07 README](README.md) · Next chapter: [Chapter 08 — Background Processing](../08-BackgroundProcessing/README.md)
