# Background Queues

## A hosted service that drains a channel

The standard in-process job pattern: a **`Channel<T>`** ([01-ChannelOfT.md](01-ChannelOfT.md)) holds queued work, and a **`BackgroundService`** ([Ch03 §01](../03-HostingAndDI/01-GenericHost.md)) consumes it. Producers (API endpoints, other services) enqueue and return immediately; the background worker processes items independently. This decouples request latency from work duration without any external broker.

```csharp
// 1. The queue abstraction (registered as a singleton, shared by producers + consumer)
public interface IBackgroundQueue {
    ValueTask EnqueueAsync(Func<IServiceProvider, CancellationToken, Task> work, CancellationToken ct);
    IAsyncEnumerable<Func<IServiceProvider, CancellationToken, Task>> DequeueAllAsync(CancellationToken ct);
}

public class BackgroundQueue : IBackgroundQueue {
    private readonly Channel<Func<IServiceProvider, CancellationToken, Task>> _channel =
        Channel.CreateBounded<Func<IServiceProvider, CancellationToken, Task>>(
            new BoundedChannelOptions(1000) { FullMode = BoundedChannelFullMode.Wait });

    public ValueTask EnqueueAsync(Func<IServiceProvider, CancellationToken, Task> work, CancellationToken ct)
        => _channel.Writer.WriteAsync(work, ct);

    public IAsyncEnumerable<Func<IServiceProvider, CancellationToken, Task>> DequeueAllAsync(CancellationToken ct)
        => _channel.Reader.ReadAllAsync(ct);
}
```

```csharp
// 2. The consumer — a BackgroundService draining the queue
public class QueueWorker(IBackgroundQueue queue, IServiceProvider services, ILogger<QueueWorker> log)
    : BackgroundService {
    protected override async Task ExecuteAsync(CancellationToken stoppingToken) {
        await foreach (var work in queue.DequeueAllAsync(stoppingToken)) {
            // Each item gets its own DI SCOPE (to use scoped services like DbContext)
            await using var scope = services.CreateAsyncScope();
            try { await work(scope.ServiceProvider, stoppingToken); }
            catch (Exception ex) { log.LogError(ex, "Background work item failed"); }  // one failure ≠ kill the worker
        }
    }
}
```

```csharp
// 3. Registration + a producer
builder.Services.AddSingleton<IBackgroundQueue, BackgroundQueue>();
builder.Services.AddHostedService<QueueWorker>();

app.MapPost("/reports", async (ReportRequest req, IBackgroundQueue queue, CancellationToken ct) => {
    await queue.EnqueueAsync(async (sp, token) => {
        var generator = sp.GetRequiredService<IReportGenerator>();   // scoped, resolved per item
        await generator.GenerateAsync(req, token);
    }, ct);
    return Results.Accepted();    // return immediately; work runs in the background
});
```

This is the canonical pattern: a singleton queue, a `BackgroundService` consumer, and a **scope per work item** so each job can use scoped services (`DbContext`, etc.).

---

## The scope-per-item rule

