# Channel&lt;T&gt;

## In-process producer-consumer

`Channel<T>` is the BCL's async-first producer-consumer queue: producers write items, consumers read them, with optional bounding for back-pressure — all without blocking threads. It's the **in-process** foundation for decoupling work (an API handler hands work to a background consumer) before you reach for an out-of-process broker.

```csharp
using System.Threading.Channels;

var channel = Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(capacity: 100) {
    FullMode = BoundedChannelFullMode.Wait     // producer awaits when full → back-pressure
});

// Producer
await channel.Writer.WriteAsync(item, ct);     // awaits (no thread blocked) if full
channel.Writer.Complete();                      // signal "no more items"

// Consumer
await foreach (WorkItem item in channel.Reader.ReadAllAsync(ct))
    await ProcessAsync(item, ct);               // drains until completed
```

> CSharpBook Ch08 §13 covers `Channel<T>` mechanics in depth (reader/writer, `SingleReader`/`SingleWriter` optimizations). This file frames it as the **in-process messaging primitive** and the entry point to the messaging chapter.

---

## Bounded vs unbounded

```csharp
// Bounded — capacity limit gives BACK-PRESSURE (recommended)
Channel.CreateBounded<T>(new BoundedChannelOptions(100) { FullMode = BoundedChannelFullMode.Wait });

// Unbounded — no limit; producers never wait (risk: unbounded memory growth)
Channel.CreateUnbounded<T>();
```

- **Bounded** — a capacity cap. When full, the producer's `WriteAsync` **awaits** (async back-pressure) until the consumer drains — so a fast producer can't outrun a slow consumer and exhaust memory. The right default.
- **Unbounded** — no cap; `WriteAsync` always completes immediately. Use only when you're certain the producer can't outpace the consumer (else memory grows without limit — a leak).

`FullMode` for bounded channels: `Wait` (back-pressure), `DropOldest`/`DropNewest` (shed load), or `DropWrite`. Choose based on whether you must process everything (`Wait`) or can drop under overload (`Drop*`).

---

## Completion

```csharp
channel.Writer.Complete();                       // no more items will be written
channel.Writer.Complete(new Exception("fault")); // complete with an error (consumers observe it)

await foreach (var item in channel.Reader.ReadAllAsync(ct))  // ends when Complete() is called and drained
    Process(item);
await channel.Reader.Completion;                  // awaitable: completes when fully drained
```

`Complete()` signals the writer is done; `ReadAllAsync` finishes once the channel is completed **and** drained. This clean completion is how a consumer loop knows to stop. Forgetting to `Complete()` leaves consumers waiting forever.

---

## Why `Channel<T>` over `BlockingCollection<T>`

`Channel<T>` is the **async** producer-consumer primitive; `BlockingCollection<T>` ([Ch02 §03](../02-BCL/03-CollectionsDeep.md)) is the older **blocking** one:

| | `Channel<T>` | `BlockingCollection<T>` |
|---|---|---|
| Model | async (`await WriteAsync`/`ReadAsync`) | blocking (`Add`/`Take` block threads) |
| Thread usage | none blocked | a thread per blocked operation |
| Back-pressure | async (await when full) | blocking |
| Cancellation | `CancellationToken` throughout | partial |

In async server code, blocking threads (`BlockingCollection`) risks thread-pool starvation ([Ch01 §08](../01-Runtime/08-Threading.md)). `Channel<T>` yields the thread instead — use it for async producer-consumer.

---

## The decoupling pattern

`Channel<T>` decouples the **request** path from the **work** path. An API handler enqueues work and returns immediately; a background consumer processes it:

```csharp
// Producer side (e.g., a Minimal API endpoint) — enqueue and return fast
app.MapPost("/emails", async (EmailRequest req, Channel<EmailJob> queue, CancellationToken ct) => {
    await queue.Writer.WriteAsync(new EmailJob(req.To, req.Body), ct);
    return Results.Accepted();        // 202 — work happens in the background
});

// Consumer side — a BackgroundService drains the channel (next file)
```

This is the in-process version of message-queue decoupling: the producer doesn't wait for the work, and back-pressure (bounded channel) protects against overload. The consumer typically runs in a **hosted service** ([02-BackgroundQueues.md](02-BackgroundQueues.md), [Ch08](../08-BackgroundProcessing/README.md)).

---

## In-process vs out-of-process messaging

`Channel<T>` is **in-process** — producer and consumer are in the same app, sharing memory. Its limits define when you need a real broker:

| | `Channel<T>` (in-process) | Message broker (RabbitMQ/Kafka/Service Bus) |
|---|---|---|
| Scope | one process | across processes/services/machines |
| Durability | lost on restart/crash | persisted (survives restarts) |
| Delivery guarantee | none (in-memory) | at-least-once, acks, retries |
| Scaling | one process's threads | many consumer instances |

Use **`Channel<T>`** for in-process decoupling where losing in-flight items on restart is acceptable (background work within one app). When you need **durability** (don't lose messages on crash), **cross-service** communication, or **independent scaling** of producers/consumers, move to a **broker** (the rest of this chapter). The programming model (produce/consume) is similar; the broker adds persistence and distribution.

---

## Common gotchas

### Unbounded channel + fast producer

Unbounded + a producer outpacing the consumer = unbounded memory growth (effectively a leak). Use a **bounded** channel for back-pressure unless you're certain.

### Forgetting to `Complete()`

A consumer `ReadAllAsync` loop runs until the channel is completed. Never calling `Complete()` leaves consumers waiting forever (e.g., a worker that never shuts down). Complete on shutdown/end-of-input.

### Blocking instead of awaiting

Using `BlockingCollection`/blocking patterns in async code starves the thread pool. Use `Channel<T>` with `await`.

### Losing items on restart

`Channel<T>` is in-memory — a crash/restart drops queued items. If you can't lose work, use a durable broker (or persist + replay).

### Single vs multiple consumers

Multiple consumers reading one channel **compete** (each item goes to one consumer) — good for parallelism. If you need every consumer to see every item (fan-out/pub-sub), `Channel<T>` doesn't do that; use a broker topic or multiple channels.

---

## Summary

- **`Channel<T>`** is the BCL's async, in-process producer-consumer queue — the foundation for decoupling work within one app (enqueue + return; background consumer processes).
- Prefer **bounded** channels (`FullMode = Wait`) for async **back-pressure** (producer awaits when full) over unbounded (memory-growth risk); `Drop*` modes shed load under overload.
- **`Complete()`** signals end-of-input so consumer `ReadAllAsync` loops finish — forgetting it hangs consumers.
- Use `Channel<T>` over `BlockingCollection<T>` in async code (no blocked threads / starvation).
- It's **in-process** (no durability, lost on restart, single process) — move to a **broker** (RabbitMQ/Kafka/Service Bus) when you need durability, cross-service messaging, or independent scaling. Mechanics: CSharpBook Ch08 §13.

→ Next: [02-BackgroundQueues.md](02-BackgroundQueues.md)
