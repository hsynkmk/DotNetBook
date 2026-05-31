# Reliability & Scale

## Making background work production-grade

Background work has reliability concerns request handlers don't: it must shut down gracefully (without losing in-flight work), survive crashes, run correctly across multiple instances, and not silently fail. This file ties together the cross-cutting reliability patterns — graceful shutdown, idempotency, exactly-once vs at-least-once, scaling out (leader election/distributed locks), and the outbox.

---

## Graceful shutdown

When a deploy or scale-down sends **SIGTERM** (or Ctrl+C), the host begins graceful shutdown: it signals the `stoppingToken` and gives hosted services a bounded window to finish. Honoring this is what lets you deploy without dropping in-flight work:

```csharp
// Configure the drain window (default 30s)
builder.Services.Configure<HostOptions>(o => o.ShutdownTimeout = TimeSpan.FromSeconds(60));

public class GracefulWorker(IBackgroundTaskQueue queue, IServiceProvider services,
    IHostApplicationLifetime lifetime, ILogger<GracefulWorker> log) : BackgroundService {
    protected override async Task ExecuteAsync(CancellationToken stoppingToken) {
        lifetime.ApplicationStopping.Register(() => log.LogInformation("Draining in-flight work..."));
        await foreach (var item in queue.DequeueAllAsync(stoppingToken)) {
            await using var scope = services.CreateAsyncScope();
            try { await ProcessAsync(item, scope.ServiceProvider, stoppingToken); }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested) { break; }
        }
        // loop exits when the token fires; in-flight item finishes within ShutdownTimeout
    }
}
```

- The **stopping token** stops the consumer loop from taking *new* work; the **in-flight item** finishes (within `ShutdownTimeout`, after which the host force-terminates).
- `IHostApplicationLifetime.ApplicationStopping` lets you react (stop accepting work, flush, log).
- In Kubernetes, pair this with **readiness** flipping to unhealthy on SIGTERM so the load balancer stops routing while you drain ([Ch04 §16](../04-AspNetCore/16-HealthChecks.md)).

The key insight: **honor cancellation and bound your work** so each item finishes (or is safely abandoned) within the shutdown window. Work that can't finish in time should be **resumable** (persisted/durable) so a restart picks it up.

---

## At-least-once vs exactly-once

The fundamental reliability reality (shared with messaging — [Ch07 §07](../07-Messaging/07-Patterns.md)): in a distributed system with crashes and retries, **exactly-once execution is generally impossible**; you get **at-least-once** (with retries) or **at-most-once** (no retries, possible loss).

```
At-least-once:  retry on failure → work may run MORE than once → must be IDEMPOTENT
At-most-once:   no retry → work may NOT run at all → possible loss
```

Almost always you want **at-least-once + idempotent processing**: retry until it succeeds, and design the work so running it twice is harmless. A job interrupted by a crash/shutdown will be retried (if durable) — so it must tolerate partial-then-repeat execution.

---

## Idempotency (the linchpin)

Because background work runs at-least-once (retries, restarts, multi-instance), it **must be idempotent** — running it twice has the same effect as once:

```csharp
// Idempotent by design — guard with a processed-marker (atomic with the work)
public async Task ProcessAsync(Guid jobId, ...) {
    await using var scope = services.CreateAsyncScope();
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    if (await db.ProcessedJobs.AnyAsync(j => j.Id == jobId)) return;   // already done → skip
    DoWork(db);
    db.ProcessedJobs.Add(new ProcessedJob { Id = jobId });
    await db.SaveChangesAsync();   // work + marker in ONE transaction → atomic dedup
}
```

Strategies (from [Ch07 §07](../07-Messaging/07-Patterns.md)): dedup by job id (atomically with the work), natural idempotency ("set status = Done" not "increment"), or conditional/upsert writes. A non-idempotent background job double-charges, double-sends, or double-counts when retried — the most common production reliability bug.

---

## Scaling out: run-once across instances

A `BackgroundService` runs in **every** replica ([05-ScheduledTasks.md](05-ScheduledTasks.md)). For work that must run **once across the cluster** (a nightly batch, a singleton processor), you need coordination:

### Distributed lock

Only the instance holding a shared lock runs the work; others skip:

```csharp
await using var scope = services.CreateAsyncScope();
var locker = scope.ServiceProvider.GetRequiredService<IDistributedLock>();   // Redis/DB-backed
await using var lease = await locker.TryAcquireAsync("nightly-batch", TimeSpan.FromMinutes(30), ct);
if (lease is null) return;   // another instance holds it → skip
await RunBatchAsync(ct);     // only the lock holder runs it
```

