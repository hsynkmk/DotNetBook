# BenchmarkDotNet

## Measuring performance correctly

Intuition about performance is usually wrong, and naive timing (`Stopwatch` around a loop) is *misleading* — JIT warmup, dead-code elimination, GC noise, and CPU frequency scaling all distort results. **BenchmarkDotNet** is the standard .NET microbenchmarking library: it runs your code through a rigorous, statistically-sound process (warmup, multiple iterations, multiple process launches), eliminates common measurement errors, and reports accurate, comparable numbers — including memory allocation. It's how you *prove* an optimization helped (or catch a performance regression) instead of guessing.

```csharp
[MemoryDiagnoser]   // also report allocations, not just time
public class StringBenchmarks {
    private readonly string[] _parts = Enumerable.Range(0, 100).Select(i => i.ToString()).ToArray();

    [Benchmark(Baseline = true)]
    public string Concat() {
        var s = "";
        foreach (var p in _parts) s += p;        // O(n²) — allocates a new string each time
        return s;
    }

    [Benchmark]
    public string StringBuilder() {
        var sb = new System.Text.StringBuilder();
        foreach (var p in _parts) sb.Append(p);
        return sb.ToString();
    }
}
// Program.cs: BenchmarkRunner.Run<StringBenchmarks>();
```

---

## Why naive timing is wrong

Wrapping code in a `Stopwatch` loop produces numbers, but they're unreliable because:

- **JIT warmup / tiered compilation** — the first runs execute unoptimized code ([Ch01 §02](../01-Runtime/02-JIT.md)); you must warm up before measuring.
- **Dead-code elimination** — if you don't *use* the result, the JIT may delete the work entirely, "measuring" nothing.
- **GC noise** — collections during measurement add variance; you need many iterations and statistics.
- **CPU frequency scaling / other processes** — single runs are noisy; you need repeated, isolated measurement.

BenchmarkDotNet handles all of this: it runs warmup iterations, repeats measurements, launches multiple processes, **consumes returned values** (returning a value from a `[Benchmark]` method prevents dead-code elimination), and reports mean/median/standard deviation with outlier analysis. **Always use BenchmarkDotNet for microbenchmarks** — hand-rolled timing is the classic way to draw wrong conclusions.

---

## Reading the output

A run produces a table with statistics and (with `[MemoryDiagnoser]`) allocations:

```
| Method        | Mean      | Error    | StdDev   | Ratio | Gen0   | Allocated |
|-------------- |----------:|---------:|---------:|------:|-------:|----------:|
| Concat        | 12.450 us | 0.083 us | 0.077 us |  1.00 | 7.91 | 16,512 B  |
| StringBuilder |  1.120 us | 0.009 us | 0.008 us |  0.09 | 0.61 |  1,280 B  |
```

- **Mean** — average time; **Error/StdDev** — confidence/variance (low StdDev = stable result).
- **Ratio** — relative to the `Baseline` benchmark (here StringBuilder is ~11× faster).
- **Gen0/Gen1/Gen2** — GC collections per 1000 ops; **Allocated** — bytes allocated per op.

The **allocation column is often more important than time** — excess allocation drives GC pressure ([Ch01 §04](../01-Runtime/04-GCDeepDive.md)), which hurts throughput and latency. `[MemoryDiagnoser]` is almost always worth enabling.

---

## Parameterizing benchmarks

Compare across input sizes or configurations with `[Params]`:

```csharp
[Params(10, 100, 1000)]
public int N { get; set; }

[Benchmark]
public int Sum() { /* work scaled by N */ }
```

BenchmarkDotNet runs each benchmark for every parameter value, so you see how performance **scales** (revealing O(n) vs O(n²) behavior). Use `[GlobalSetup]`/`[IterationSetup]` for setup that shouldn't be timed (allocate test data once in `[GlobalSetup]`, not inside the benchmark).

---

## Diagnosers and jobs

- **Diagnosers**: `[MemoryDiagnoser]` (allocations/GC), `[DisassemblyDiagnoser]` (the JIT'd assembly), `[ThreadingDiagnoser]` (lock contention), hardware-counter diagnosers (cache misses, branch mispredictions).
- **Jobs**: run the same benchmarks across multiple runtimes/configs to compare (e.g., `[SimpleJob(RuntimeMoniker.Net90)]` vs `Net10_0`) — useful for verifying a .NET upgrade's perf impact, or `Server` vs `Workstation` GC.

```csharp
[SimpleJob(RuntimeMoniker.Net90)]
[SimpleJob(RuntimeMoniker.Net100)]
[MemoryDiagnoser]
public class CrossRuntimeBenchmarks { ... }
```

---

## Benchmarks in CI (regression detection)

You can run benchmarks in CI to **catch performance regressions** — but microbenchmarks are noisy on shared CI hardware, so compare trends/thresholds rather than absolute numbers, and run them in a controlled job (not on every PR if the runner is noisy). The goal is to notice "this got 3× slower" before it ships, complementing correctness tests with performance guardrails.

---

## Common gotchas

### Benchmarking in Debug / under the debugger

Debug builds disable optimizations; the debugger distorts timing. BenchmarkDotNet **requires Release** and refuses to run attached — always benchmark optimized Release builds.

### Not returning the result (dead-code elimination)

If a benchmark computes something but doesn't return/consume it, the JIT may delete the work. **Return** the result (BenchmarkDotNet consumes it) so the work is actually measured.

### Setup work inside the timed method

Allocating test data or doing one-time setup *inside* the benchmark inflates the measurement. Move it to `[GlobalSetup]` (untimed).

### Ignoring allocations

Time alone misses GC pressure. Enable `[MemoryDiagnoser]` — the allocation column often reveals the real cost and is the better optimization target.

### Trusting noisy/single-run numbers

High StdDev means an unreliable result (background load, too few iterations). Trust results with low error/StdDev; re-run in a quiet environment if noisy.

---

## Summary

- **BenchmarkDotNet** is the standard for **accurate .NET microbenchmarking** — it handles JIT warmup, **dead-code elimination** (by consuming returned values), GC noise, and statistics that naive `Stopwatch` timing gets wrong.
- Mark methods `[Benchmark]` (a `Baseline` for ratios), enable **`[MemoryDiagnoser]`** (allocations matter as much as time — GC pressure), and `[Params]` to see how performance **scales** across inputs.
- Read **Mean/StdDev** (low StdDev = trustworthy), **Ratio** (vs baseline), and **Allocated/Gen0**; use **jobs** to compare runtimes (e.g., .NET 9 vs 10) and **diagnosers** for deeper insight (disassembly, threading, HW counters).
- **Always benchmark Release, not attached**; move setup to `[GlobalSetup]`; **return results** to avoid elimination; optionally gate **regressions** in CI (compare trends, not noisy absolutes).

→ Next: [09-PropertyBased.md](09-PropertyBased.md)
