# Chapter 21 — Performance & Tooling — Coding Problems

Profile a sample app, find and fix regressions. Each problem has a hidden solution — attempt it first.

---

### Problem 1 — Benchmark string building

Benchmark `+=` concatenation vs `StringBuilder` vs `string.Join` for 1000 items, reporting allocations.

<details>
<summary>Solution</summary>

```csharp
[MemoryDiagnoser]
public class StringBenchmarks {
    private readonly string[] _items = Enumerable.Range(0, 1000).Select(i => i.ToString()).ToArray();

    [Benchmark(Baseline = true)]
    public string Concat() { var s = ""; foreach (var x in _items) s += x; return s; }

    [Benchmark]
    public string Builder() { var sb = new StringBuilder(); foreach (var x in _items) sb.Append(x); return sb.ToString(); }

    [Benchmark]
    public string Join() => string.Join("", _items);
}
```

`Concat` is O(n²) allocations (a new string each iteration); `StringBuilder`/`Join` are ~linear. `[MemoryDiagnoser]` shows the huge allocation gap ([02-BenchmarkDotNet.md](02-BenchmarkDotNet.md), [10-CommonWins.md](10-CommonWins.md)). Return results to avoid elimination; run Release.
</details>

---

### Problem 2 — Triage a slow live app

A running service is sluggish. What's your first command, and what do you look at?

<details>
<summary>Solution</summary>

```bash
dotnet-counters monitor -p <pid> --counters System.Runtime
```

Look at: **CPU %** (compute-bound?), **GC Heap Size / Allocation Rate / Gen2 count** (GC pressure/leak?), **ThreadPool Thread Count + Queue Length** (starvation?), **Exception Count** (failing path?). This **categorizes** the problem in seconds before escalating to a trace or dump ([03-DotnetCounters.md](03-DotnetCounters.md)).
</details>

---

### Problem 3 — Capture and analyze a CPU trace

Counters show high CPU. Find which methods are responsible.

<details>
<summary>Solution</summary>

```bash
dotnet-trace collect -p <pid> --duration 00:00:30 -o cpu.nettrace
# under realistic load
dotnet-trace convert cpu.nettrace --format speedscope    # or open in VS / PerfView
```

Analyze the **flame graph**: the widest frames (highest exclusive/self time) are the hot methods. Distinguish inclusive (self+callees) from exclusive time to find the true hot spot ([04-DotnetTrace.md](04-DotnetTrace.md)). Capture under load, not idle.
</details>

---

### Problem 4 — Find a memory leak with a dump

Memory climbs and GC doesn't reclaim it. Find what's leaking.

<details>
<summary>Solution</summary>

```bash
dotnet-dump collect -p <pid>           # ideally two, over time, to compare
dotnet-dump analyze core_<pid>
```
```
> dumpheap -stat                       # which type dominates the heap?
> dumpheap -type MyApp.CachedItem      # pick an instance address
> gcroot <address>                     # what roots it?
  -> MyApp.CacheService  (static _cache)   ← the leak: an unbounded static cache
```

`gcroot` reveals the **retention path** — here a static cache that never evicts. Fix: bound/evict the cache (or unsubscribe the event / clear the static). A .NET leak = unintended reachability ([05-DotnetDump.md](05-DotnetDump.md)).
</details>

---

### Problem 5 — Diagnose a hang

The app stops responding. Diagnose with a dump.

<details>
<summary>Solution</summary>

```bash
dotnet-dump collect -p <pid>
dotnet-dump analyze core_<pid>
```
```
> clrthreads                  # list managed threads
> clrstack                    # (per thread) what each is executing
```

Look for many threads blocked on `.Result`/`.Wait()` (**sync-over-async → thread-pool starvation**) or two threads each holding a lock the other needs (**deadlock**). The fix for the common case is async all the way ([05-DotnetDump.md](05-DotnetDump.md), [10-CommonWins.md](10-CommonWins.md)).
</details>

---

### Problem 6 — Fix sync-over-async

```csharp
public Order GetOrder(int id) => _httpClient.GetFromJsonAsync<Order>($"/orders/{id}").Result;
```

Why does this hurt under load, and what's the fix?

<details>
<summary>Solution</summary>

