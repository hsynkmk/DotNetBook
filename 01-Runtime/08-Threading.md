# Threading & the Thread Pool

## What the runtime provides

The runtime gives you OS threads, but you rarely create them directly. Instead, almost all concurrency flows through the **thread pool** — a managed set of reusable worker threads that execute `Task`s, `async` continuations, timer callbacks, and I/O completions. Understanding the pool explains why `async`/`await` scales, why blocking it is dangerous, and why thread counts behave the way they do.

> CSharpBook Chapter 08 covers the **language/usage** side (async/await, Tasks, locks, channels). This file covers the **runtime internals**: the thread pool, its self-tuning, work-stealing queues, and I/O completion.

```
Your code:  Task.Run(...)   await SomethingAsync()   timer fires   socket data arrives
                  ↓                  ↓                     ↓              ↓
            ┌─────────────────────────────────────────────────────────────┐
            │                 .NET Thread Pool                              │
            │   worker threads  +  global queue  +  per-thread local queues │
            │   I/O completion handling (epoll/kqueue/IOCP)                 │
            └─────────────────────────────────────────────────────────────┘
                  ↓ schedules onto ↓
            OS threads (mapped 1:1 to managed threads)
```

---

## Why a pool?

Creating an OS thread is expensive (~1 MB stack, kernel bookkeeping, scheduler overhead). A server handling thousands of concurrent operations can't afford a thread each. The pool **reuses a small number of threads** to run a large number of short work items, amortizing thread cost. The key insight that makes this work for I/O: with `async`/`await`, a thread isn't *held* during the I/O wait — it returns to the pool and picks up other work, resuming the continuation later (see "I/O completion" below).

---

## Queues and work-stealing

The pool uses a two-level queue design for efficiency:

- A **global queue** — work submitted from outside the pool (`Task.Run`, `ThreadPool.QueueUserWorkItem`) lands here.
- **Per-thread local queues** — when a pool thread *itself* queues work (e.g., a Task creates a child Task), it goes into that thread's **local** queue (a LIFO deque). This keeps related work on the same thread (good cache locality) and avoids contending on the global queue.

When a worker's local queue is empty, it **steals** work from the tail of another thread's local queue (or the global queue). This **work-stealing** balances load across cores without central coordination — the same design used by high-performance fork/join schedulers.

```
Worker A local: [w1 w2 w3]   ← A pushes/pops its end (LIFO, cache-friendly)
Worker B local: [ ]          ← B is idle → steals w1 from A's other end (FIFO)
Global queue:   [g1 g2]      ← external submissions; any worker can take these
```

---

## Self-tuning: hill climbing

How many worker threads should the pool have? Too few underutilizes cores; too many causes context-switching overhead and memory waste. The pool **auto-tunes** the worker count using a **hill-climbing** algorithm:

- It periodically adjusts the thread count and measures **throughput** (work items completed per unit time).
- If adding a thread improved throughput, it tends to add more; if it hurt, it backs off. It "climbs the hill" toward the count that maximizes throughput.
- There's a baseline minimum (roughly the processor count) and a configurable maximum.

This is why thread counts rise under load and settle — the pool is searching for the optimal level empirically.

```csharp
ThreadPool.GetMinThreads(out int workerMin, out int ioMin);
ThreadPool.GetMaxThreads(out int workerMax, out int ioMax);
ThreadPool.SetMinThreads(workerMin: 50, completionPortMin: 50);   // raise the floor (rarely needed)
```

---

## Thread starvation — and why blocking is dangerous

Here's the critical failure mode. The pool grows slowly (hill climbing adds threads gradually — historically ~1–2 per second when starved). If you **block** pool threads (e.g., `Task.Run(() => SomeAsync().Result)` or synchronous I/O on many items), those threads sit idle-but-occupied. New work queues up. The pool *eventually* injects more threads, but slowly — so for a while, **throughput collapses**: this is **thread-pool starvation**.

```csharp
// ✗ — blocks a pool thread; do this on many items and you starve the pool
var result = SomethingAsync().Result;       // sync-over-async
Thread.Sleep(1000);                          // blocks a pool thread for a full second

// ✓ — releases the thread during the wait
var result = await SomethingAsync();
await Task.Delay(1000);
```

