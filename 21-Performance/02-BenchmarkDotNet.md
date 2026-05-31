# BenchmarkDotNet

## Rigorous microbenchmarking

**BenchmarkDotNet** is the standard for measuring whether one piece of code is faster than another — it runs your methods through a statistically-sound process (warmup, multiple iterations, multiple process launches) that eliminates the errors naive `Stopwatch` timing produces. It's the tool for the question *"is implementation A faster than B?"* — comparing alternatives, validating an optimization, or catching a microbenchmark-level regression. (For finding the bottleneck in a *whole app*, you profile — [04-DotnetTrace.md](04-DotnetTrace.md); BenchmarkDotNet is for *micro* comparisons once you know what to measure.)

```csharp
[MemoryDiagnoser]
public class JoinBenchmarks {
    private readonly string[] _items = Enumerable.Range(0, 1000).Select(i => i.ToString()).ToArray();

    [Benchmark(Baseline = true)]
    public string Concat() { var s = ""; foreach (var i in _items) s += i; return s; }

    [Benchmark]
    public string Join() => string.Join("", _items);
}
// BenchmarkRunner.Run<JoinBenchmarks>();  // in Release
```

> BenchmarkDotNet is covered in depth in **[Ch17 §08](../17-Testing/08-BenchmarkDotNet.md)** (the Testing chapter). This section recaps it in the performance-tooling context.

---

## Why not just use `Stopwatch`?

Naive timing is misleading because of (recap — [01-Mindset.md](01-Mindset.md), [Ch17 §08](../17-Testing/08-BenchmarkDotNet.md)):

- **JIT warmup / tiered compilation** — first runs are unoptimized ([Ch01 §02](../01-Runtime/02-JIT.md)); you must warm up.
- **Dead-code elimination** — if the result isn't used, the JIT may delete the work entirely.
- **GC noise & CPU scaling** — single runs are noisy; you need many iterations and statistics.

BenchmarkDotNet handles all of this — warmup, repeated/multi-process measurement, consuming returned values (so work isn't eliminated), and outlier-aware statistics. It's the *only* reliable way to do .NET microbenchmarks.

---

## The essentials

- **`[Benchmark]`** marks a method; **`Baseline = true`** makes others report a **Ratio** relative to it.
- **`[MemoryDiagnoser]`** — reports **allocations** and **GC collections** per op (often more important than time — allocations drive GC pressure — [Ch01 §04](../01-Runtime/04-GCDeepDive.md)).
- **`[Params(...)]`** — runs each benchmark across input values to reveal **scaling** (O(n) vs O(n²)).
- **`[GlobalSetup]`** — one-time setup that isn't timed (allocate test data here, not in the benchmark).
- **Jobs** (`[SimpleJob(RuntimeMoniker.Net90)]` vs `Net100`) — compare across runtimes (e.g., a .NET upgrade's perf impact).

```csharp
[Params(100, 10_000)] public int N { get; set; }
[GlobalSetup] public void Setup() { /* build data of size N, untimed */ }
```

---

## Reading the output

```
| Method | Mean      | Error    | StdDev   | Ratio | Gen0   | Allocated |
|--------|----------:|---------:|---------:|------:|-------:|----------:|
| Concat | 850.2 us  | 6.1 us   | 5.7 us   |  1.00 | 250.0  | 524,288 B |
| Join   |  12.4 us  | 0.09 us  | 0.08 us  |  0.01 |   3.1  |   8,192 B |
```

- **Mean** + **Error/StdDev** — average and confidence (low StdDev = trustworthy).
- **Ratio** — relative to baseline (Join is ~70× faster here).
- **Gen0 / Allocated** — GC pressure and bytes/op (Concat's O(n²) string building allocates ~64× more).

The **Allocated** column frequently tells the real story — excessive allocation, not raw CPU, is the common .NET performance culprit (it drives GC — [10-CommonWins.md](10-CommonWins.md)).

---

## Where it fits in the workflow

BenchmarkDotNet sits at the **micro** end of the methodology ([01-Mindset.md](01-Mindset.md)):

1. **Profile the whole app** ([04-DotnetTrace.md](04-DotnetTrace.md)) to find the hot method.
2. **Benchmark alternatives** for that method with BenchmarkDotNet to pick the fastest/lowest-allocating implementation.
3. **Re-profile** to confirm the app-level improvement.

Don't benchmark code that isn't a proven hot spot — a faster implementation of a rarely-called method is wasted effort ([01-Mindset.md](01-Mindset.md)). Use BenchmarkDotNet to *decide between implementations* of code you've already proven matters.

---

## Common gotchas

### Benchmarking in Debug / attached

Debug disables optimizations; the debugger distorts timing. BenchmarkDotNet **requires Release** and refuses to run attached. Always Release, never attached.

### Not returning the result

Unused results get eliminated by the JIT, "measuring" nothing. **Return** the value (BenchmarkDotNet consumes it).

### Setup inside the timed method

One-time setup/allocation inside the benchmark inflates results. Move it to `[GlobalSetup]`.

### Ignoring allocations

Time alone misses GC pressure. Enable `[MemoryDiagnoser]` — the allocation column is often the real target ([10-CommonWins.md](10-CommonWins.md)).

### Benchmarking non-hot-path code

Optimizing a microbenchmark of code that's not a real bottleneck wastes effort. Profile first; benchmark only proven hot paths.

---

## Summary

- **BenchmarkDotNet** rigorously answers *"is A faster than B?"* — handling JIT warmup, **dead-code elimination**, GC noise, and statistics that naive `Stopwatch` timing gets wrong (depth in [Ch17 §08](../17-Testing/08-BenchmarkDotNet.md)).
- Mark methods `[Benchmark]` (with a `Baseline` for ratios), enable **`[MemoryDiagnoser]`** (allocations matter as much as time), use **`[Params]`** for scaling and `[GlobalSetup]` for untimed setup; **jobs** compare runtimes.
- Read **Mean/StdDev** (trust low StdDev), **Ratio** (vs baseline), and **Allocated/Gen0** — excess **allocation** is the common .NET culprit ([10-CommonWins.md](10-CommonWins.md)).
- It's the **micro** tool: **profile the app first** to find the hot path, **benchmark alternatives** for it, then **re-profile** — always **Release, not attached**, returning results, with setup in `[GlobalSetup]`.

→ Next: [03-DotnetCounters.md](03-DotnetCounters.md)