`.Result` **blocks a thread-pool thread** waiting on async I/O; under load this exhausts the pool (**starvation**) — requests queue, the app stalls (visible as growing thread count/queue in counters). Fix — async all the way:

```csharp
public Task<Order?> GetOrderAsync(int id, CancellationToken ct = default)
    => _httpClient.GetFromJsonAsync<Order>($"/orders/{id}", ct);
```

Never block on async with `.Result`/`.Wait()`; propagate `await` up the call chain ([10-CommonWins.md](10-CommonWins.md), [CSharpBook Ch08](../../CSharpBook/08-Concurrency/README.md)).
</details>

---

### Problem 7 — Fix an N+1 query

```csharp
var orders = await db.Orders.ToListAsync();
foreach (var o in orders)
    Console.WriteLine($"{o.Id}: {o.Customer.Name}");   // lazy-loads Customer per order
```

What's the problem and the fix?

<details>
<summary>Solution</summary>

It's an **N+1 query**: 1 query for orders + N queries (one per order) to load each `Customer` — a flood of round-trips. Fix with eager loading or a projection:

```csharp
// Eager load:
var orders = await db.Orders.Include(o => o.Customer).ToListAsync();
// Or project only what's needed (best):
var rows = await db.Orders
    .Select(o => new { o.Id, CustomerName = o.Customer.Name })
    .ToListAsync();
```

The VS **Database** profiler exposes N+1 as a flood of identical small queries ([07-VisualStudioProfiler.md](07-VisualStudioProfiler.md), [Ch05 §02](../05-EFCore/02-Querying.md)).
</details>

---

### Problem 8 — Reduce allocations in a hot path

This parses many small numeric substrings in a hot loop and allocates heavily. Reduce allocations.

<details>
<summary>Solution</summary>

```csharp
// ✗ allocates a substring per token:
int Sum(string csv) => csv.Split(',').Sum(int.Parse);

// ✓ Span-based, zero intermediate string allocations:
int Sum(ReadOnlySpan<char> csv) {
    int total = 0;
    foreach (var range in csv.Split(','))      // Span split (no string[])
        total += int.Parse(csv[range]);
    return total;
}
```

`Span<char>` slices the original string without allocating substrings or an array — a major GC-pressure win on hot paths ([10-CommonWins.md](10-CommonWins.md), [CSharpBook Ch09 §05](../../CSharpBook/09-MemoryPerformance/README.md)). Verify with allocation profiling ([04-DotnetTrace.md](04-DotnetTrace.md)).
</details>

---

### Problem 9 — Add a read-model cache

A dashboard recomputes an expensive aggregate on every request. Add caching with stampede protection.

<details>
<summary>Solution</summary>

```csharp
public class StatsService(HybridCache cache, IStatsRepo repo) {
    public ValueTask<Stats> GetAsync(CancellationToken ct) =>
        cache.GetOrCreateAsync("dashboard-stats",
            async token => await repo.ComputeExpensiveAsync(token),
            new HybridCacheEntryOptions { Expiration = TimeSpan.FromMinutes(5) }, cancellationToken: ct);
}
```

