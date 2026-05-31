# BackgroundService

## The base class for long-running loops

`BackgroundService` is an abstract base class implementing `IHostedService` for the common case: a component that runs a **continuous loop** for the app's lifetime. You override one method — `ExecuteAsync(stoppingToken)` — and the base class wires up the start/stop lifecycle correctly (avoiding the "long work in `StartAsync`" trap from [01-HostedServices.md](01-HostedServices.md)).

```csharp
public class PollingWorker(ILogger<PollingWorker> log) : BackgroundService {
    protected override async Task ExecuteAsync(CancellationToken stoppingToken) {
        while (!stoppingToken.IsCancellationRequested) {
            log.LogInformation("Polling at {Time}", DateTimeOffset.UtcNow);
            await DoWorkAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);   // honor the token!
        }
    }
}

builder.Services.AddHostedService<PollingWorker>();
```

`ExecuteAsync` is called **after** the host starts (not blocking startup, unlike a raw `StartAsync` loop). The `stoppingToken` is signaled on shutdown — your job is to honor it.

---

## How `BackgroundService` works

The base class implements `IHostedService` so you don't have to:

```
StartAsync()  → calls ExecuteAsync(stoppingToken) but does NOT await it
                (returns once ExecuteAsync hits its first await → startup isn't blocked)
ExecuteAsync  → your long-running loop, running independently
StopAsync()   → signals stoppingToken (cancellation) and awaits ExecuteAsync to finish
                (within the shutdown timeout)
```

The key behaviors:
- `StartAsync` starts `ExecuteAsync` but **returns at the first `await`** inside it — so a long loop doesn't block host startup (the bug you'd hit writing the loop in a raw `StartAsync`).
- On shutdown, `StopAsync` **cancels** `stoppingToken` and **awaits** `ExecuteAsync` to complete (graceful drain, bounded by the shutdown timeout).

This is why `BackgroundService` is the right base for loops — it handles the start-without-blocking and stop-with-cancellation correctly.

---

## Honor the stopping token — everywhere

The `stoppingToken` is your shutdown signal. **Pass it to every async call and check it in loops**, so the service stops promptly:

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken) {
    while (!stoppingToken.IsCancellationRequested) {
        await ProcessBatchAsync(stoppingToken);                  // forward the token
        await Task.Delay(_interval, stoppingToken);              // cancellable delay (not Thread.Sleep!)
    }
}
```

A service that ignores `stoppingToken` (e.g., uses `Thread.Sleep`, or doesn't forward the token to I/O) won't stop until its current iteration finishes — and if that's slow or blocking, the host force-terminates it after the shutdown timeout, dropping in-flight work. Always thread the token through. `await Task.Delay(..., stoppingToken)` (not `Thread.Sleep`) makes the wait cancellable.

---

## Exception behavior (the critical detail)

What happens when `ExecuteAsync` throws an unhandled exception? **By default (.NET 6+), it stops the host** — an unhandled exception in a background service crashes the whole app. This is configurable:

```csharp
builder.Services.Configure<HostOptions>(o =>
    o.BackgroundServiceExceptionBehavior = BackgroundServiceExceptionBehavior.Ignore);
//   Ignore = log and keep the host running (the service stops, but the app doesn't crash)
//   StopHost (default) = an unhandled exception stops the entire host
```

So you have two choices:
1. **Let it crash the host** (default) — appropriate when the service is essential (if it can't run, the app shouldn't either) — combined with an orchestrator restart, this is fail-fast + self-heal.
2. **Catch exceptions inside the loop** so one iteration's failure doesn't kill the service:

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken) {
    while (!stoppingToken.IsCancellationRequested) {
        try { await ProcessAsync(stoppingToken); }
        catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested) { break; }  // shutdown
        catch (Exception ex) { log.LogError(ex, "Iteration failed; continuing"); }                  // isolate
        await Task.Delay(_interval, stoppingToken);
    }
}
```

