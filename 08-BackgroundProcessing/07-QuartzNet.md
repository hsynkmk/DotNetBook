# Quartz.NET

## A full-featured job scheduler

Quartz.NET is a mature, enterprise job-scheduling library — the .NET port of Java's Quartz. Where Hangfire emphasizes durable *queued jobs* with a dashboard, Quartz emphasizes **rich scheduling**: jobs, triggers (cron, calendar, interval), misfire handling, calendars (exclude holidays), and clustering for high-availability scheduling across instances. Use it when scheduling is the central concern and you need fine-grained control.

```csharp
builder.Services.AddQuartz(q => {
    var jobKey = new JobKey("cleanup");
    q.AddJob<CleanupJob>(opts => opts.WithIdentity(jobKey));
    q.AddTrigger(opts => opts
        .ForJob(jobKey)
        .WithIdentity("cleanup-trigger")
        .WithCronSchedule("0 0 2 * * ?"));     // 2 AM daily (Quartz cron format)
});
builder.Services.AddQuartzHostedService(o => o.WaitForJobsToComplete = true);

public class CleanupJob(ICleanupService cleanup) : IJob {
    public async Task Execute(IJobExecutionContext context) => await cleanup.RunAsync(context.CancellationToken);
}
```

The model: **jobs** (`IJob` — what to do) + **triggers** (when to do it) wired together, run by a scheduler hosted in the app.

---

## Jobs and triggers

Quartz separates *what* runs from *when* — a job can have multiple triggers, and triggers are richly configurable:

```csharp
// Cron trigger — calendar/wall-clock schedules
.WithCronSchedule("0 0/15 * * * ?")          // every 15 minutes
.WithCronSchedule("0 30 9 ? * MON-FRI")       // 9:30 AM on weekdays

// Simple trigger — interval + repeat count
.WithSimpleSchedule(s => s.WithIntervalInSeconds(30).RepeatForever())

// Daily/calendar triggers
.WithDailyTimeIntervalSchedule(s => s.StartingDailyAt(TimeOfDay.HourAndMinuteOfDay(9, 0)))
```

- **`IJob`** — the unit of work (`Execute(context)`); receives an execution context (merged job data, cancellation token, fire time).
- **Triggers** — cron (calendar schedules), simple (interval + repeat), daily-time-interval (within a daily window). One job + many triggers = run the same work on multiple schedules.
- **JobDataMap** — pass parameters to a job via its data map.

This job/trigger separation gives more scheduling flexibility than Hangfire's recurring jobs — multiple/complex triggers, calendars, and misfire policies.

---

## Misfire handling

A distinctive Quartz feature: what happens when a trigger **misfires** — the scheduler was down, busy, or paused when a trigger should have fired (e.g., the app was restarting at 2 AM when the nightly job was due)?

```csharp
.WithCronSchedule("0 0 2 * * ?", x => x
    .WithMisfireHandlingInstructionFireAndProceed())   // on misfire: fire once now, then resume schedule
//   alternatives: DoNothing (skip the missed fire), or fire-now-then-resume
```

Misfire instructions let you decide whether a missed scheduled run should be **executed late** (fire-and-proceed) or **skipped** (do-nothing). This matters for jobs where a missed run should still happen (a daily report you don't want to skip) vs. ones where running late is pointless (a "every minute" health poke). Hangfire/PeriodicTimer don't give this fine-grained control.

---

## Clustering — HA scheduling across instances

Like Hangfire, Quartz solves the multi-instance problem ([05-ScheduledTasks.md](05-ScheduledTasks.md)) — but for *scheduling*. With a **persistent job store (ADO.NET) + clustering enabled**, multiple Quartz instances share schedule state, and a triggered job runs on **one** node (with failover if a node dies):

```csharp
builder.Services.AddQuartz(q => {
    q.UsePersistentStore(s => {
        s.UseProperties = true;
        s.UsePostgres(connectionString);
        s.UseClustering();                 // share schedule state; run each fire once across the cluster
    });
});
```

Clustering gives **high-availability scheduling**: triggers fire once across the cluster (not once per instance), and if the node running a job fails, another picks up (per misfire/recovery settings). This is the production setup for scheduled work that must run reliably exactly once across replicas — the alternative to a distributed-lock guard.

---

## Quartz.NET vs Hangfire

Both are durable, both cluster, both solve "run once across instances" — but they emphasize different things:

| | Quartz.NET | Hangfire |
|---|---|---|
| Emphasis | **rich scheduling** (cron, calendars, misfire, triggers) | **durable queued jobs** + dashboard |
| Ad-hoc fire-and-forget | possible but not the focus | first-class (`Enqueue`) |
| Dashboard | not built in (third-party) | **built-in** |
| Scheduling flexibility | **very high** (multiple triggers, misfire policies, calendars) | recurring (cron) + delayed |
| Clustering | yes (persistent store) | yes (shared storage) |
| Best for | complex scheduling requirements | durable jobs + monitoring with simple scheduling |

Choose **Quartz.NET** when scheduling is complex (multiple/calendar triggers, misfire handling, holiday calendars) and you need fine control. Choose **Hangfire** when you want durable fire-and-forget/delayed jobs with a great dashboard and simple recurring schedules suffice. For simple intervals, neither — use **`PeriodicTimer`** ([05](05-ScheduledTasks.md)).

---

## Common gotchas

### Using Quartz for simple intervals

Quartz is heavyweight for "every 30 seconds." Use `PeriodicTimer` in a `BackgroundService` for simple intervals; reserve Quartz for genuinely complex scheduling.

### In-memory store assumed to cluster

The default in-memory job store is **per-instance** (no clustering — same multi-instance problem). For run-once-across-the-cluster, you must use a **persistent store + `UseClustering()`**.

### Ignoring misfire policy

Without a misfire instruction, a missed fire (during downtime) follows the default policy, which may skip or flood. Set the misfire policy deliberately based on whether late execution is wanted.

### Non-idempotent jobs

Clustering + recovery + misfire-fire-now can run a job more than once in edge cases. Make jobs idempotent ([Ch07 §07](../07-Messaging/07-Patterns.md)).

### Long jobs vs `WaitForJobsToComplete`

On shutdown, `WaitForJobsToComplete = true` waits for running jobs — good for graceful drain, but long jobs can exceed the shutdown timeout. Honor the job's cancellation token.

### Scheduling in local time across DST

Quartz cron triggers run in a configured time zone; local-time schedules across DST can skip/double-fire. Be explicit about the time zone (UTC where possible) — [Ch02 §06](../02-BCL/06-DateTimeAndTime.md).

---

## Summary

- **Quartz.NET** is a full-featured **scheduler**: **jobs** (`IJob`) + **triggers** (cron, simple/interval, daily-time, calendars), with a job runnable on multiple triggers and rich **misfire handling** (fire late vs skip a missed run).
- **Clustering** with a persistent store gives **HA scheduling** — triggers fire **once across the cluster** with failover (solving the multi-instance scheduling problem); the in-memory store is per-instance.
- Choose **Quartz** for **complex scheduling** (multiple/calendar triggers, misfire policies); choose **Hangfire** ([06](06-Hangfire.md)) for durable queued jobs + a dashboard with simple scheduling; use **`PeriodicTimer`** ([05](05-ScheduledTasks.md)) for simple intervals.
- Set **misfire policies** deliberately, make jobs **idempotent**, schedule in **UTC**, and honor cancellation on shutdown.

→ Next: [08-ReliabilityAndScale.md](08-ReliabilityAndScale.md)