`HybridCache` caches the result and provides **stampede protection** (concurrent requests for a cold key don't all recompute) ([Ch06 §05](../06-DataAndCaching/05-HybridCache.md)). Cache only what profiling shows is hot, with sensible expiry ([10-CommonWins.md](10-CommonWins.md)).
</details>

---

### Problem 10 — Profile-driven workflow

You're told "make the orders page faster." Describe the steps you take (the methodology).

<details>
<summary>Solution</summary>

1. **Define the metric** — e.g., p95 latency of the orders page.
2. **Baseline** — measure current p95 under realistic load.
3. **Profile** — `dotnet-counters` to triage (CPU? GC? threads?), then `dotnet-trace` (CPU/alloc) and/or the VS **Database** tool (N+1?) to find the hot spot.
4. **Hypothesis + one fix** — e.g., the trace shows an N+1 query dominates → add `Include`/projection.
5. **Re-measure** — confirm p95 dropped; ensure no regression elsewhere.
6. **Repeat or stop** — if p95 meets the goal, **stop**; else profile the next hot spot.

Measure-driven, one change at a time, confirmed by re-measurement ([01-Mindset.md](01-Mindset.md)).
</details>

---

### Problem 11 — Spot the dead-code-elimination bug in a benchmark

```csharp
[Benchmark]
public void Compute() {
    var result = ExpensiveCalc(_input);   // result unused
}
```

Why is this benchmark meaningless, and how to fix it?

<details>
<summary>Solution</summary>

The result is **unused**, so the JIT may eliminate the call entirely — the benchmark "measures" nothing (near-zero time). Fix by **returning** the result (BenchmarkDotNet consumes returned values, preventing elimination):

```csharp
[Benchmark]
public int Compute() => ExpensiveCalc(_input);
```

Always return/consume results in benchmarks ([02-BenchmarkDotNet.md](02-BenchmarkDotNet.md)).
</details>

---

### Problem 12 — Choose the right tool

For each, name the tool: (a) is `Span` parsing faster than `Split` here, (b) why is memory growing in production on Linux, (c) what's my running app doing right now, (d) deep GC pause analysis on Windows.

<details>
<summary>Solution</summary>

- **(a) Is A faster than B → BenchmarkDotNet** (micro comparison) ([02](02-BenchmarkDotNet.md)).
- **(b) Memory growing on Linux → dotnet-dump** (`dumpheap`/`gcroot` to find the leak root; EventPipe-based, works on Linux) ([05](05-DotnetDump.md)).
- **(c) What's it doing now → dotnet-counters** (live triage) ([03](03-DotnetCounters.md)).
- **(d) Deep GC analysis on Windows → PerfView** (ETW, GCStats/alloc stacks) ([06](06-PerfView.md)).

Match the tool to the question and platform ([01-Mindset.md](01-Mindset.md), [09-ETW-EventPipe.md](09-ETW-EventPipe.md)).
</details>

---

### Problem 13 — Reduce exception-driven overhead

```csharp
public int Parse(string s) {
    try { return int.Parse(s); }      // throws on every invalid input (a common case here)
    catch (FormatException) { return 0; }
}
```

Counters show a high exception rate. Fix it.

<details>
<summary>Solution</summary>

Invalid input is a *common* case here, so throwing/catching per call is expensive (stack capture, etc.). Use the non-throwing `Try` pattern:

```csharp
public int Parse(string s) => int.TryParse(s, out var n) ? n : 0;
```

Reserve exceptions for genuinely exceptional cases; use `Try...`/result patterns for expected failures ([10-CommonWins.md](10-CommonWins.md), [CSharpBook Ch17](../../CSharpBook/17-BestPractices/README.md)). Exception count in counters drops.
</details>

---

### Problem 14 — Emit a custom metric and watch it live

Expose an `orders.placed` counter and view it with dotnet-counters.

<details>
<summary>Solution</summary>

```csharp
public class OrderMetrics {
    private readonly Counter<int> _placed;
    public OrderMetrics(IMeterFactory mf) =>
        _placed = mf.Create("MyApp.Orders").CreateCounter<int>("orders.placed");
    public void OrderPlaced() => _placed.Add(1);
}
```
```bash
dotnet-counters monitor -p <pid> --counters MyApp.Orders
```

The `Meter`/`Counter` ([Ch12 §05](../12-Observability/05-Metrics.md)) is published via EventPipe and shown live by dotnet-counters ([03-DotnetCounters.md](03-DotnetCounters.md), [09-ETW-EventPipe.md](09-ETW-EventPipe.md)) — no full observability backend needed for local diagnosis.
</details>

---

### Problem 15 — Avoid a false optimization

You "optimized" a method and it feels faster, but you have no numbers. Your teammate is skeptical. What do you do?

<details>
<summary>Solution</summary>

Without measurement, "feels faster" is meaningless (and the change may have regressed allocations or another path). **Prove it with data**:

1. Benchmark the old vs new implementation with BenchmarkDotNet (`[MemoryDiagnoser]`, baseline = old) — compare Mean and Allocated.
2. Or, if it's an app-level change, compare the **baseline** profile/metric to the post-change one under the same realistic load.

Keep the change only if the numbers show a real improvement in the **target metric** with no regression; otherwise revert. "Measure, don't guess" — and don't ship unmeasured optimizations ([01-Mindset.md](01-Mindset.md)).
</details>