This per-iteration try/catch is the usual choice for resilient workers — a transient failure (DB blip, bad message) logs and the loop continues, rather than crashing the app. Catch `OperationCanceledException` from the stopping token separately and break (it's normal shutdown, not an error).

---

## Scoped services — create a scope per unit of work

`BackgroundService` is a **singleton** ([01-HostedServices.md](01-HostedServices.md)), so it can't inject scoped services (`DbContext`). Create a scope per work item / iteration:

```csharp
public class CleanupWorker(IServiceProvider services) : BackgroundService {
    protected override async Task ExecuteAsync(CancellationToken ct) {
        using var timer = new PeriodicTimer(TimeSpan.FromHours(1));
        while (await timer.WaitForNextTickAsync(ct)) {
            await using var scope = services.CreateAsyncScope();             // fresh scope per cycle
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>(); // scoped, correct
            await db.Orders.Where(o => o.Expired).ExecuteDeleteAsync(ct);
        }
    }
}
```

Each iteration gets a fresh scope (and a fresh `DbContext`) — like a request scope. Forgetting this and injecting a scoped service is a captive-dependency bug ([Ch03 §03](../03-HostingAndDI/03-Lifetimes.md)); reusing one `DbContext` across iterations leaks/corrupts. (Detailed in [04-QueuedBackgroundWork.md](04-QueuedBackgroundWork.md).)

---

## Common loop shapes

```csharp
// 1. Periodic — PeriodicTimer (no overlap, no drift; the modern choice — see 05-ScheduledTasks)
using var timer = new PeriodicTimer(_interval);
while (await timer.WaitForNextTickAsync(stoppingToken)) await DoWorkAsync(stoppingToken);

// 2. Queue consumer — drain a channel (see 04-QueuedBackgroundWork)
await foreach (var item in _queue.DequeueAllAsync(stoppingToken)) await ProcessAsync(item, stoppingToken);

// 3. Continuous poll with delay
while (!stoppingToken.IsCancellationRequested) { await PollAsync(stoppingToken); await Task.Delay(_interval, stoppingToken); }
```

The most common shapes: periodic work (`PeriodicTimer`), queue consumption (channel — [04](04-QueuedBackgroundWork.md)), and polling. For scheduling, prefer `PeriodicTimer` over a `while + Task.Delay` (it avoids overlap and drift — [05-ScheduledTasks.md](05-ScheduledTasks.md)).

---

## Common gotchas

### Ignoring the stopping token

Not forwarding `stoppingToken` (or using `Thread.Sleep`) means the service won't shut down promptly → forced termination, dropped work. Pass the token to every async call; use `Task.Delay(..., token)`.

### Unhandled exception crashing the host

By default, an exception in `ExecuteAsync` stops the host. For resilient workers, wrap each iteration in try/catch (logging) so transient failures don't crash the app.

### Injecting scoped services

`BackgroundService` is a singleton — create a scope per iteration/work item for `DbContext` etc.

### `Thread.Sleep` instead of `await Task.Delay`

`Thread.Sleep` blocks a thread (and isn't cancellable). Use `await Task.Delay(interval, stoppingToken)` — non-blocking and cancellable.

### Treating `OperationCanceledException` as an error

On shutdown, the stopping token cancels awaited operations, throwing `OperationCanceledException`. Catch it separately and break/return — it's normal shutdown, not a failure to log as an error.

### Overlapping iterations

A `while + Task.Delay` loop where the work takes longer than the delay can drift or overlap. Use `PeriodicTimer` (you control the loop, no overlap) for periodic work.

---

## Summary

- **`BackgroundService`** is the base class for long-running loops: override **`ExecuteAsync(stoppingToken)`**; the base correctly starts it after host startup (without blocking) and cancels + awaits it on shutdown.
- **Honor the `stoppingToken`** — forward it to every async call, use `await Task.Delay(..., token)` (not `Thread.Sleep`) — so the service stops promptly within the shutdown timeout.
- **Exception behavior**: an unhandled exception in `ExecuteAsync` **stops the host** by default — for resilient workers, wrap each iteration in try/catch (log + continue), catching shutdown's `OperationCanceledException` separately.
- It's a **singleton** — create a **scope per iteration/work item** for scoped services (`DbContext`).
- Common shapes: periodic (`PeriodicTimer`), queue consumer (channel), polling — prefer `PeriodicTimer` over `while + Task.Delay` for scheduling.

→ Next: [03-WorkerServices.md](03-WorkerServices.md)
