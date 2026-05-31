# Queued Background Work

## Offloading work from the request path

The most common background pattern: a request handler **enqueues** work and returns immediately; a `BackgroundService` **dequeues** and processes it. This decouples request latency from work duration — the user gets a fast 202 response, the slow work (email, report, external API call) happens off the request thread. In-process, this uses a **`Channel<T>`** ([Ch07 §01](../07-Messaging/01-ChannelOfT.md)); the consumer is a hosted service ([02-BackgroundService.md](02-BackgroundService.md)).

```
HTTP request → handler enqueues a work item → returns 202 (fast)
                         ↓
              [ bounded Channel<T> ]
                         ↓
       BackgroundService consumer → processes items (slow, off-request)
```

> This overlaps with [Ch07 §02 (Background Queues)](../07-Messaging/02-BackgroundQueues.md) — that file frames it from the messaging angle; here it's the canonical background-processing pattern with the **scope-per-item** rule front and center.

---

## The queue abstraction

Define a queue interface backed by a bounded channel, registered as a singleton (shared by producers and the consumer):

```csharp
public interface IBackgroundTaskQueue {
    ValueTask EnqueueAsync(Func<IServiceProvider, CancellationToken, ValueTask> workItem, CancellationToken ct);
    ValueTask<Func<IServiceProvider, CancellationToken, ValueTask>> DequeueAsync(CancellationToken ct);
}

public sealed class BackgroundTaskQueue : IBackgroundTaskQueue {
    private readonly Channel<Func<IServiceProvider, CancellationToken, ValueTask>> _queue =
        Channel.CreateBounded<Func<IServiceProvider, CancellationToken, ValueTask>>(
            new BoundedChannelOptions(capacity: 1000) { FullMode = BoundedChannelFullMode.Wait });

    public ValueTask EnqueueAsync(Func<IServiceProvider, CancellationToken, ValueTask> workItem, CancellationToken ct)
        => _queue.Writer.WriteAsync(workItem, ct);

    public ValueTask<Func<IServiceProvider, CancellationToken, ValueTask>> DequeueAsync(CancellationToken ct)
        => _queue.Reader.ReadAsync(ct);
}

builder.Services.AddSingleton<IBackgroundTaskQueue, BackgroundTaskQueue>();
```

