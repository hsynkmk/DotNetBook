# 08 — Background Processing

## ⚡ 30-second answer

**`IHostedService`** is the host's hook for work outside requests; services **start in
registration order and stop in reverse**, and **`StartAsync` blocks startup** — so long-running
loops belong in **`BackgroundService.ExecuteAsync(stoppingToken)`**. Hosted services are
**singletons**, so the cardinal rule is **create a DI scope per work item** to use scoped services
like `DbContext` — otherwise you have a captive dependency shared across every item. Honor the
**stopping token** everywhere so shutdown drains cleanly. The trap that catches everyone in
production: a scheduled `BackgroundService` **runs in every replica**, so "nightly at 2am" runs N
times — for run-once-per-cluster work you need a distributed lock, a clustered scheduler
(**Hangfire**/**Quartz.NET**), or an external one (Kubernetes CronJob). And because everything
here is at-least-once, **make background jobs idempotent**.

---

## Core mechanics

### The two interfaces

```csharp
// IHostedService — explicit start/stop hooks. StartAsync BLOCKS startup.
public class Warmup(IServiceScopeFactory scopes) : IHostedService
{
    public Task StartAsync(CancellationToken ct) => …;   // keep fast
    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}

// BackgroundService — long-running loops. Started after startup, cancelled on shutdown.
public class Worker(IServiceScopeFactory scopes, ILogger<Worker> log) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromMinutes(5));
        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            try
            {
                await using var scope = scopes.CreateAsyncScope();      // ← scope per iteration
                var db = scope.ServiceProvider.GetRequiredService<AppDb>();
                await DoWorkAsync(db, stoppingToken);
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
            {
                break;                                                   // shutdown, not a failure
            }
            catch (Exception ex)
            {
                log.LogError(ex, "Iteration failed");                    // log and keep going
            }
        }
    }
}
```

Three things in that snippet are the whole lesson: **scope per iteration**, **`PeriodicTimer`**
(async, cancellable, non-overlapping), and **try/catch per iteration** — because an unhandled
exception in `ExecuteAsync` **stops the host** by default.

### The queued-work pattern

```csharp
// Producer (a request handler): enqueue and return 202 immediately
await queue.EnqueueAsync(new SendEmail(orderId), ct);
return TypedResults.Accepted($"/jobs/{id}");
```

A singleton **`Channel<T>`-backed queue** plus a `BackgroundService` consumer. Rules:

- **A fresh DI scope per work item.** The singleton consumer can't inject `DbContext`.
- **Capture plain data in the work item** — ids and values, never request-scoped objects
  (`HttpContext`, a `DbContext`, the `ClaimsPrincipal`). They're disposed by the time you run.
- **Bounded channel** for back-pressure (`Wait`) or load-shedding (`Drop*`).
- **Bound parallelism** so draining a backlog doesn't flatten the database.
- **try/catch per item**, so one failure doesn't kill the consumer.
- In-memory queues **lose items on restart** — use a broker ([07](07-Messaging.md)) or a durable
  job store when the work must not be lost.

### Worker Services

`dotnet new worker` — the Generic Host without HTTP. Same DI, configuration and logging.

Run background work **in the web app** when it's light and app-coupled; use a **dedicated Worker
Service** when it needs to scale independently, needs failure isolation, or has different resource
needs. Host it as a **Windows Service** (`AddWindowsService`), a **systemd daemon**
(`AddSystemd`), or — for cloud — just a container the orchestrator manages.

### Scheduling

| Need | Use |
|---|---|
| Simple interval | **`PeriodicTimer`** in a `BackgroundService` |
| Cron / wall-clock ("2 AM daily") | a cron library (Cronos) — **compute in UTC** |
| Durable queued jobs + recurring + dashboard | **Hangfire** |
| Complex scheduling: multiple triggers, calendars, misfire policies | **Quartz.NET** |
| Run-once-per-cluster, infra-managed | **Kubernetes CronJob** / cloud timer |

**Hangfire** stores jobs in SQL/Postgres/Redis, so they survive restarts, **retry automatically**,
and recurring jobs run **once across the cluster**. Jobs are serialized method-call expressions —
pass **ids and simple values**, never entities, and re-fetch inside the job. Secure the dashboard.

**Quartz.NET** is a full scheduler: jobs plus triggers (cron, interval, daily-time, calendars),
with explicit **misfire policies** (fire late, or skip a missed run). Clustering with a persistent
store gives HA scheduling with failover.

### Reliability

- **Graceful shutdown**: honor the stopping token, finish in-flight work within `ShutdownTimeout`,
  and make unfinished work **resumable**. Pair with readiness draining in Kubernetes
  ([04](04-AspNetCore.md)).
- **At-least-once is the reality** — retries, redelivery and crash recovery all mean a job can run
  twice. Design for idempotency; exactly-once is impractical.
- **Run-once-per-cluster** needs a distributed lock, leader election, or a clustered/external
  scheduler. **Distributable** work scales the opposite way: competing consumers on a durable
  shared queue.
- **Observe it.** Background work fails **silently** — no user sees a 500. Logs, metrics
  (queue depth, processing time, failures), traces, dead-letter alerts, and a health check
  ([12](12-Observability.md)).

---

## Comparison tables

| | `IHostedService` | `BackgroundService` |
|---|---|---|
| Shape | `StartAsync`/`StopAsync` | override `ExecuteAsync` |
| Blocks startup | **yes**, until `StartAsync` returns | no |
| For | one-time setup/teardown hooks | long-running loops |
| Unhandled exception | in `StartAsync` → **host fails to start** | **stops the host** (by default) |

| | In-process channel | Hangfire | Broker (Rabbit/SB/Kafka) |
|---|---|---|---|
| Durable | ❌ lost on restart | ✅ | ✅ |
| Cross-service | ❌ | ❌ (in-app) | ✅ |
| Retries | you write them | **built in** | built in |
| Runs once per cluster | ❌ per instance | ✅ | ✅ |
| Complexity | lowest | middle | highest |

| Scheduler | Strength |
|---|---|
| `PeriodicTimer` | simplest; non-overlapping; **per-instance** |
| Hangfire | durable jobs, retries, dashboard, cluster-safe recurring |
| Quartz.NET | richest triggers + misfire policies + HA clustering |
| K8s CronJob | infra-owned, language-agnostic, one run per schedule |

---

## 🪤 Traps & gotchas

- **Injecting `DbContext` into a `BackgroundService`** — a captive dependency. The hosted service
  is a singleton, so one context is shared across every work item for the app's lifetime, on
  multiple threads. Scope per work item ([03](03-Hosting-DI-Config.md)).
- **Sharing one scope across all items** — tracked entities accumulate, memory grows, and one
  item's failure poisons the next.
- **Long-running work in `StartAsync`** — it blocks host startup, so your app never becomes ready
  and the orchestrator kills it.
- **An unhandled exception in `ExecuteAsync` stops the host.** Surprising the first time. Wrap
  each iteration; catch `OperationCanceledException` from shutdown separately so a clean stop
  isn't logged as an error.
- **Ignoring the stopping token** — `Thread.Sleep`, or an async call without the token, means
  shutdown waits for `ShutdownTimeout` and then kills you mid-work.
- **A scheduled `BackgroundService` runs in every replica.** Three pods means your nightly report
  is generated and emailed three times. The single most common background-processing bug in
  production.
- **`while (true) { await Task.Delay(interval); await Work(); }`** — the interval doesn't account
  for how long the work took, and a slow iteration can overlap the next. `PeriodicTimer` waits
  properly and doesn't re-enter.
- **Cron schedules in local time** — DST means "2 AM daily" runs twice on one night and not at all
  on another. Schedule in UTC.
- **Capturing request-scoped objects in a queued work item** — the `DbContext` is disposed and
  `HttpContext` is gone by the time the consumer runs. Capture ids and plain data.
- **Losing user/tenant context** when work moves to the background — there's no `HttpContext`, so
  ambient context is empty. Capture it at enqueue time
  ([22](22-BestPractices-Architecture.md)).
- **Unbounded parallelism when draining a backlog** — the worker recovers, processes 50,000 queued
  items at once, and takes down the database. Bound it.
- **Passing entities to Hangfire** — the whole object is serialized into storage, goes stale, and
  breaks on the next model change. Pass the id.
- **Non-idempotent jobs with automatic retries** — Hangfire retries by default, so "send the
  invoice email" sends it three times.
- **An unsecured Hangfire dashboard** — it's remote job execution with a UI, and by default it's
  only protected on localhost.
- **Assuming a job ran** — background failures are silent. Without metrics and alerting, you find
  out when a customer asks where their report is.
- **In-memory queue plus a rolling deploy** — every queued item is silently dropped on each
  restart.
- **No dead-letter handling** — a permanently failing item retries forever and starves everything
  behind it.

---

## ❓ Likely questions

**Q: `IHostedService` vs `BackgroundService`?**
A: `IHostedService` gives explicit `StartAsync`/`StopAsync` hooks, and `StartAsync` **blocks host
startup** — so it's for fast one-time setup and teardown. `BackgroundService` is the base class
for long-running loops: you override `ExecuteAsync(stoppingToken)`, and the base class starts it
without blocking startup and cancels and awaits it on shutdown.

**Q: How do you use a scoped service like `DbContext` in a hosted service?**
A: You can't inject it — the hosted service is a singleton, so that's a captive dependency. Inject
`IServiceScopeFactory`, create a scope per work item or iteration with `CreateAsyncScope`, resolve
from that scope, and dispose it when the item completes.

**Q: What happens if `ExecuteAsync` throws?**
A: By default the host stops — `BackgroundServiceExceptionBehavior.StopHost` since .NET 6. That's
usually right (a dead worker shouldn't look healthy), but for a resilient worker you wrap each
iteration in try/catch, log, and continue.

**Q: Why `PeriodicTimer` rather than `while (true) { await Task.Delay(...) }`?**
A: `PeriodicTimer` is async, cancellable, and — because you await the work *between* ticks —
non-overlapping. The `Task.Delay` loop makes the effective interval `delay + work time`, and the
callback-based `Timer` can re-enter while the previous run is still going.

**Q: What's the biggest gotcha with scheduled work in a scaled-out app?**
A: A `BackgroundService` runs in **every instance**, so a nightly job runs once per replica. For
run-once-per-cluster you need a distributed lock or leader election, a clustered scheduler like
Hangfire or Quartz, or an external scheduler such as a Kubernetes CronJob.

**Q: Describe the queued background work pattern.**
A: A singleton queue backed by a bounded `Channel<T>`, and a `BackgroundService` consumer. The
request handler enqueues a work item and returns 202 immediately; the consumer dequeues, creates a
DI scope per item, processes it, and catches exceptions per item so one failure doesn't stop the
loop.

**Q: What do you capture in a queued work item, and what must you not?**
A: Capture plain data — ids, values, a small DTO. Never capture request-scoped objects: the
`DbContext` will be disposed and `HttpContext` gone by the time the item runs. Resolve services
from the per-item scope instead.

**Q: When do you use Hangfire instead of an in-process queue?**
A: When the work must survive a restart. Hangfire persists jobs to SQL/Postgres/Redis, retries
automatically with backoff, runs recurring jobs once across a cluster, and gives you a dashboard —
all things you'd otherwise build. It's the middle ground between a channel and a full broker.

**Q: Hangfire or Quartz.NET?**
A: Hangfire for durable queued jobs plus simple recurring schedules and a dashboard. Quartz for
genuinely complex scheduling — multiple triggers per job, calendars, and explicit misfire policies
for what should happen when a run is missed. Both cluster for HA.

**Q: When do you split background work into a separate Worker Service?**
A: When it needs to scale independently of the API, needs failure isolation (a runaway job
shouldn't affect request latency), or has different resource requirements. Keeping it in the web
app is fine for light, app-coupled tasks.

**Q: How do you shut a worker down gracefully?**
A: Honor the stopping token through every async call, finish in-flight work inside
`ShutdownTimeout`, and make anything unfinished resumable — which means durable state, since
in-memory queues are dropped. In Kubernetes, fail readiness first so nothing new is routed while
you drain.

**Q: Why must background jobs be idempotent?**
A: Because everything here is at-least-once. Retries after a transient failure, a crash before the
job was marked complete, and redelivery from a broker all cause the same job to run twice. Design
so that running twice is harmless.

**Q: How do you monitor background work?**
A: Deliberately, because it fails silently — nobody gets a 500. Log each item with a correlation
id, emit metrics for queue depth, processing duration and failure count, trace the work as a span
linked to the originating request, and alert on dead-letter depth and on the *absence* of expected
runs.

---

## 🎓 Senior Extra

- **A distributed lock is subtler than it looks.** Lock leases expire, so a job that outruns its
  lease can run concurrently with the next holder. Use fencing tokens or make the job idempotent —
  and prefer a clustered scheduler that has already solved this.
- **Leader election** (a lease row in the database, or the Kubernetes lease API) is often simpler
  and more debuggable than a per-job distributed lock when you have many scheduled jobs.
- **Misfire policy is a business decision.** If the process was down at 2 AM, should the nightly
  batch run at 6 AM when it comes back, or skip to tomorrow? Quartz makes you answer; most
  hand-rolled schedulers silently pick "skip".
- **Queue depth is the metric that predicts an outage** — it rises long before latency does, and
  it's the honest signal of whether you have enough consumer capacity.
- **Bounded parallelism belongs next to the downstream it protects**, not at the worker. A
  `SemaphoreSlim` sized to your database's connection pool is more meaningful than a guess at
  `MaxDegreeOfParallelism`.
- **`IHostApplicationLifetime`** gives you `ApplicationStarted`/`Stopping`/`Stopped` — the right
  hooks for registering with a service registry, draining, and flushing telemetry.
- **Don't run EF migrations from a hosted service in a multi-replica deployment** — every replica
  races to apply them. It's the canonical example of "this works on one instance and corrupts on
  three" ([05](05-EFCore.md)).
- **The outbox relay is itself a background service** — and the one place where at-least-once
  publishing is the *feature*, not a hazard ([07](07-Messaging.md)).
- **Worker Services get a health check too.** A worker with no HTTP endpoint can still expose one
  for liveness, or heartbeat into a table the monitoring system watches — otherwise a wedged
  consumer looks identical to an idle one.
- **`ShutdownTimeout` defaults to 30 seconds** but Kubernetes' `terminationGracePeriodSeconds`
  defaults to 30 too, so a long-draining worker gets SIGKILLed mid-item. Set both deliberately and
  make the work resumable regardless.

→ Deeper: [`../08-BackgroundProcessing/`](../08-BackgroundProcessing/README.md)