A distributed lock (Redis `SET NX`, a DB row lock, or a library) ensures single execution. Mind lock **expiry** — if the holder dies, the lease should expire so another instance can take over (but not while the first is still working — use a TTL longer than the work, or renew).

### Leader election

One instance is elected "leader" and runs all singleton work; others stand by (failover if the leader dies). Built into orchestration platforms (Kubernetes lease objects) and libraries.

### Delegate to a coordinator

Use a **durable job scheduler** (Hangfire/Quartz clustering — [06](06-Hangfire.md)/[07](07-QuartzNet.md)) or an **external scheduler** (Kubernetes `CronJob`, cloud timer) that inherently runs once, instead of a per-instance timer.

The pattern: **competing consumers** (every instance pulls from a shared queue → each item processed once, scales horizontally) for *distributable* work; **leader/lock** for *singleton* work that can't be parallelized.

---

## Scaling consumers (competing consumers)

For *distributable* background work (process a queue), scale by running **more consumer instances** that compete for items from a shared durable queue/broker — each item goes to one consumer ([Ch07 §03](../07-Messaging/03-RabbitMQ.md)):

```
Shared durable queue (broker / Hangfire storage) ← many worker instances pull
   → each message/job processed by exactly one worker → add workers to scale throughput
```

This requires a **durable, shared** queue (a broker or Hangfire-style store) — an in-process channel can't be shared across instances. Bound per-instance concurrency to protect downstreams (DB pool, external APIs — [04-QueuedBackgroundWork.md](04-QueuedBackgroundWork.md)).

---

## The outbox (reliable publish from background work)

When background work needs to both update the database **and** publish a message/event, use the **outbox pattern** to avoid the dual-write problem ([Ch07 §07](../07-Messaging/07-Patterns.md), [Ch05 §08](../05-EFCore/08-Transactions.md)): write the outgoing message to an outbox table in the **same transaction** as the DB change, and relay it separately. This guarantees the DB change and the publish are atomic — no lost or phantom messages even if the process crashes between them.

---

## Observability for background work

Background work fails **silently** (no user gets an error response), so observability is essential:
- **Log** each job's start/success/failure with correlation ids; **don't swallow** exceptions silently.
- **Metrics** — items processed/sec, queue depth, failure rate, processing duration (`Meter` — [Ch02 §08](../02-BCL/08-Diagnostics.md), [Ch12](../12-Observability/README.md)).
- **Traces** — wrap processing in an `Activity` so background work appears in distributed traces.
- **Dead-letter monitoring** — alert on growing failed-job / dead-letter queues.
- **Health checks** — report worker liveness/readiness ([Ch04 §16](../04-AspNetCore/16-HealthChecks.md)).

A queue silently backing up or a worker silently dying is a common production incident — instrument so you find out.

---

## Common gotchas

### Dropping in-flight work on shutdown

Ignoring the stopping token / not bounding work means in-flight items are killed on deploy. Honor cancellation, finish within `ShutdownTimeout`, and make work resumable (durable) for what can't finish.

### Non-idempotent work + retries

At-least-once + retries/restarts means possible double execution. Make all background work idempotent — the #1 reliability bug.

### Singleton work running per-instance

A `BackgroundService` timer/loop runs in every replica. For run-once work, use a distributed lock, leader election, or a clustered/external scheduler.

### Scaling consumers on an in-process queue

You can't share an in-memory channel across instances. Competing-consumer scaling needs a durable shared queue (broker/Hangfire).

### Silent failures

Background errors don't surface to users. Log, meter, trace, and alert — don't swallow exceptions.

### Lock expiry races

A distributed lock that expires while the holder is still working lets a second instance run concurrently. Use a TTL longer than the work (or renew the lease).

---

## Summary

- **Graceful shutdown**: honor the **stopping token**, finish in-flight work within `ShutdownTimeout`, and make unfinished work **resumable** (durable); pair with readiness draining in Kubernetes.
- Distributed reality is **at-least-once** — design every background job to be **idempotent** (dedup by id atomically, natural idempotency, conditional writes); exactly-once is generally impractical.
- For **run-once-across-the-cluster** work, coordinate with a **distributed lock**, **leader election**, or a **clustered/external scheduler**; for *distributable* work, scale with **competing consumers** on a **durable shared queue** (broker/Hangfire) — an in-process channel can't scale across instances.
- Use the **outbox** for reliable DB-write-plus-publish; **observe** background work (logs, metrics, traces, dead-letter alerts, health checks) since it fails silently.

→ Next: [Questions.md](Questions.md)