This is the runtime-level reason behind the "**async all the way / never block on async**" rule (CSharpBook Ch08 §17, Ch17 §05). Blocking a pool thread doesn't just block one operation — it removes a worker from a small shared pool, and the slow injection rate means the whole app stalls under load. Diagnose with `dotnet-counters` (`threadpool-queue-length` climbing, `threadpool-thread-count` maxing).

---

## I/O completion — why async scales

The pool handles two kinds of work: **CPU work** (worker threads) and **I/O completions**. For I/O, the runtime uses the OS's asynchronous I/O facilities:
- **Windows**: I/O Completion Ports (IOCP).
- **Linux**: epoll (and increasingly io_uring).
- **macOS**: kqueue.

When you `await` a network/file read, the runtime registers the operation with the OS and **returns the thread to the pool**. No thread is parked waiting. When the OS signals completion, the pool dispatches the continuation onto a worker. This is the magic of `async` I/O: **N concurrent I/O operations don't need N threads** — they need only enough threads to run the brief continuations as data arrives. A server can handle tens of thousands of concurrent connections on a handful of threads.

```
1000 concurrent HTTP calls, all awaiting responses:
  Blocking model:  needs ~1000 threads (each parked on a socket)  → doesn't scale
  Async model:     needs a handful of threads (run continuations as responses land) → scales
```

---

## Tasks and scheduling

A `Task` is a unit of work + a promise of its result. By default, Tasks run on the thread pool via the default `TaskScheduler`:
- `Task.Run(f)` queues `f` to the pool (global queue).
- `await`'d continuations are queued back to the pool (or to a captured `SynchronizationContext` — e.g., the UI thread — if present and not suppressed with `ConfigureAwait(false)`).
- The `async` state machine (CSharpBook Ch08 §02) parks at each `await` and resumes via a continuation scheduled on the pool — it does **not** hold a thread across the await.

Modern .NET also avoids allocating where it can: `ValueTask` (no Task alloc on sync completion) and .NET 10's **async state-machine box elision** for fast-path async (CSharpBook Ch11 §08) reduce the per-operation overhead so the pool churns less garbage.

---

## `Thread` vs the pool — when to make your own thread

The pool is for **short, frequent** work. For a **long-running, dedicated** task (a background loop that runs for the app's lifetime), don't occupy a pool thread — create a dedicated `Thread` or use `TaskCreationOptions.LongRunning`:

```csharp
// Long-running dedicated work — don't tie up a pool thread for the app's lifetime
var t = new Thread(BackgroundLoop) { IsBackground = true };
t.Start();

// or
Task.Factory.StartNew(BackgroundLoop, TaskCreationOptions.LongRunning);
```

For background *services* in a host, prefer `BackgroundService`/`IHostedService` ([Chapter 08](../08-BackgroundProcessing/README.md)) which integrates with lifetime and DI.

---

## Common gotchas

### Blocking pool threads → starvation

`.Result`/`.Wait()`/sync I/O on pool threads starves the pool under load (slow thread injection). Await instead. The #1 production async bug.

### `Task.Run` for I/O

`Task.Run(() => httpClient.GetAsync(...))` wastes a thread to *start* async I/O that needs no thread. Just `await httpClient.GetAsync(...)`. Use `Task.Run` only to offload **CPU-bound** work.

### Long-running work on the pool

A forever-loop via `Task.Run` permanently consumes a pool thread. Use a dedicated `Thread`/`LongRunning`/`BackgroundService`.

### Raising min threads as a "fix"

`SetMinThreads` to a high value masks starvation by pre-creating threads, but the real fix is to stop blocking. Use it only with understanding.

---

## Summary

- Most concurrency runs on the **thread pool** — reusable workers executing Tasks, continuations, timers, and I/O completions, so a few threads serve many operations.
- The pool uses a **global queue + per-thread local queues with work-stealing** for locality and load balancing, and **hill-climbing** to auto-tune the worker count by measured throughput.
- **Blocking pool threads causes starvation** (slow thread injection → throughput collapse) — the runtime reason for "never block on async."
- **Async I/O** registers with the OS (IOCP/epoll/kqueue) and **returns the thread**, so N concurrent I/Os need only a handful of threads — this is why async scales.
- Use a **dedicated thread / `BackgroundService`** for long-running work, not a pool thread; reserve `Task.Run` for CPU-bound offload.
- Usage-level async/await, locks, and channels: CSharpBook Chapter 08.

→ Next: [09-PInvokeInternals.md](09-PInvokeInternals.md)
