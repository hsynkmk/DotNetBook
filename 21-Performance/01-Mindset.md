# Performance Mindset

## Measure, don't guess

The single most important rule of performance work: **measure, don't guess.** Developers' intuition about *where* a program spends its time is famously, reliably wrong — the bottleneck is almost never where you think. Optimizing based on a hunch wastes effort on code that doesn't matter (and often makes it more complex and buggier) while the real hot path goes untouched. The discipline is: **profile first** to find the actual bottleneck, optimize *that*, then **measure again** to confirm the change helped. Everything in this chapter — BenchmarkDotNet ([02-BenchmarkDotNet.md](02-BenchmarkDotNet.md)), the `dotnet-*` tools ([03](03-DotnetCounters.md)–[05](05-DotnetDump.md)), profilers ([06](06-PerfView.md)–[08](08-JetBrainsTools.md)) — exists to replace guessing with data.

> The memory/performance *mechanics* (GC, allocations, Span, value vs reference) are in [CSharpBook Ch09](../../CSharpBook/09-MemoryPerformance/README.md). This chapter is about the *methodology and tooling* to find and fix problems in real .NET apps.

---

## Why intuition fails

Several biases make guessing unreliable:

- **The bottleneck is concentrated** — by Amdahl's law and the typical 90/10 rule, a tiny fraction of code dominates runtime. You can't eyeball which 10% it is.
- **Modern hardware/runtime is non-intuitive** — caching, branch prediction, JIT optimization ([Ch01 §02](../01-Runtime/02-JIT.md)), and GC ([Ch01 §04](../01-Runtime/04-GCDeepDive.md)) mean "obviously slow" code may be fast and vice versa.
- **Micro-optimizations mislead** — shaving cycles off a function called rarely is pointless; the win is in the hot path or in *allocations* driving GC pressure.
- **Confirmation bias** — once you suspect a culprit, you "find" evidence for it. A profiler is impartial.

So the first move on any perf problem is **not** to start editing code — it's to **attach a profiler / run a benchmark** and let the data point you at the real hot spot.

---

## The methodology

A disciplined performance workflow:

1. **Define the goal** — what metric matters? Latency (p50/p95/p99), throughput (req/s), memory footprint, startup time, allocation rate? Optimize for the *right* metric (p99 latency ≠ average).
2. **Establish a baseline** — measure current performance under **realistic** conditions/data. You can't tell if you improved without a before number.
3. **Profile to find the bottleneck** — use the right tool ([01-Mindset.md](01-Mindset.md) → tool selection below) to locate where time/memory actually goes.
4. **Form a hypothesis and change *one* thing** — targeted at the proven bottleneck.
5. **Measure again** — confirm the change helped (and didn't regress elsewhere). Keep it only if it did.
6. **Repeat or stop** — when you've hit the goal, **stop** (further optimization adds complexity for diminishing returns).

The loop is **baseline → profile → fix → re-measure**. Skipping the measurement steps is how teams "optimize" their way into slower, more complex code.

---

## Choosing the right tool

Different problems need different tools (detailed in the rest of the chapter):

| You want to know | Tool |
|---|---|
| Is this *small* code path faster than that one? | **BenchmarkDotNet** ([02](02-BenchmarkDotNet.md)) — microbenchmarks |
| What's happening in a *live* process right now (GC, CPU, thread pool)? | **dotnet-counters** ([03](03-DotnetCounters.md)) |
| Where does CPU time / allocation go (sampled)? | **dotnet-trace** ([04](04-DotnetTrace.md)), profilers ([06](06-PerfView.md)–[08](08-JetBrainsTools.md)) |
| What's on the heap / why isn't it collected (a leak)? | **dotnet-dump** ([05](05-DotnetDump.md)), dotMemory ([08](08-JetBrainsTools.md)) |
| Deep CLR/ETW analysis (Windows) | **PerfView** ([06](06-PerfView.md)) |

Match the tool to the question: a **microbenchmark** for "which implementation is faster," **live counters** for "what's my running app doing," a **trace** for "where's the CPU going," a **dump** for "why is memory growing."

---

## Premature optimization (and its opposite)

Knuth's "premature optimization is the root of all evil" is widely quoted — and widely misused. The real point: **don't optimize *speculatively*, before measuring**, sacrificing clarity for unproven gains. It does **not** mean "ignore performance":

- **Architecture/algorithm choices** matter up front — an O(n²) algorithm or an N+1 query ([Ch05 §02](../05-EFCore/02-Querying.md)) won't be saved by micro-tuning later. Choose sound algorithms and data structures from the start.
- **Micro-optimizations** (caching a `MethodInfo`, avoiding an allocation) should wait for **profiling evidence**.

So: get the **big-picture** design and complexity right early (that's not premature), but defer **low-level tuning** until a profiler proves it's needed. The opposite failure — ignoring performance entirely until it's a crisis — is just as costly.

---

## Realistic measurement conditions

Measurements only matter if they reflect reality:

- **Release builds, not Debug** — Debug disables optimizations ([02-BenchmarkDotNet.md](02-BenchmarkDotNet.md)).
- **Representative data/load** — profiling with 10 rows when production has 10 million tells you nothing about the real hot path.
- **Production-like environment** — your fast dev machine isn't a constrained container; profile where it matters ([Ch15 §10](../15-MAUI/10-Performance.md) makes the same point for devices).
- **Warm state** — account for JIT warmup ([Ch01 §02](../01-Runtime/02-JIT.md)) and cold caches; measure steady-state unless startup *is* the concern.

---

## Common gotchas

### Optimizing without measuring

Editing code based on a hunch wastes effort on the wrong place and adds complexity/bugs. Always **profile first**, then optimize the proven bottleneck.

### No baseline

Without a before-measurement you can't prove an "optimization" helped (or that it regressed something). Establish a baseline first; re-measure after.

### Optimizing the wrong metric

Tuning average latency when p99 is the SLA, or CPU when allocations are the problem, misses the goal. Define and measure the metric that actually matters.

### Unrealistic test conditions

Profiling Debug builds, tiny data, or a fast dev box gives misleading results. Measure Release, realistic data, production-like environment, warm state.

### Never stopping

Endless optimization adds complexity for diminishing returns. Set a goal; when you hit it, **stop**.

---

## Summary

- **Measure, don't guess** — intuition about bottlenecks is reliably wrong (the hot path is concentrated and non-intuitive); **profile first**, optimize the proven bottleneck, **re-measure** to confirm.
- Follow the methodology: **define the goal/metric → baseline → profile → change one thing → re-measure → stop when the goal is met** (the mechanics are in [CSharpBook Ch09](../../CSharpBook/09-MemoryPerformance/README.md); this chapter is the tooling).
- **Match the tool to the question**: BenchmarkDotNet (micro comparisons), dotnet-counters (live process), dotnet-trace/profilers (CPU/alloc hot spots), dotnet-dump/dotMemory (heap/leaks), PerfView (deep CLR/ETW).
- "**Premature optimization**" means avoid *speculative* micro-tuning before measuring — **not** ignoring performance: get **algorithms/architecture** right early, defer **micro-optimizations** to profiling evidence.
- Measure under **realistic conditions** (Release, representative data/load, production-like environment, warm state) or the numbers mislead.

→ Next: [02-BenchmarkDotNet.md](02-BenchmarkDotNet.md)
