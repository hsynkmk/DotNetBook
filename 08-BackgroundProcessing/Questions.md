# Chapter 08 — Background Processing — Q & A

---

### Q1. What is `IHostedService` and how do its methods behave?

The host's hook for components running outside requests. `StartAsync` is called at startup (in registration order) and **blocks startup until it returns** — keep it fast. `StopAsync` is called at shutdown (reverse order, bounded by the shutdown timeout). Services start in order, stop in reverse.

---

### Q2. Why not put a long-running loop in `StartAsync`?

`StartAsync` blocks the host from finishing startup until it returns — a loop that never returns means the app never becomes ready. Kick off background work and return, or (better) use `BackgroundService`, whose `ExecuteAsync` runs after startup without blocking it.

---

### Q3. When use raw `IHostedService` vs `BackgroundService`?

Use `BackgroundService` for continuous loops (it handles start-without-blocking and stop-with-cancellation). Use raw `IHostedService` for explicit start/stop hooks — one-time startup work (cache warm, migrations), resource setup/teardown — where there's no continuous loop.

---

### Q4. How does `BackgroundService` avoid blocking startup?

Its `StartAsync` calls `ExecuteAsync` but doesn't await it — it returns at `ExecuteAsync`'s first `await`, so the long loop runs independently without blocking host startup. On shutdown, `StopAsync` cancels the stopping token and awaits `ExecuteAsync` to finish.

---

### Q5. What's the default exception behavior of `BackgroundService.ExecuteAsync`?

By default (.NET 6+), an unhandled exception **stops the host** (crashes the app). Configurable via `HostOptions.BackgroundServiceExceptionBehavior` (`Ignore` = log and keep running). For resilient workers, wrap each iteration in try/catch so transient failures don't crash the app.

---

### Q6. Why create a DI scope per work item in a background service?

A hosted service is a **singleton**; it can't inject scoped services (`DbContext`) — that's a captive dependency. Create a scope per item (`CreateAsyncScope`) and resolve scoped services from it, like a per-request scope. Sharing one `DbContext` across items corrupts state (not thread-safe).

---

### Q7. Why `await Task.Delay(token)` instead of `Thread.Sleep`?

`Thread.Sleep` blocks a thread and isn't cancellable. `await Task.Delay(interval, stoppingToken)` yields the thread (no starvation) and is cancellable, so the service stops promptly on shutdown.

---

### Q8. What is a Worker Service?

A headless .NET app (`dotnet new worker`) for background processing — the Generic Host without ASP.NET Core/HTTP, but the same DI, config, logging, and hosted-service model. Used for daemons: queue processors, scheduled jobs, message consumers.

---

### Q9. Background service in the web app vs a dedicated Worker Service?

In the web app: simplest for light, app-coupled work (one deployable, shared lifecycle). Dedicated Worker Service: when background work must **scale independently** of the API, needs **failure isolation**, or has different resource needs — separate deployable, often communicating with the API via a queue/broker.

---

### Q10. How do you host a Worker Service as an OS service?

`AddWindowsService` (Windows SCM integration) or `AddSystemd` (Linux systemd). In containers/Kubernetes you use neither — the container is the process and the orchestrator manages lifecycle via signals.

---

### Q11. Describe the queued-background-work pattern.

A request handler enqueues a work item into a bounded `Channel<T>` and returns immediately (202); a `BackgroundService` dequeues and processes it off the request path. Decouples request latency from work duration. The consumer creates a fresh DI scope per item for scoped services.

---

### Q12. What should the enqueued closure capture?

**Plain data** (ids, request values) — not request-scoped objects (`HttpContext`, scoped services), which outlive the request improperly. Resolve scoped services (`DbContext`) inside the work item from the per-item scope's provider.

---

### Q13. Bounded vs unbounded queue for background work?

**Bounded** with `FullMode.Wait` gives back-pressure (the producer/endpoint awaits when full, slowing intake under load) or `Drop*` sheds load. **Unbounded** risks memory exhaustion if producers outpace the consumer. Use bounded.

---

### Q14. Best primitive for "run every N seconds" in a background service?

