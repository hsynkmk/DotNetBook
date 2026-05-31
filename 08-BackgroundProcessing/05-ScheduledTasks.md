# Scheduled Tasks

## Running work on a schedule

A huge class of background work is **periodic** or **scheduled**: run every 30 seconds, every hour, nightly at 2 AM, on a cron expression. The right tool depends on the schedule's complexity — from the built-in `PeriodicTimer` (simple intervals) to cron libraries (complex schedules) to durable job schedulers ([06-Hangfire.md](06-Hangfire.md)/[07-QuartzNet.md](07-QuartzNet.md)).

```csharp
public class HourlyCleanup(IServiceProvider services) : BackgroundService {
    protected override async Task ExecuteAsync(CancellationToken ct) {
        using var timer = new PeriodicTimer(TimeSpan.FromHours(1));
        while (await timer.WaitForNextTickAsync(ct)) {        // fires every hour, no drift/overlap
            await using var scope = services.CreateAsyncScope();
            await scope.ServiceProvider.GetRequiredService<ICleanupJob>().RunAsync(ct);
        }
    }
}
```

---

## `PeriodicTimer` — the modern choice for intervals

`PeriodicTimer` (.NET 6+) is the right primitive for "run every N" work in a `BackgroundService`. Its `WaitForNextTickAsync` is async (no blocked thread), cancellable, and — crucially — **doesn't overlap or drift**:

```csharp
using var timer = new PeriodicTimer(TimeSpan.FromSeconds(30));
while (await timer.WaitForNextTickAsync(stoppingToken)) {
    await DoWorkAsync(stoppingToken);   // even if this takes 20s, the NEXT tick waits its full interval
}
```

Why it beats `while + Task.Delay`:
- **No overlap** — because *you* `await DoWorkAsync` between ticks, the next tick can't start until the current work finishes. (A naive `Timer` callback fires regardless, so a slow callback overlaps the next.)
- **No callback re-entrancy** — the loop is sequential by construction.
- **Cancellable** — `WaitForNextTickAsync(stoppingToken)` returns false / throws on shutdown, so the loop exits cleanly.

Note: `PeriodicTimer` measures the interval **between** ticks (period after the previous tick completes), so total cycle time = interval + work duration. For "every hour on the hour" exact wall-clock scheduling, compute the delay to the next boundary or use a cron library.

---

## `Task.Delay` loop (older pattern) and the callback `Timer`

```csharp
// while + Task.Delay — works, but can drift (interval + work time accumulates) and you manage it manually
while (!ct.IsCancellationRequested) { await DoWorkAsync(ct); await Task.Delay(_interval, ct); }

// System.Threading.Timer — callback-based; callbacks CAN OVERLAP if work outlasts the period (re-entrancy!)
var timer = new Timer(async _ => await DoWorkAsync(), null, 0, 30_000);   // ⚠ overlap risk
```

Prefer **`PeriodicTimer`** over both. The callback `Timer` is the most dangerous — its callbacks re-enter if the work takes longer than the period (two runs concurrently), and exceptions in the callback are easy to lose. Use `PeriodicTimer` in a `BackgroundService` loop instead.

---

## Cron schedules

For "nightly at 2 AM," "every weekday at 9," or other wall-clock/calendar schedules, an interval timer isn't enough — you need **cron**. Compute the next occurrence with a cron library (e.g., **Cronos**) and delay until then:

```csharp
using Cronos;

public class NightlyJob(IServiceProvider services, ILogger<NightlyJob> log) : BackgroundService {
    private readonly CronExpression _cron = CronExpression.Parse("0 2 * * *");   // 02:00 daily

    protected override async Task ExecuteAsync(CancellationToken ct) {
        while (!ct.IsCancellationRequested) {
            DateTime? next = _cron.GetNextOccurrence(DateTime.UtcNow, TimeZoneInfo.Utc);
            if (next is null) break;
            var delay = next.Value - DateTime.UtcNow;
            if (delay > TimeSpan.Zero) await Task.Delay(delay, ct);   // wait until the scheduled time

            await using var scope = services.CreateAsyncScope();
            try { await scope.ServiceProvider.GetRequiredService<INightlyJob>().RunAsync(ct); }
            catch (Exception ex) { log.LogError(ex, "Nightly job failed"); }
        }
    }
}
```

