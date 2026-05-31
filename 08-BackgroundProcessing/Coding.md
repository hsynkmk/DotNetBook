# Chapter 08 — Background Processing — Coding Problems

Build hosted services and background workers correctly: scope-per-item, graceful shutdown, scheduling without overlap, run-once-across-instances, and idempotency.

---

## Problem 1: A startup task with `IHostedService`

Run a one-time startup task (warm a cache) that doesn't block the app from starting if it fails.

<details><summary>Solution</summary>

```csharp
public class CacheWarmer(IServiceProvider services, ILogger<CacheWarmer> log) : IHostedService {
    public async Task StartAsync(CancellationToken ct) {
        try {
            await using var scope = services.CreateAsyncScope();      // scope for scoped services
            var loader = scope.ServiceProvider.GetRequiredService<IReferenceDataLoader>();
            await loader.WarmAsync(ct);
            log.LogInformation("Cache warmed");
        } catch (Exception ex) {
            log.LogWarning(ex, "Cache warm failed; continuing");      // non-fatal → app still starts
        }
    }
    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}
builder.Services.AddHostedService<CacheWarmer>();
```

`StartAsync` does one-time work (kept reasonably fast — it blocks startup), creates a scope for scoped services, and swallows failure so a cache-warm error doesn't prevent the app from starting. ([01-HostedServices.md](01-HostedServices.md).)

</details>

---

## Problem 2: A resilient `BackgroundService` loop

Write a polling worker that survives per-iteration failures and shuts down gracefully.

<details><summary>Solution</summary>

```csharp
public class PollingWorker(IServiceProvider services, ILogger<PollingWorker> log) : BackgroundService {
    protected override async Task ExecuteAsync(CancellationToken stoppingToken) {
        while (!stoppingToken.IsCancellationRequested) {
            try {
                await using var scope = services.CreateAsyncScope();
                await scope.ServiceProvider.GetRequiredService<IPoller>().PollAsync(stoppingToken);
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested) { break; }  // shutdown
            catch (Exception ex) { log.LogError(ex, "Poll iteration failed; continuing"); }             // isolate
            await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);   // cancellable, non-blocking
        }
    }
}
```