**`PeriodicTimer`** in a `BackgroundService` loop (`while (await timer.WaitForNextTickAsync(token))`) — async, cancellable, and no overlap/re-entrancy (you await the work between ticks). Prefer it over `while + Task.Delay` (drift) and the callback `Timer` (overlap).

---

### Q15. Why is the callback `System.Threading.Timer` risky?

Its callbacks **re-enter** — if the work takes longer than the period, two runs execute concurrently. Exceptions in the callback are also easy to lose. Use `PeriodicTimer` in a sequential loop instead.

---

### Q16. How do you schedule cron/wall-clock work?

Use a cron library (e.g., Cronos) to compute the next occurrence and `await Task.Delay` until then, then run and repeat — or a durable scheduler (Hangfire/Quartz). Schedule in **UTC** to avoid DST ambiguity.

---

### Q17. What's the #1 scheduled-task mistake in multi-instance apps?

A scheduled `BackgroundService` runs in **every** replica → the job runs N times (once per instance). For run-once-per-cluster work, use a distributed lock/leader election, a clustered job scheduler (Hangfire/Quartz), or an external scheduler (K8s CronJob, cloud timer).

---

### Q18. What does Hangfire provide over an in-process queue?

**Durability** (jobs persisted to a DB, survive restarts), **automatic retries** with backoff, **runs once across a cluster** (shared storage coordinates workers), built-in **scheduling** (delayed/recurring), and a **dashboard**. Use it for durable in-app jobs; an in-process channel loses work on restart.

---

### Q19. What should Hangfire job arguments be?

**Simple values/ids** — they're serialized to the database. Don't pass large/complex objects or entities; re-fetch entities inside the job from the id. Keep the serialized payload small and stable.

---

### Q20. Quartz.NET vs Hangfire?

**Quartz** emphasizes rich **scheduling** (cron, calendar, interval triggers, misfire handling, multiple triggers per job) and clusters for HA scheduling. **Hangfire** emphasizes durable **queued jobs** (fire-and-forget/delayed/recurring) with a built-in dashboard. Choose Quartz for complex scheduling, Hangfire for durable jobs + monitoring; `PeriodicTimer` for simple intervals.

---

### Q21. What is misfire handling in Quartz?

What happens when a trigger should have fired but couldn't (scheduler down/busy) — e.g., the app restarted at 2 AM. Misfire instructions decide whether to **fire late** (fire-and-proceed) or **skip** the missed run (do-nothing). Set it based on whether a missed run should still execute.

---

### Q22. How do Hangfire/Quartz solve the multi-instance scheduling problem?

A **shared persistent store + clustering**: instances coordinate via the store so a recurring job/trigger fires **once across the cluster** (and fails over if a node dies), instead of once per instance. The in-memory store doesn't cluster.

---

### Q23. Why must background work be idempotent?

Background work runs **at-least-once** — retries, restarts, and multi-instance coordination mean it can run more than once. A non-idempotent job double-charges/double-sends. Make running it twice equivalent to once (dedup by id atomically, natural idempotency, conditional writes).

---

### Q24. How do you make background work shut down gracefully?

Honor the **stopping token** (stop taking new work; let the in-flight item finish within `ShutdownTimeout`), make unfinished work **resumable** (durable), and use `IHostApplicationLifetime.ApplicationStopping` to react. In Kubernetes, flip readiness to unhealthy on SIGTERM to drain traffic first.

---

### Q25. How do you run singleton work once across instances vs scale distributable work?

**Singleton work** (a nightly batch): a **distributed lock** (only the lock holder runs it) or **leader election**, or delegate to a clustered/external scheduler. **Distributable work** (a queue): **competing consumers** — more worker instances pulling from a shared durable queue, each item processed once. (The latter needs a broker/Hangfire store, not an in-process channel.)

---

### Q26. Why is observability critical for background work?

It fails **silently** — no user gets an error response. Log start/success/failure (don't swallow exceptions), emit metrics (queue depth, failure rate, throughput), trace with `Activity`, monitor dead-letter/failed-job queues, and expose health checks — so a silently backing-up queue or dead worker is detected.

---

→ Next: [Coding.md](Coding.md)
