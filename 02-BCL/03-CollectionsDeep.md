# Collections Deep — Specialized & Concurrent

## Beyond the everyday collections

CSharpBook Chapter 07 covers the core collections (`List<T>`, `Dictionary`, `HashSet`, `Queue`, `Stack`, sorted/immutable/frozen). This file covers the **specialized** ones the BCL ships for specific needs: concurrent collections, the producer/consumer `BlockingCollection`, `OrderedDictionary<TKey,TValue>`, `BitArray`, `PriorityQueue`, and how to pick among the immutable/frozen families.

The meta-skill: matching a collection's **concurrency, ordering, and access pattern** to your workload. The wrong choice is a correctness bug (race) or a performance cliff (O(n) where O(1) was available).

---

## Concurrent collections (`System.Collections.Concurrent`)

Standard collections are **not thread-safe** — concurrent mutation corrupts them (and can crash). The concurrent collections are designed for safe lock-free or fine-grained-locked access from multiple threads:

| Type | Use |
|---|---|
| `ConcurrentDictionary<TKey,TValue>` | Thread-safe map; the workhorse |
| `ConcurrentQueue<T>` | Thread-safe FIFO |
| `ConcurrentStack<T>` | Thread-safe LIFO |
| `ConcurrentBag<T>` | Unordered, optimized for "same thread adds & removes" |
| `BlockingCollection<T>` | Producer/consumer with blocking + bounding (wraps the above) |

### `ConcurrentDictionary` — the key patterns

```csharp
var cache = new ConcurrentDictionary<string, byte[]>();

// Atomic get-or-add — the factory may run more than once under contention,
// but only one result is stored.
byte[] data = cache.GetOrAdd(key, k => LoadFromDisk(k));

// Atomic add-or-update
cache.AddOrUpdate(key, addValue, (k, existing) => Combine(existing, addValue));

// Try operations (no exceptions)
if (cache.TryGetValue(key, out var v)) { ... }
cache.TryRemove(key, out _);
```

Critical subtlety: **`GetOrAdd`'s value factory is not atomic** — under contention two threads may both run it; only one value wins. If the factory is expensive or has side effects, guard it (e.g., store a `Lazy<T>`):

```csharp
// Ensure the expensive factory runs exactly once per key:
var cache = new ConcurrentDictionary<string, Lazy<byte[]>>();
byte[] data = cache.GetOrAdd(key, k => new Lazy<byte[]>(() => LoadFromDisk(k))).Value;
```

`Count`, enumeration, and `ToArray()` take a moment-in-time snapshot and are safe but not "frozen" — the dictionary can change around them.

### `ConcurrentBag` — the niche one

`ConcurrentBag<T>` is optimized for the pattern where **each thread adds and removes its own items** (work-stealing-style). It uses thread-local storage. If one thread produces and another consumes, `ConcurrentQueue<T>` is usually better — don't reach for `ConcurrentBag` by default.

> When *do* you need a concurrent collection? Only when **multiple threads** access the same collection. Single-threaded or per-request-isolated code should use the plain (faster) collections. Don't pay for concurrency you don't have.

---

## `BlockingCollection<T>` — producer/consumer

Wraps a concurrent collection to add **blocking** (consumers wait when empty) and **bounding** (producers wait when full → back-pressure):

```csharp
var queue = new BlockingCollection<WorkItem>(boundedCapacity: 100);

// Producer
Task.Run(() => {
    foreach (var item in source) queue.Add(item);   // blocks if at capacity (back-pressure)
    queue.CompleteAdding();                           // signal "no more"
});

// Consumer(s)
foreach (var item in queue.GetConsumingEnumerable())  // blocks until item or completion
    Process(item);
```

`BlockingCollection` is the classic producer/consumer primitive. **However**, for new async code, prefer **`Channel<T>`** ([12-Threading.md](12-Threading.md)) — it's async-friendly (no thread blocking; `await ReadAsync()`), lower-overhead, and integrates with cancellation. Use `BlockingCollection` for synchronous, thread-based pipelines.

---

## `OrderedDictionary<TKey,TValue>` (.NET 9+)

A long-missing generic: a dictionary that **preserves insertion order** *and* supports O(1) key lookup *and* indexed access:

```csharp
var od = new OrderedDictionary<string, int>();
od["a"] = 1; od["b"] = 2; od["c"] = 3;
od.GetAt(0);                       // first entry by insertion order
od.Insert(1, "x", 99);             // insert at position
foreach (var (k, v) in od) { }     // enumerates in insertion order
```

Before .NET 9, you had the non-generic `System.Collections.Specialized.OrderedDictionary` (boxes, untyped) or had to combine a `List` + `Dictionary` by hand. The new generic `OrderedDictionary<TKey,TValue>` does both cleanly. Use it when **order matters and you need fast key lookup** (e.g., ordered config, LRU-ish structures).

(Note: a regular `Dictionary<TKey,TValue>` *happens* to preserve insertion order in practice today, but that is **not guaranteed** — never rely on it. Use `OrderedDictionary` when order is a requirement.)

---

## `PriorityQueue<TElement,TPriority>` (.NET 6+)

A binary-heap priority queue — dequeues the **lowest-priority** value first:

```csharp
var pq = new PriorityQueue<string, int>();
pq.Enqueue("low", 5);
pq.Enqueue("urgent", 1);
pq.Enqueue("medium", 3);
pq.Dequeue();                       // "urgent" (priority 1 = highest priority here)
pq.EnqueueRange(items);             // bulk
pq.TryPeek(out var el, out var pri);
```

O(log n) enqueue/dequeue, O(1) peek. Use a custom `IComparer<TPriority>` to reverse order or compare composite priorities. Ideal for schedulers, Dijkstra/A*, event simulation. (Also in CSharpBook Ch07 §07; here noting it's a BCL heap, **not stable** — equal priorities have no guaranteed order.)

---

## `BitArray` — compact bit storage

```csharp
var bits = new BitArray(1000);      // 1000 bits, packed (~125 bytes, not 1000 bools)
bits[42] = true;
bits.Set(43, true);
bits.And(otherBits);                 // bitwise ops across the whole array
int setCount = 0; foreach (bool b in bits) if (b) setCount++;
```

`BitArray` packs booleans into bits (8× denser than `bool[]`, which uses a byte per bool). Use it for large bitmaps, sieves, feature flags at scale, or set membership over a dense integer range. For typed bit flags on a single value, prefer a `[Flags]` enum (CSharpBook Ch03 §04).

---

## Immutable vs Frozen vs Concurrent — choosing

These three families solve different problems; people conflate them:

| Family | Mutability | Thread-safety | Best for |
|---|---|---|---|
| `Concurrent*` | mutable | safe concurrent mutation | shared state changed by many threads |
| `Immutable*` | immutable (returns new on "change") | safe (immutable) | shared state that's mostly read, occasionally rebuilt; snapshots |
| `Frozen*` | immutable, read-optimized | safe (immutable) | built once, read **very** often (lookup tables) |

```csharp
// Frozen — build once at startup, read millions of times (fastest reads)
private static readonly FrozenSet<string> Keywords =
    new[] { "if", "else", "while" }.ToFrozenSet();          // slow to build, fastest Contains

// Immutable — share safely, "modify" by producing a new instance (structural sharing for ImmutableList)
ImmutableArray<int> a = [1, 2, 3];
ImmutableArray<int> b = a.Add(4);                            // a unchanged

// Concurrent — many threads mutate the SAME instance
var counters = new ConcurrentDictionary<string, int>();
```

Decision: **mutated by many threads → `Concurrent*`. Shared, rarely changed, want value snapshots → `Immutable*`. Built once, read-heavy → `Frozen*`.** (Full treatment: CSharpBook Ch07 §09–10.)

---

## `ReadOnlyCollection<T>` and read-only views

```csharp
private readonly List<int> _items = new();
public IReadOnlyList<int> Items => _items;                    // a read-only VIEW (not a copy)
public ReadOnlyCollection<int> Snapshot => _items.AsReadOnly(); // wrapper that blocks mutation
```

`AsReadOnly()` / `IReadOnlyList<T>` expose a collection without letting callers mutate it — but they're **views**, not snapshots: the underlying list's changes are visible through them. For a true independent snapshot, copy (`.ToArray()`) or use `ImmutableArray<T>`. (Collection API design: CSharpBook Ch17 §06.)

---

## Common gotchas

### Using a non-concurrent collection across threads

`Dictionary`/`List` mutated concurrently corrupts internal state (and can throw or hang). Use `Concurrent*`, or lock, or isolate per thread.

### `GetOrAdd` factory running multiple times

It's not atomic under contention. Wrap expensive/side-effecting factories in `Lazy<T>`.

### `ConcurrentBag` as a general queue

It's tuned for same-thread add/remove. For cross-thread producer/consumer use `ConcurrentQueue`/`Channel<T>`.

### Relying on `Dictionary` insertion order

Not guaranteed. Use `OrderedDictionary<TKey,TValue>` (.NET 9+) when order is a requirement.

### `BlockingCollection` in async code

It blocks threads. Prefer `Channel<T>` for async producer/consumer to avoid thread-pool starvation ([Ch01 §08](../01-Runtime/08-Threading.md)).

### Rebuilding a `FrozenSet`/`FrozenDictionary` repeatedly

Frozen collections are **expensive to build** (they optimize for reads). Build once, reuse — never per request.

---

## Summary

- **Concurrent collections** (`ConcurrentDictionary/Queue/Stack/Bag`) are for **multi-thread** access; don't use them when single-threaded (plain ones are faster). `GetOrAdd`'s factory isn't atomic — wrap in `Lazy<T>`.
- **`BlockingCollection<T>`** is the classic blocking/bounded producer/consumer; for async, prefer **`Channel<T>`**.
- **`OrderedDictionary<TKey,TValue>`** (.NET 9+) = insertion order + O(1) lookup + indexing; don't rely on `Dictionary` order.
- **`PriorityQueue`** (binary heap, not stable), **`BitArray`** (packed bits, 8× denser than `bool[]`).
- Choose families by need: **Concurrent** (multi-thread mutation), **Immutable** (shared, snapshot), **Frozen** (build-once, read-heavy).
- Read-only views (`IReadOnlyList<T>`, `AsReadOnly()`) block mutation but aren't snapshots — copy for independence.

→ Next: [04-IO.md](04-IO.md)