The work item is a `Func<IServiceProvider, CancellationToken, ValueTask>` — it receives a **service provider** (so the consumer can pass it the per-item scope's provider) and the stopping token. The bounded channel gives back-pressure (producers await when full).

---

## The consumer with scope-per-item

```csharp
public class QueuedHostedService(IBackgroundTaskQueue queue, IServiceProvider services,
    ILogger<QueuedHostedService> log) : BackgroundService {
    protected override async Task ExecuteAsync(CancellationToken stoppingToken) {
        while (!stoppingToken.IsCancellationRequested) {
            var workItem = await queue.DequeueAsync(stoppingToken);

            await using var scope = services.CreateAsyncScope();        // ← scope PER work item
            try {
                await workItem(scope.ServiceProvider, stoppingToken);    // pass the scope's provider
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested) { break; }
            catch (Exception ex) { log.LogError(ex, "Background work item failed"); }   // isolate failure
        }
    }
}
builder.Services.AddHostedService<QueuedHostedService>();
```

**The scope-per-item rule is the heart of this pattern.** The consumer is a singleton, but work items routinely need scoped services (`DbContext`, repositories). Each item gets its own DI scope (`CreateAsyncScope`), and the work item resolves scoped services from `scope.ServiceProvider` — exactly like each web request gets a scope. Per-item try/catch isolates failures so one bad item doesn't kill the consumer ([02-BackgroundService.md](02-BackgroundService.md)).

---

## The producer

```csharp
app.MapPost("/reports", async (ReportRequest req, IBackgroundTaskQueue queue, CancellationToken ct) => {
    await queue.EnqueueAsync(async (sp, token) => {
        var generator = sp.GetRequiredService<IReportGenerator>();   // SCOPED, from the per-item scope
        var db = sp.GetRequiredService<AppDbContext>();               // SCOPED — fresh per item
        await generator.GenerateAndSaveAsync(req, db, token);
    }, ct);
    return Results.Accepted();   // 202 — return immediately; work runs in the background
});
```

The handler enqueues a closure capturing the request data and returns 202 (Accepted). The closure resolves its dependencies from the **scoped** provider the consumer passes in — so it gets a fresh `DbContext`/services per execution, never sharing the singleton consumer's (nonexistent) scoped services.

---

## Why a fresh scope per item (not per consumer)

The cardinal correctness rule, restated because it's the #1 bug source:

```csharp
// ✗ — injecting a scoped service into the singleton consumer = captive dependency;
//     the DbContext lives forever, is shared across all items → corruption, "second operation" errors
public class BadWorker(AppDbContext db) : BackgroundService { ... }   // WRONG

// ✓ — a scope (and thus a DbContext) per work item, like a request scope
await using var scope = services.CreateAsyncScope();
var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();    // fresh, isolated per item
```

A `DbContext` is scoped and not thread-safe ([Ch05 §01](../05-EFCore/01-DbContext.md)); sharing one across many background items is a serious bug. The scope-per-item pattern mirrors the per-request scope — each unit of work is isolated. (Same rule for any scoped service.)

---

## Back-pressure and overload

Use a **bounded** channel so a flood of enqueues doesn't exhaust memory:

```csharp
Channel.CreateBounded<T>(new BoundedChannelOptions(1000) { FullMode = BoundedChannelFullMode.Wait });
//   Wait → producer (the request handler) awaits when full → natural back-pressure (slows intake)
//   DropOldest/DropNewest → shed load if you can drop work under overload
```

`FullMode.Wait` applies back-pressure to producers (the endpoint awaits, slowing request acceptance under load — protecting the system). `Drop*` modes shed work when overloaded (acceptable if losing some items is OK). Choose based on whether every item must be processed. An **unbounded** queue risks memory exhaustion if producers outpace the consumer — avoid it.

---

## Parallel processing

To process multiple items concurrently, run several consumer loops (or one loop fanning out to bounded concurrency):

```csharp
protected override async Task ExecuteAsync(CancellationToken ct) {
    // N parallel consumers, each with its own scope per item
    await Parallel.ForEachAsync(
        ReadAllAsync(ct), new ParallelOptions { MaxDegreeOfParallelism = 4, CancellationToken = ct },
        async (workItem, token) => {
            await using var scope = services.CreateAsyncScope();   // scope per item, still
            await workItem(scope.ServiceProvider, token);
        });
}
```

Bound the parallelism (`MaxDegreeOfParallelism`) so you don't overwhelm downstream resources (DB connection pool, external APIs). Each concurrent item still gets its **own scope** (each its own `DbContext`) — never share a scope across parallel items.

---

## In-process vs durable queue

This in-process queue loses unprocessed items on restart/crash (the channel is in memory). For **durability**, use a broker ([Ch07](../07-Messaging/README.md)) or a durable job library ([06-Hangfire.md](06-Hangfire.md)/[07-QuartzNet.md](07-QuartzNet.md)):

- **In-process channel** — fast, simple; fine when losing queued items on restart is acceptable (e.g., best-effort notifications).
- **Durable broker/job store** — when you must not lose work (the job must run even across a crash/restart).

Match the durability to the work's importance.

---

## Common gotchas

### Sharing a scoped service across items

The #1 bug — injecting a scoped `DbContext` into the singleton consumer, or reusing one scope for all items. Create a **fresh scope per item**.

### Unbounded queue → OOM

An unbounded channel + producers outpacing the consumer exhausts memory. Use a bounded channel with `Wait` (back-pressure) or `Drop*` (shed load).

### Unhandled exception killing the consumer

An exception in `ExecuteAsync` stops the service. Wrap each item in try/catch (log/retry/dead-letter) so one bad item doesn't stop processing.

### Losing queued work on restart

In-memory queues drop unprocessed items on crash/restart. Use a durable broker/job store if the work can't be lost.

### Unbounded parallelism

Processing items with no concurrency limit can exhaust the DB connection pool or hammer downstreams. Bound `MaxDegreeOfParallelism`.

### Capturing request-scoped state in the closure incorrectly

The enqueued closure outlives the request; don't capture the request's `HttpContext`/scoped services directly — capture **plain data** and resolve scoped services from the per-item scope's provider.

---

## Summary

- The queued-work pattern: a request handler **enqueues** a work item (returns 202 fast); a **`BackgroundService`** dequeues from a bounded **`Channel<T>`** and processes it off the request path.
- **Create a fresh DI scope per work item** (`CreateAsyncScope`) and resolve scoped services (`DbContext`) from it — the singleton consumer can't inject them, and sharing one across items corrupts state. This is the cardinal rule.
- Capture **plain data** in the enqueued closure (not request-scoped objects); resolve scoped services from the per-item scope's provider.
- Use a **bounded** channel for back-pressure (`Wait`) or load-shedding (`Drop*`); bound parallelism (`MaxDegreeOfParallelism`) to protect downstreams; wrap each item in try/catch.
- In-memory queues lose items on restart — use a **broker** ([Ch07](../07-Messaging/README.md)) or **durable job store** ([06](06-Hangfire.md)/[07](07-QuartzNet.md)) when work must not be lost.

→ Next: [05-ScheduledTasks.md](05-ScheduledTasks.md)