A `BackgroundService` is a **singleton**, but work items often need **scoped** services (a `DbContext`, a repository, the current operation's services). You can't inject scoped services into the singleton worker (captive dependency — [Ch03 §03](../03-HostingAndDI/03-Lifetimes.md)). The fix, shown above: **create a DI scope per work item** and resolve scoped services from it:

```csharp
await using var scope = services.CreateAsyncScope();
var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();   // scoped, correct per item
// ... process ...
// scope disposed → db disposed, like end-of-request
```

Each work item gets a fresh scope (its own `DbContext`, etc.), exactly like a web request gets a scope. This is the single most important detail in background-queue code — forgetting it leads to sharing one `DbContext` across all items (thread-safety violations, stale data — [Ch05 §01](../05-EFCore/01-DbContext.md)).

---

## Error handling per item

Wrap each item's processing in try/catch so **one failing item doesn't tear down the worker** (an unhandled exception in `ExecuteAsync` stops the `BackgroundService` by default — [Ch03 §01](../03-HostingAndDI/01-GenericHost.md)):

```csharp
try { await work(scope.ServiceProvider, ct); }
catch (OperationCanceledException) when (ct.IsCancellationRequested) { throw; }  // shutdown — let it stop
catch (Exception ex) { log.LogError(ex, "Item failed"); /* optionally: dead-letter / retry */ }
```

For failed items you might log, retry (with backoff), or move them to a dead-letter store. The key: isolate per-item failures so the queue keeps draining.

---

## Graceful shutdown

On shutdown (SIGTERM/Ctrl+C), the `stoppingToken` is signaled. Decide your drain behavior:

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken) {
    await foreach (var work in queue.DequeueAllAsync(stoppingToken)) { ... }
    // When stoppingToken fires, ReadAllAsync stops yielding → loop exits → worker stops
}
```

`ReadAllAsync(stoppingToken)` stops yielding when cancellation is requested, so in-flight items finish (within the host's shutdown timeout — [Ch03 §01](../03-HostingAndDI/01-GenericHost.md)) and queued-but-unstarted items are dropped (in-memory). If you must not lose queued work on shutdown, use a **durable** queue (a broker — later in this chapter) instead of an in-memory channel.

---

## In-process queue vs broker — when to graduate

This in-process pattern (the same ground as [Ch08 Background Processing](../08-BackgroundProcessing/README.md)) is great for:
- Offloading slow work from the request path (send email, generate a report, call a slow API) **within one app**.
- Work where losing queued items on restart is acceptable.

Graduate to a **message broker** (RabbitMQ/Kafka/Service Bus — rest of this chapter) when you need:
- **Durability** — don't lose queued work on crash/restart.
- **Cross-service** delivery — the consumer is a different app.
- **Independent scaling** — scale consumers separately from producers.
- **Delivery guarantees / retries / dead-lettering** built in.

The in-memory background queue is the lightweight first step; the broker is the durable, distributed evolution.

---

## Common gotchas

### Sharing one `DbContext`/scoped service across items

The singleton worker can't hold scoped services. Create a **scope per item** and resolve scoped services from it — else you share one `DbContext` across all items (corruption).

### Unhandled exception killing the worker

An exception escaping `ExecuteAsync` stops the `BackgroundService`. Wrap each item in try/catch (logging/retrying) so one bad item doesn't stop processing.

### Unbounded queue

An unbounded channel + a producer outpacing the consumer grows memory without limit. Use a **bounded** channel for back-pressure (`EnqueueAsync` awaits when full).

### Losing queued work on restart

In-memory queues drop unstarted items on shutdown/crash. If that's unacceptable, use a durable broker.

### Not honoring the stopping token

Ignoring `stoppingToken` prevents graceful shutdown (forced termination, dropped in-flight work). Pass it through `DequeueAllAsync` and to the work.

---

## Summary

- The in-process background-job pattern: a singleton **`Channel<T>`-backed queue** + a **`BackgroundService`** consumer; producers `EnqueueAsync` and return immediately (e.g., 202), the worker drains asynchronously.
- **Create a DI scope per work item** (`CreateAsyncScope`) so each job can use scoped services (`DbContext`) — the singleton worker can't inject them directly (captive dependency).
- **Wrap each item in try/catch** so one failure doesn't stop the worker; log/retry/dead-letter failed items.
- Honor the **stopping token** for graceful shutdown (in-flight items finish within the shutdown timeout; queued items are dropped — in-memory).
- Use this for in-process offloading where losing queued items on restart is OK; **graduate to a broker** for durability, cross-service delivery, independent scaling, and built-in delivery guarantees. (See also [Ch08](../08-BackgroundProcessing/README.md).)

→ Next: [03-RabbitMQ.md](03-RabbitMQ.md)