A cron library parses the expression and computes the next occurrence (handling DST, month lengths, etc.); you delay until then, run, and repeat. **Use UTC** for scheduling to avoid DST ambiguity ([Ch02 §06](../02-BCL/06-DateTimeAndTime.md)) — or be deliberate about the time zone if "2 AM local" is required.

---

## The multi-instance problem (the critical caveat)

A scheduled `BackgroundService` runs **in every instance** of your app. So if you deploy 3 replicas, your "nightly cleanup" runs **3 times** at 2 AM — once per instance. For idempotent work that may be fine; for non-idempotent work (sending a daily report, charging subscriptions) it's a serious bug.

Solutions for "run once across the cluster":
- **Distributed lock / leader election** — only the instance holding a lock (in Redis, the DB, or via a leader-election library) runs the job; others skip ([08-ReliabilityAndScale.md](08-ReliabilityAndScale.md)).
- **A durable job scheduler with a shared store** — **Hangfire**/**Quartz.NET** with clustering ensure a recurring job runs once across all instances ([06](06-Hangfire.md)/[07](07-QuartzNet.md)).
- **An external scheduler** — Kubernetes `CronJob`, a cloud scheduler (Azure timer-triggered Functions, AWS EventBridge), or a dedicated single-instance scheduler service that triggers the work (e.g., enqueues a message).

```csharp
// Distributed-lock guard (conceptual): only one instance runs the job
await using var scope = services.CreateAsyncScope();
var locker = scope.ServiceProvider.GetRequiredService<IDistributedLock>();
await using var lease = await locker.TryAcquireAsync("nightly-job", TimeSpan.FromMinutes(10), ct);
if (lease is not null) await RunJobAsync(ct);   // only the lock holder runs it
```

**This is the #1 scheduled-task mistake**: assuming a `BackgroundService` timer runs once when, across replicas, it runs N times. Always consider multi-instance behavior for scheduled work.

---

## Common gotchas

### Callback `Timer` overlap

`System.Threading.Timer` callbacks re-enter if work outlasts the period (concurrent runs, lost exceptions). Use `PeriodicTimer` in a `BackgroundService` loop.

### Scheduled job runs once per instance

A `BackgroundService` timer fires in **every** replica → N runs. For run-once-per-cluster work, use a distributed lock/leader election, a clustered job scheduler, or an external scheduler.

### Drift with `Task.Delay` loops

`while { work; await Task.Delay }` accumulates work-time into the interval (drift). `PeriodicTimer` is cleaner; for exact wall-clock times use cron + computed delay.

### Local time + DST for schedules

Scheduling in local time across DST transitions causes skipped/double runs. Schedule in **UTC** (or handle DST explicitly with a zone-aware cron library) — [Ch02 §06](../02-BCL/06-DateTimeAndTime.md).

### Ignoring the stopping token

A scheduled loop must honor `stoppingToken` (in the delay and the work) for graceful shutdown.

### Non-idempotent scheduled work

If a job might run more than once (multi-instance, retries, restarts), make it idempotent or guard it — don't assume exactly-once.

---

## Summary

- For **simple intervals**, use **`PeriodicTimer`** in a `BackgroundService` loop — async, cancellable, and **no overlap/re-entrancy** (you await the work between ticks); prefer it over `while + Task.Delay` and the callback `Timer`.
- For **cron/wall-clock schedules** ("nightly at 2 AM"), use a cron library (Cronos) to compute the next occurrence and delay until it — **schedule in UTC** to dodge DST issues.
- **The critical caveat**: a scheduled `BackgroundService` runs in **every instance** → N executions across replicas. For run-once-per-cluster work, use a **distributed lock/leader election**, a **clustered job scheduler** (Hangfire/Quartz — [06](06-Hangfire.md)/[07](07-QuartzNet.md)), or an **external scheduler** (Kubernetes CronJob, cloud timer).
- Honor the stopping token; make scheduled work **idempotent** (it may run more than once).

→ Next: [06-Hangfire.md](06-Hangfire.md)
