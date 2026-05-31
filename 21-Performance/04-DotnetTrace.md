# dotnet-trace

## Where does the time (and allocation) go?

Once `dotnet-counters` ([03-DotnetCounters.md](03-DotnetCounters.md)) tells you *what kind* of problem you have (CPU, allocations), **`dotnet-trace`** tells you *where* — it captures a **trace** of a running process (CPU samples, allocation events, and other EventPipe events — [09-ETW-EventPipe.md](09-ETW-EventPipe.md)) that you analyze to find the **hot methods** and **call stacks** consuming time or memory. It's the cross-platform CPU/allocation profiler that needs no IDE and works on production processes, including in containers.

```bash
dotnet-trace collect --process-id 1234                                   # default CPU profile
dotnet-trace collect -p 1234 --duration 00:00:30 -o trace.nettrace        # 30s capture
dotnet-trace collect -p 1234 --providers Microsoft-DotNETCore-SampleProfiler
```

---

## What it captures

`dotnet-trace` records **EventPipe** events ([09-ETW-EventPipe.md](09-ETW-EventPipe.md)) — the cross-platform tracing infrastructure. The most common profile is **CPU sampling**: it periodically samples the call stacks of all threads, building a statistical picture of where CPU time is spent. You can also capture:

- **Allocation events** — what's allocating and where (drives GC pressure — [10-CommonWins.md](10-CommonWins.md)).
- **GC events** — collection timing/pauses.
- **Exception events**, **JIT events**, **thread-pool events**, and **custom EventSource** events ([Ch12 §08](../02-BCL/08-Diagnostics.md)).
- **Provider presets** (`gc-verbose`, `cpu-sampling`, etc.) select what to record.

CPU sampling is statistical (low overhead) — methods that appear in many samples are where the CPU time goes.

---

## Capturing a trace

```bash
# Find the process, then capture for a representative window under load:
dotnet-trace ps
dotnet-trace collect -p 1234 --duration 00:00:30 -o app.nettrace
```

Key practices:
- **Capture under realistic load** ([01-Mindset.md](01-Mindset.md)) — an idle app's trace is useless; profile while it's doing the work you care about.
- **Capture for a representative window** — long enough to gather statistically meaningful samples, short enough to be manageable.
- **Low overhead** — sampling is light, so it's safe on production (unlike instrumenting profilers).

---

## Analyzing the trace

The output `.nettrace` (or converted formats) is analyzed in a viewer:

- **Visual Studio** ([07-VisualStudioProfiler.md](07-VisualStudioProfiler.md)) — open `.nettrace` directly for a call-tree/flame view.
- **PerfView** ([06-PerfView.md](06-PerfView.md)) — the most powerful analysis (convert with `dotnet-trace convert --format speedscope` or open ETL).
- **Speedscope** (web) — `--format speedscope` produces a flame-graph view in the browser.

You look for:
- **Hot methods** — those appearing in the most CPU samples (the bottleneck).
- **Call stacks / flame graph** — *who calls* the hot method and how time aggregates up the tree (inclusive vs exclusive time).
- **Unexpected costs** — a serialization/regex/LINQ call dominating, or a method called far more than expected.

The flame graph is the key artifact: width = time, so the widest frames are where to optimize. A method you "knew" was fine often turns out to dominate — exactly why you measure ([01-Mindset.md](01-Mindset.md)).

---

## CPU vs allocation profiling

Two common modes:

- **CPU profiling** — find where *time* goes. Use when CPU is the bottleneck (high CPU %, slow compute).
- **Allocation profiling** — find what *allocates* (and the stacks doing it). Use when GC pressure is the problem (high allocation rate/Gen2 from counters — [03-DotnetCounters.md](03-DotnetCounters.md)). Reducing allocations in hot paths is one of the biggest .NET wins ([10-CommonWins.md](10-CommonWins.md)).

Pick the mode matching the symptom your counters revealed — don't CPU-profile a memory problem.

---

## dotnet-trace vs profilers vs dump

- **dotnet-trace** — cross-platform, scriptable, production-safe **sampling**; analyze offline. Great for servers/containers where you can't attach an IDE.
- **Visual Studio / JetBrains profilers** ([07](07-VisualStudioProfiler.md), [08](08-JetBrainsTools.md)) — richer interactive UI, but typically used on dev machines.
- **PerfView** ([06](06-PerfView.md)) — deepest CLR analysis (Windows).
- **dotnet-dump** ([05](05-DotnetDump.md)) — a *snapshot of the heap* (what's in memory now), not a time profile — for leaks, not CPU.

For "where's my CPU/allocation going on a Linux container," `dotnet-trace` is the go-to.

---

## Common gotchas

### Tracing an idle app

A trace captured while the app isn't doing the relevant work shows nothing useful. Capture **under realistic load**, during the slow operation.

### CPU-profiling a memory problem (or vice versa)

Match the mode to the symptom: CPU sampling for compute hot spots, allocation profiling for GC pressure, a dump for leaks. The counters ([03](03-DotnetCounters.md)) tell you which.

### Reading exclusive vs inclusive time wrong

A method with high *inclusive* time may just be calling expensive children; *exclusive* (self) time is where the method itself spends time. Distinguish them when finding the real hot spot.

### Capturing too little / too much

Too short a window gives few samples (noisy); too long produces huge files. Capture a representative window (e.g., tens of seconds under load).

### Forgetting it's statistical

Sampling is probabilistic — rare methods may not appear, and exact counts aren't precise. It's excellent for finding *dominant* hot spots, less so for precise call counts (use instrumentation/BenchmarkDotNet for that).

---

## Summary

- **`dotnet-trace`** captures a **trace** (CPU samples, allocation/GC/exception/EventSource events via **EventPipe** — [09](09-ETW-EventPipe.md)) of a running process to find **hot methods and call stacks** — the cross-platform, production-safe CPU/allocation profiler (no IDE needed).
- **CPU sampling** (statistical, low overhead) shows where *time* goes; **allocation profiling** shows what *allocates* — pick the mode matching the symptom your **counters** revealed ([03](03-DotnetCounters.md)).
- **Capture under realistic load** for a representative window; analyze the `.nettrace` in **Visual Studio**, **PerfView**, or **Speedscope** — read the **flame graph** (widest frames = optimize there) and distinguish **inclusive vs exclusive** time.
- It's a **time profile** (vs a heap snapshot — use **dotnet-dump** ([05](05-DotnetDump.md)) for leaks); go-to for "where's the CPU/allocation going" on servers/containers.

→ Next: [05-DotnetDump.md](05-DotnetDump.md)