Per-iteration try/catch (one failure doesn't crash the host), shutdown's `OperationCanceledException` caught separately, `Task.Delay` with the token (not `Thread.Sleep`). ([02-BackgroundService.md](02-BackgroundService.md).)

</details>

---

## Problem 3: Queued background work with scope-per-item

Build the queue + consumer + producer, with a fresh scope per item.

<details><summary>Solution</summary>

```csharp
// Queue (singleton, bounded)
public sealed class TaskQueue {
    private readonly Channel<Func<IServiceProvider, CancellationToken, ValueTask>> _ch =
        Channel.CreateBounded<Func<IServiceProvider, CancellationToken, ValueTask>>(
            new BoundedChannelOptions(1000) { FullMode = BoundedChannelFullMode.Wait });
    public ValueTask EnqueueAsync(Func<IServiceProvider, CancellationToken, ValueTask> w, CancellationToken ct) => _ch.Writer.WriteAsync(w, ct);
    public IAsyncEnumerable<Func<IServiceProvider, CancellationToken, ValueTask>> DequeueAllAsync(CancellationToken ct) => _ch.Reader.ReadAllAsync(ct);
}

// Consumer
public class QueueWorker(TaskQueue queue, IServiceProvider services, ILogger<QueueWorker> log) : BackgroundService {
    protected override async Task ExecuteAsync(CancellationToken ct) {
        await foreach (var work in queue.DequeueAllAsync(ct)) {
            await using var scope = services.CreateAsyncScope();        // ← scope PER item
            try { await work(scope.ServiceProvider, ct); }
            catch (Exception ex) when (!ct.IsCancellationRequested) { log.LogError(ex, "Work item failed"); }
        }
    }
}

// Producer
builder.Services.AddSingleton<TaskQueue>();
builder.Services.AddHostedService<QueueWorker>();
app.MapPost("/jobs", async (JobRequest req, TaskQueue queue, CancellationToken ct) => {
    await queue.EnqueueAsync(async (sp, token) => {
        var db = sp.GetRequiredService<AppDbContext>();   // SCOPED, fresh per item
        await sp.GetRequiredService<IJobProcessor>().RunAsync(req.Id, db, token);
    }, ct);
    return Results.Accepted();
});
```

Bounded queue (back-pressure), scope per item (each work item gets its own `DbContext`), 202 response. The closure captures `req.Id` (plain data), resolving scoped services from the per-item scope. ([04-QueuedBackgroundWork.md](04-QueuedBackgroundWork.md).)

</details>

---

## Problem 4: Periodic work without overlap

Run a cleanup every 5 minutes, guaranteeing no overlap even if cleanup takes longer.

<details><summary>Solution</summary>

```csharp
public class CleanupWorker(IServiceProvider services, ILogger<CleanupWorker> log) : BackgroundService {
    protected override async Task ExecuteAsync(CancellationToken ct) {
        using var timer = new PeriodicTimer(TimeSpan.FromMinutes(5));
        while (await timer.WaitForNextTickAsync(ct)) {       // next tick waits until current work is awaited
            await using var scope = services.CreateAsyncScope();
            try {
                var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
                await db.Sessions.Where(s => s.Expired).ExecuteDeleteAsync(ct);
            } catch (Exception ex) { log.LogError(ex, "Cleanup failed"); }
        }
    }
}
```

`PeriodicTimer` + awaiting the work between ticks guarantees no overlap (the next tick can't start until the current cleanup finishes) — unlike a callback `Timer`. Cancellable on shutdown. ([05-ScheduledTasks.md](05-ScheduledTasks.md).)

</details>

---

## Problem 5: Cron-scheduled nightly job

Run a job nightly at 2 AM UTC using a cron expression.

<details><summary>Solution</summary>

```csharp
using Cronos;

public class NightlyReport(IServiceProvider services, ILogger<NightlyReport> log) : BackgroundService {
    private readonly CronExpression _cron = CronExpression.Parse("0 2 * * *");   // 02:00 daily

    protected override async Task ExecuteAsync(CancellationToken ct) {
        while (!ct.IsCancellationRequested) {
            var next = _cron.GetNextOccurrence(DateTime.UtcNow, TimeZoneInfo.Utc);
            if (next is null) break;
            var delay = next.Value - DateTime.UtcNow;
            if (delay > TimeSpan.Zero) await Task.Delay(delay, ct);   // wait until 2 AM UTC

            await using var scope = services.CreateAsyncScope();
            try { await scope.ServiceProvider.GetRequiredService<IReportJob>().RunAsync(ct); }
            catch (Exception ex) { log.LogError(ex, "Nightly report failed"); }
        }
    }
}
```

Cron computes the next 2 AM occurrence; delay until then; run; repeat. UTC avoids DST issues. **Caveat**: this runs in every instance — see Problem 6 for run-once-across-the-cluster. ([05-ScheduledTasks.md](05-ScheduledTasks.md).)

</details>

---

## Problem 6: Run a scheduled job once across instances

Problem 5 runs in every replica. Make the nightly job run **once** across the cluster with a distributed lock.

<details><summary>Solution</summary>

```csharp
public class SingletonNightlyReport(IServiceProvider services, ILogger<SingletonNightlyReport> log) : BackgroundService {
    private readonly CronExpression _cron = CronExpression.Parse("0 2 * * *");

    protected override async Task ExecuteAsync(CancellationToken ct) {
        while (!ct.IsCancellationRequested) {
            var next = _cron.GetNextOccurrence(DateTime.UtcNow, TimeZoneInfo.Utc)!.Value;
            await Task.Delay(next - DateTime.UtcNow, ct);

            await using var scope = services.CreateAsyncScope();
            var locker = scope.ServiceProvider.GetRequiredService<IDistributedLock>();   // Redis/DB-backed
            // TTL longer than the expected job duration so the lease doesn't expire mid-run
            await using var lease = await locker.TryAcquireAsync("nightly-report", TimeSpan.FromMinutes(30), ct);
            if (lease is null) { log.LogInformation("Another instance holds the lock; skipping"); continue; }

            try { await scope.ServiceProvider.GetRequiredService<IReportJob>().RunAsync(ct); }
            catch (Exception ex) { log.LogError(ex, "Nightly report failed"); }
        }
    }
}
```

All instances wake at 2 AM, but only the one that acquires the distributed lock runs the job; the rest skip. The lease TTL exceeds the job duration so it doesn't expire mid-run (with failover if the holder dies). Alternatively, use Hangfire/Quartz clustering or a Kubernetes CronJob. ([05-ScheduledTasks.md](05-ScheduledTasks.md), [08-ReliabilityAndScale.md](08-ReliabilityAndScale.md).)

</details>

---

## Problem 7: Idempotent job processing

A job may be retried/redelivered. Make it idempotent so it runs effectively once.

<details><summary>Solution</summary>

```csharp
public async Task ProcessChargeAsync(Guid jobId, int orderId, decimal amount, CancellationToken ct) {
    await using var scope = services.CreateAsyncScope();
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();

    // Dedup by job id — atomic with the work (one transaction)
    if (await db.ProcessedJobs.AnyAsync(j => j.Id == jobId, ct)) return;   // already processed → skip

    await ChargeAsync(orderId, amount);                    // (ideally also idempotent at the gateway via an idempotency key)
    db.ProcessedJobs.Add(new ProcessedJob { Id = jobId, ProcessedAt = DateTimeOffset.UtcNow });
    await db.SaveChangesAsync(ct);                          // work + dedup marker commit atomically
}
```

Recording the processed `jobId` in the **same transaction** as the work ensures a retry sees "already processed" and skips — so a redelivered/retried job doesn't double-charge. Pair with a gateway idempotency key for the external call. ([08-ReliabilityAndScale.md](08-ReliabilityAndScale.md), [Ch07 §07](../07-Messaging/07-Patterns.md).)

</details>

---

## Problem 8: Durable recurring job with Hangfire

Replace the in-process scheduled job with Hangfire so it's durable and runs once across the cluster.

<details><summary>Solution</summary>

```csharp
builder.Services.AddHangfire(cfg => cfg.UsePostgreSqlStorage(connectionString));
builder.Services.AddHangfireServer();
var app = builder.Build();
app.UseHangfireDashboard("/hangfire", new() { Authorization = [new AdminOnlyFilter()] });  // secured

// Register a recurring job — runs ONCE across all instances (shared storage coordinates)
RecurringJob.AddOrUpdate<ICleanupService>(
    "nightly-cleanup",
    svc => svc.RunAsync(CancellationToken.None),
    Cron.Daily(2));   // 2 AM

public class CleanupService : ICleanupService {
    [AutomaticRetry(Attempts = 3)]                          // durable retries on failure
    public async Task RunAsync(CancellationToken ct) { /* idempotent cleanup */ }
}
```

Hangfire persists the schedule and coordinates via shared storage, so the recurring job runs **once** across replicas (not per-instance), retries on failure, and is visible in the dashboard — no manual distributed lock. Keep the job idempotent. ([06-Hangfire.md](06-Hangfire.md).)

</details>

---

## Problem 9: Graceful shutdown that drains in-flight work

Make a queue consumer finish its current item on shutdown rather than dropping it.

<details><summary>Solution</summary>

```csharp
builder.Services.Configure<HostOptions>(o => o.ShutdownTimeout = TimeSpan.FromSeconds(60));  // drain window

public class DrainingWorker(TaskQueue queue, IServiceProvider services, ILogger<DrainingWorker> log) : BackgroundService {
    protected override async Task ExecuteAsync(CancellationToken stoppingToken) {
        await foreach (var work in queue.DequeueAllAsync(stoppingToken)) {
            // Once stoppingToken fires, DequeueAllAsync stops yielding NEW items,
            // but the current item completes (within ShutdownTimeout).
            await using var scope = services.CreateAsyncScope();
            try { await work(scope.ServiceProvider, stoppingToken); }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested) {
                log.LogInformation("Draining: finishing in-flight work"); break;
            }
            catch (Exception ex) { log.LogError(ex, "Work item failed"); }
        }
    }
}
```

On SIGTERM, the loop stops taking new items but the in-flight one finishes within `ShutdownTimeout`. For work that can't finish in time, use a **durable** queue so a restart resumes it. In Kubernetes, also flip readiness to unhealthy on SIGTERM to stop new traffic. ([08-ReliabilityAndScale.md](08-ReliabilityAndScale.md).)

</details>

---

## Problem 10: Choose the right background-processing approach

For each, pick the approach and justify.
1. Send a confirmation email after an API request; losing it on a rare restart is acceptable.
2. A nightly billing run that must execute exactly once across 5 replicas and must not be lost.
3. Process a high-volume queue of image-resize jobs; scale processing independently of the API.
4. Poll an external API every minute for updates, within a single small service.

<details><summary>Solution</summary>

1. **In-process queue (`Channel<T>`) + `BackgroundService`** — light, app-coupled, loss-on-restart acceptable. ([04-QueuedBackgroundWork.md](04-QueuedBackgroundWork.md).)
2. **Hangfire (or Quartz) recurring job with shared storage**, or a **distributed-lock-guarded** `BackgroundService` — must run **once across replicas** and be **durable** (persisted, retried). A plain `BackgroundService` timer would run 5×. ([06](06-Hangfire.md)/[07](07-QuartzNet.md)/[08](08-ReliabilityAndScale.md).)
3. **Dedicated Worker Service** consuming a **durable broker queue** with **competing consumers** — scale workers independently of the API; each job processed once. ([03](03-WorkerServices.md)/[Ch07](../07-Messaging/README.md).)
4. **`BackgroundService` + `PeriodicTimer`** in the service — simple interval polling, single instance, no overlap. ([05-ScheduledTasks.md](05-ScheduledTasks.md).)

The principle: in-process channel for light app-coupled offload; durable scheduler (Hangfire/Quartz) or distributed lock for run-once durable scheduled work; dedicated worker + broker for independently-scaling high-volume processing; `PeriodicTimer` for simple single-instance intervals.

</details>

---

You can now build correct, production-grade background processing: hosted services and `BackgroundService` loops with scope-per-item and resilient error handling, non-overlapping and cron scheduling, run-once-across-the-cluster coordination, idempotent processing, durable jobs with Hangfire, and graceful shutdown that drains in-flight work.

→ Back to [Chapter 08 README](README.md) · Next chapter: [Chapter 09 — Networking & HTTP](../09-NetworkingAndHttp/README.md)
