# Hangfire

## Durable background jobs with a dashboard

Hangfire is a popular library for **durable** background jobs — fire-and-forget, delayed, and recurring — backed by persistent storage (SQL Server, PostgreSQL, Redis). Unlike an in-process `Channel<T>` queue (lost on restart — [04-QueuedBackgroundWork.md](04-QueuedBackgroundWork.md)), Hangfire **persists jobs to a database**, so they survive restarts/crashes, retry automatically on failure, and run **once across a cluster** — plus it gives you a web dashboard to monitor everything.

```csharp
builder.Services.AddHangfire(cfg => cfg.UsePostgreSqlStorage(connectionString));
builder.Services.AddHangfireServer();   // the worker that processes jobs

var app = builder.Build();
app.UseHangfireDashboard("/hangfire");   // monitoring UI (secure it!)

// Fire-and-forget — enqueue a job, processed by a Hangfire worker
BackgroundJob.Enqueue<IEmailService>(svc => svc.SendWelcomeAsync(userId));
```

The job's method and arguments are **serialized to the database**; a Hangfire server picks it up and invokes it. Because it's persisted, the job runs even if the app restarts before processing it.

---

## Job types

```csharp
// Fire-and-forget — run once, ASAP
BackgroundJob.Enqueue<IReportService>(s => s.GenerateAsync(reportId));

// Delayed — run once, after a delay
BackgroundJob.Schedule<IEmailService>(s => s.SendReminderAsync(id), TimeSpan.FromHours(24));

// Recurring — run on a cron schedule (named; idempotent registration)
RecurringJob.AddOrUpdate<ICleanupService>("nightly-cleanup", s => s.RunAsync(), Cron.Daily(2));

// Continuation — run after a parent job completes
var parentId = BackgroundJob.Enqueue<IService>(s => s.StepOneAsync());
BackgroundJob.ContinueJobWith<IService>(parentId, s => s.StepTwoAsync());
```

- **Fire-and-forget** — durable version of enqueueing work (survives restart).
- **Delayed** — run later (reminders, deferred processing).
- **Recurring** — cron-scheduled jobs, registered by name (re-registering updates them — idempotent).
- **Continuations** — chain jobs (step B after step A succeeds).

The jobs are expressed as method-call expressions (`s => s.MethodAsync(args)`); Hangfire serializes the type, method, and arguments to storage and resolves the service from DI when running.

---

## Durability & automatic retries

The core value over an in-process queue: **persistence + retries**.

- **Persistence** — jobs live in the database until completed. An app crash/restart doesn't lose enqueued or in-progress jobs; a Hangfire server resumes them.
- **Automatic retry** — a job that throws is **retried automatically** (default ~10 attempts with increasing delays). Configure per job:

```csharp
[AutomaticRetry(Attempts = 5, DelaysInSeconds = new[] { 60, 300, 900 })]
public async Task SendAsync(int id) { ... }   // retried with backoff on failure
```

After retries are exhausted, the job is marked **failed** (visible in the dashboard) for inspection/manual requeue — Hangfire's equivalent of dead-lettering. This durable-retry behavior is what makes Hangfire suitable for work that **must** eventually run (charging, notifications, syncs).

---

## Runs once across a cluster

A key advantage over a scheduled `BackgroundService` (which runs per-instance — [05-ScheduledTasks.md](05-ScheduledTasks.md)): Hangfire's shared storage coordinates workers, so an **enqueued or recurring job runs exactly once** regardless of how many app/server instances you run. Multiple Hangfire servers compete for jobs from the shared queue — each job is processed by one. So your "nightly cleanup" registered with `RecurringJob.AddOrUpdate` runs **once** at 2 AM, not once per replica. This solves the multi-instance scheduling problem out of the box.

---

## The dashboard

```csharp
app.UseHangfireDashboard("/hangfire", new DashboardOptions {
    Authorization = [new MyDashboardAuthFilter()]   // SECURE it — don't expose publicly!
});
```

Hangfire's web dashboard shows enqueued/processing/succeeded/failed jobs, recurring-job schedules, retry history, and lets you requeue failed jobs or trigger recurring ones manually. It's a major operational benefit — visibility into background work. **Secure the dashboard** (it exposes job data and controls); the default allows only local access, so configure authorization for production.

---

## Hangfire vs in-process queue vs broker

| | In-process `Channel<T>` | Hangfire | Message broker (Ch07) |
|---|---|---|---|
| Durability | none (lost on restart) | **DB-persisted** | broker-persisted |
| Retries | manual | **automatic + backoff** | configurable |
| Runs once across cluster | no (per-instance) | **yes** (shared storage) | yes (competing consumers) |
| Scheduling (cron/delayed) | manual | **built-in** | partial |
| Dashboard | none | **yes** | broker tools |
| Cross-service | no | within apps sharing the storage | **yes** (decoupled services) |
| Best for | best-effort in-process offload | durable in-app jobs + scheduling | cross-service messaging |

Hangfire occupies the middle ground: more durable/featureful than an in-process queue, simpler than a full message broker. Use it for **durable background jobs and scheduled work within an app (or apps sharing its storage)** where you want persistence, automatic retries, scheduling, and a dashboard without standing up a broker. For **cross-service** event-driven messaging, a broker ([Ch07](../07-Messaging/README.md)) is the better fit.

---

## Common gotchas

### Unsecured dashboard

The dashboard exposes job data and lets users requeue/trigger jobs. Don't expose it publicly — configure authorization (it's local-only by default).

### Storing complex objects as job arguments

Job arguments are serialized to the database. Pass **simple values/ids**, not large or complex objects (and not entities) — keep the serialized payload small and stable. Re-fetch entities inside the job from the id.

### Non-idempotent jobs with automatic retry

Hangfire retries failed jobs (and at-least-once semantics mean a job could run more than once). Make jobs **idempotent** — a retried "charge customer" must not double-charge ([Ch07 §07](../07-Messaging/07-Patterns.md)).

### Long-running jobs and shutdown

A long job interrupted by shutdown is retried (it's persisted), so it may run partially twice — another reason for idempotency. Honor cancellation where possible.

### Coupling apps via shared Hangfire storage

Multiple apps sharing one Hangfire database can run each other's jobs (any server processes any job). That's a feature for a job cluster but a coupling risk if unintended — isolate storage per logical job domain when needed.

### Using Hangfire where a broker fits better

For decoupled cross-service event-driven communication, a message broker is more appropriate than enqueueing Hangfire jobs across service boundaries.

---

## Summary

- **Hangfire** runs **durable** background jobs (fire-and-forget, delayed, recurring, continuations) backed by **persistent storage** (SQL/PostgreSQL/Redis) — jobs survive restarts, **retry automatically** with backoff, and run **once across a cluster**.
- Jobs are method-call expressions serialized to storage; pass **simple ids/values** (not entities), and re-fetch inside the job.
- It solves the multi-instance scheduling problem (recurring jobs run once, not per-replica) and provides a **dashboard** for monitoring/requeueing — **secure the dashboard**.
- Make jobs **idempotent** (retries + at-least-once mean possible double execution).
- It's the middle ground: more durable/featureful than an in-process queue ([04](04-QueuedBackgroundWork.md)), simpler than a broker — use it for durable in-app jobs + scheduling; use a **broker** ([Ch07](../07-Messaging/README.md)) for cross-service messaging.

→ Next: [07-QuartzNet.md](07-QuartzNet.md)
