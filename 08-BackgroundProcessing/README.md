# Chapter 08 — Background Processing

> Work that happens outside a request: hosted services, worker processes, queued jobs, and scheduled tasks. From `BackgroundService` to Hangfire and Quartz.NET — and how to make background work reliable at scale.

**Prerequisites**: Chapter 03 (Hosting & DI), CSharpBook Chapter 08 (Concurrency).

**Time to read**: ~6-8 hours.

---

## Why this chapter

Most non-trivial apps need to do work that isn't tied to an incoming HTTP request: sending emails, processing a queue, running a nightly report, polling an external system, cleaning up stale data. .NET's hosting model makes this first-class through `IHostedService`/`BackgroundService`, and a rich ecosystem (Hangfire, Quartz.NET) handles persistence, scheduling, and retries. This chapter covers the spectrum from in-process background loops to durable, distributed job systems.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-HostedServices.md](01-HostedServices.md) | `IHostedService` lifecycle (`StartAsync`/`StopAsync`), registration, ordering, the host's startup/shutdown integration. |
| [02-BackgroundService.md](02-BackgroundService.md) | `BackgroundService` base class, `ExecuteAsync`, the long-running loop pattern, exception behavior (and the silent-crash trap). |
| [03-WorkerServices.md](03-WorkerServices.md) | The Worker Service template (`dotnet new worker`), running as a Windows Service / systemd daemon, headless hosts. |
| [04-QueuedBackgroundWork.md](04-QueuedBackgroundWork.md) | In-process job queue with `Channel<T>`, a hosted consumer, back-pressure, and offloading request work safely. |
| [05-ScheduledTasks.md](05-ScheduledTasks.md) | Timer-based and cron-style scheduling, `PeriodicTimer` (.NET 6+), avoiding overlap and drift. |
| [06-Hangfire.md](06-Hangfire.md) | Durable fire-and-forget, delayed, recurring jobs; storage (SQL/Redis), the dashboard, retries, idempotency. |
| [07-QuartzNet.md](07-QuartzNet.md) | Quartz.NET jobs, triggers, cron expressions, clustering for HA scheduling. |
| [08-ReliabilityAndScale.md](08-ReliabilityAndScale.md) | Graceful shutdown & `IHostApplicationLifetime`, cancellation, idempotency, exactly-once vs at-least-once, scaling out (leader election, distributed locks), the outbox pattern. |
| [Questions.md](Questions.md) | ~25 drilling questions. |
| [Coding.md](Coding.md) | Build a `BackgroundService` queue consumer; a cron-scheduled job; make a job idempotent; add graceful shutdown. |

---

## Learning objectives

After this chapter you should be able to:
- Choose between `BackgroundService`, a queued consumer, and a durable job library for a given need.
- Write a long-running background loop that shuts down gracefully and survives exceptions.
- Schedule recurring work without drift or overlap.
- Make background jobs idempotent and safe to run on multiple instances.
- Decide when in-process work suffices vs. when you need Hangfire/Quartz or a message broker (Chapter 07).

---

## How this relates to Messaging (Chapter 07)

Background processing and messaging overlap: a message-broker consumer *is* often a hosted service. The distinction this book draws:
- **Chapter 07 (Messaging)** — communication between services/processes via brokers (RabbitMQ, Kafka, Service Bus).
- **Chapter 08 (this chapter)** — running work in the background *within* a process, plus scheduling and durable in-app job systems.

They compose: a `BackgroundService` ([§02](02-BackgroundService.md)) is the typical host for a message consumer.

→ Begin: [01-HostedServices.md](01-HostedServices.md)
