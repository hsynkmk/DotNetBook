# Threading Primitives (Beyond Tasks)

## The lower-level threading toolkit

CSharpBook Chapter 08 covers async/await, `Task`, locks, `Interlocked`, `SemaphoreSlim`, and concurrency patterns in depth, and [Ch01 §08](../01-Runtime/08-Threading.md) covers the thread pool internals. This file fills in the **BCL primitives those build on or sit beside**: `Channel<T>` (the modern producer/consumer), `AsyncLocal<T>`/`ExecutionContext` (ambient flow across async), `Volatile`/memory barriers, `ThreadLocal<T>`, and timers.

---

## `Channel<T>` — the modern async producer/consumer

`Channel<T>` is the async-first replacement for `BlockingCollection<T>`: producers write, consumers read, with optional bounding (back-pressure) — all **without blocking threads** (`await` instead of block):

```csharp
using System.Threading.Channels;

// Bounded channel — producers wait (asynchronously) when full → back-pressure
var channel = Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(capacity: 100) {
    FullMode = BoundedChannelFullMode.Wait,
    SingleReader = false, SingleWriter = false,    // optimize if you know the topology
});

// Producer
await channel.Writer.WriteAsync(item, ct);     // awaits if full (no thread blocked)
channel.Writer.Complete();                      // signal "no more items"

// Consumer — drains until completed
await foreach (WorkItem item in channel.Reader.ReadAllAsync(ct))
    await ProcessAsync(item);
```

Why `Channel<T>` over `BlockingCollection<T>`:
- **Async** — `WriteAsync`/`ReadAsync` yield the thread instead of blocking it (no thread-pool starvation — [Ch01 §08](../01-Runtime/08-Threading.md)).
- **Bounded back-pressure** — a slow consumer naturally throttles a fast producer.
- **Lower overhead**, integrates with `CancellationToken` and `IAsyncEnumerable`.

`Channel<T>` is the backbone of in-process pipelines and the foundation for background-work queues ([Ch08 Background Processing](../08-BackgroundProcessing/README.md)). Use **unbounded** channels only when you trust the producer not to outrun the consumer (otherwise unbounded memory growth). (Also CSharpBook Ch08 §13.)

---

## `AsyncLocal<T>` & `ExecutionContext` — ambient data across async

A `ThreadStatic` field doesn't survive an `await` (the continuation may resume on a different thread). **`AsyncLocal<T>`** stores ambient data that **flows with the logical async call chain**, surviving thread hops:

```csharp
private static readonly AsyncLocal<string?> CorrelationId = new();

async Task HandleRequest(string id) {
    CorrelationId.Value = id;                 // set at the top
    await Step1Async();                        // flows through awaits...
    await Step2Async();                        // ...even across thread changes
}

async Task Step2Async() {
    await Task.Delay(10);
    Log($"[{CorrelationId.Value}] done");      // still sees the id
}
```

This is how the framework flows **`Activity.Current`** (the current trace span — [08-Diagnostics.md](08-Diagnostics.md)), the logging scope, and `CultureInfo` across async operations. The mechanism underneath is **`ExecutionContext`** — a snapshot of ambient state (including `AsyncLocal` values) that the runtime captures at each `await` and restores on the continuation.

```csharp
// Suppress flow when you deliberately don't want ambient state to propagate (rare):
using (ExecutionContext.SuppressFlow()) {
    _ = Task.Run(BackgroundWork);    // won't inherit AsyncLocal values from here
}
```

**Gotchas**: `AsyncLocal` flows *downward* (to async operations started after setting it) — a value set in a child doesn't propagate back to the parent. And setting it has a cost (it dirties the `ExecutionContext`), so use it for genuine cross-cutting context (correlation IDs, trace context), not as a general variable-passing mechanism.

---

## Memory model: `Volatile`, barriers, and `Interlocked`

On multicore CPUs, threads can observe memory writes out of order due to CPU/compiler reordering and caching. The .NET **memory model** defines what's guaranteed. Three tools:

```csharp
// Volatile — prevent reordering around a single field access
private bool _stop;
public void Stop() => Volatile.Write(ref _stop, true);
void Loop() { while (!Volatile.Read(ref _stop)) DoWork(); }   // sees the write reliably

// Interlocked — atomic read-modify-write (no lock needed)
Interlocked.Increment(ref _counter);
Interlocked.Add(ref _total, n);
int old = Interlocked.CompareExchange(ref _state, newVal, comparand);  // CAS — lock-free algorithms

// Full fence (rarely needed directly)
Thread.MemoryBarrier();
```

- **`Volatile.Read/Write`** stops the compiler/CPU from reordering that access — use for simple flags shared across threads (a plain `bool` field read in a loop may be cached and never see another thread's write).
- **`Interlocked`** does atomic operations (increment, add, exchange, **compare-and-swap**) without a lock — the basis of lock-free counters and algorithms.
- For anything beyond a single flag/counter, prefer a **`lock`** (clearer and correct) over hand-rolled volatile/barrier code. The memory model is subtle; don't out-clever it. (CSharpBook Ch08 §11, §16.)

---

## `ThreadLocal<T>` & `[ThreadStatic]`

Per-thread storage (each thread gets its own value) — for non-async, thread-bound state:

```csharp
// ThreadLocal<T> — lazy per-thread value with a factory
private static readonly ThreadLocal<Random> Rng = new(() => new Random());
int r = Rng.Value!.Next();        // each thread gets its own Random

[ThreadStatic] private static StringBuilder? _scratch;   // per-thread field (no initializer support!)
```

`ThreadLocal<T>` (lazy, has a factory, disposable) is friendlier than the raw `[ThreadStatic]` attribute (no initializer — it runs once per type, not per thread, so initialize lazily). **Caution**: per-thread state does **not** flow across `await` (use `AsyncLocal<T>` for that) and can pin large objects on pool threads. Use it for genuinely thread-bound scratch state.

---

## Timers

```csharp
// PeriodicTimer — async, the modern choice for periodic loops (.NET 6+)
using var timer = new PeriodicTimer(TimeSpan.FromSeconds(5));
while (await timer.WaitForNextTickAsync(ct))     // no overlap, no drift accumulation, cancellable
    await PollAsync();

// System.Threading.Timer — callback-based (older; callbacks can overlap)
var t = new Timer(_ => DoWork(), null, dueTime: 0, period: 5000);
```

**`PeriodicTimer`** is the modern primitive for "do X every N seconds" — it's async (no thread blocked), naturally prevents overlapping ticks (you control the loop), and respects cancellation. Prefer it over the callback-based `Timer` (whose callbacks can re-enter/overlap if the work takes longer than the period). For scheduled background work, use a `BackgroundService` with a `PeriodicTimer` ([Ch08](../08-BackgroundProcessing/README.md)). `TimeProvider.CreateTimer` ([06-DateTimeAndTime.md](06-DateTimeAndTime.md)) makes timers testable.

---

## Synchronization primitives (quick map)

| Primitive | Use |
|---|---|
| `lock` / `Lock` (.NET 9+) | mutual exclusion (the default — CSharpBook Ch08 §09) |
| `SemaphoreSlim` | async-compatible throttling (`await WaitAsync`) |
| `ReaderWriterLockSlim` | many readers / few writers |
| `Interlocked` | atomic counters / CAS (lock-free) |
| `Barrier` | phase synchronization of N participants |
| `CountdownEvent` | wait for N operations to complete |
| `ManualResetEventSlim` / `SemaphoreSlim` | signaling |

For **async** code, use `SemaphoreSlim.WaitAsync` (not blocking primitives) to avoid thread-pool starvation. The `lock`/`Lock` and `Monitor` story, plus `SemaphoreSlim` throttling, is in CSharpBook Ch08 §09–10.

---

## Common gotchas

### Plain field read in a spin loop

A non-`volatile` flag may be cached by a thread and never observe another thread's write → infinite loop. Use `Volatile.Read`/`Write` or a proper synchronization primitive.

### `ThreadStatic`/`ThreadLocal` across `await`

Per-thread state doesn't flow across `await` (continuation may run on another thread). Use `AsyncLocal<T>` for async-flowing context.

### Unbounded `Channel<T>`

An unbounded channel with a fast producer and slow consumer grows memory without limit. Use a **bounded** channel for back-pressure unless you're certain.

### Blocking primitives in async code

`Semaphore.Wait()`, `ManualResetEvent.WaitOne()`, `BlockingCollection.Take()` block threads → starvation. Use async equivalents (`SemaphoreSlim.WaitAsync`, `Channel`).

### Hand-rolled lock-free code

Volatile/barrier/CAS algorithms are extremely subtle. Unless you've measured a real need and understand the memory model, use a `lock`.

### Overlapping `Timer` callbacks

Callback `Timer` can fire again before the previous callback finishes. Use `PeriodicTimer` (you control the loop) or guard re-entrancy.

---

## Summary

- **`Channel<T>`** is the modern async producer/consumer (bounded for back-pressure) — prefer it over `BlockingCollection<T>`; it underpins in-process queues ([Ch08](../08-BackgroundProcessing/README.md)).
- **`AsyncLocal<T>`** flows ambient data across `await` (via **`ExecutionContext`**) — how `Activity.Current`, logging scopes, and culture propagate; `ThreadStatic`/`ThreadLocal` do **not** flow across awaits.
- **`Volatile`** prevents reordering of a single field access; **`Interlocked`** does atomic ops/CAS (lock-free) — but prefer a `lock` for anything non-trivial.
- **`PeriodicTimer`** is the modern, non-overlapping, async periodic timer; use async sync primitives (`SemaphoreSlim.WaitAsync`) in async code.
- The memory model is subtle — don't hand-roll lock-free code without a measured need. Async/await and locks: CSharpBook Chapter 08.

→ Next: [13-MemoryPrimitives.md](13-MemoryPrimitives.md)
