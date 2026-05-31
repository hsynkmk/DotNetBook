# Chapter 21 — Performance & Tooling — Q & A

---

### Q1. What's the cardinal rule of performance work?

**Measure, don't guess.** Intuition about bottlenecks is reliably wrong (the hot path is concentrated and non-intuitive). Profile first, optimize the proven bottleneck, then re-measure to confirm.

---

### Q2. What's the performance methodology?

Define the goal/metric → establish a **baseline** → **profile** to find the bottleneck → change **one** thing → **re-measure** → stop when the goal is met. Skipping baseline/re-measure is how teams "optimize" into slower, more complex code.

---

### Q3. What does "premature optimization is the root of all evil" actually mean?

Avoid **speculative** micro-tuning before measuring (sacrificing clarity for unproven gains). It does **not** mean ignore performance — get **algorithms/architecture** right up front (an O(n²) algorithm or N+1 query won't be saved by micro-tuning); defer **micro-optimizations** to profiling evidence.

---

### Q4. Why must you measure under realistic conditions?

Debug builds disable optimizations, tiny data hides the real hot path, a fast dev box isn't a constrained container, and cold state misrepresents steady-state. Profile **Release**, realistic data/load, production-like environment, warm state — or the numbers mislead.

---

### Q5. When use BenchmarkDotNet vs a profiler?

**BenchmarkDotNet** for *micro* comparisons ("is A faster than B?") of code you've already proven matters. A **profiler** (dotnet-trace/VS/PerfView) to find *which* code is the bottleneck in a whole app. Profile to find the hot path, benchmark to choose its implementation, re-profile to confirm.

---

### Q6. Why is naive `Stopwatch` benchmarking misleading?

JIT warmup/tiered compilation, dead-code elimination (unused results deleted), GC noise, and CPU scaling distort it. BenchmarkDotNet handles warmup, repeated/multi-process runs, consumes results, and reports statistics + allocations.

---

### Q7. What is dotnet-counters and when is it the first tool?

A low-overhead CLI showing **live runtime counters** (GC heap/collections, allocation rate, CPU, exceptions, thread-pool count/queue, lock contention) from a running process. It's the **triage** first step — categorize the problem (GC? threads? CPU? exceptions?) before escalating to a trace or dump. Production-safe, cross-platform, no restart.

---

### Q8. How do you spot thread-pool starvation with counters?

Growing **ThreadPool Thread Count** + **Queue Length** (work waiting for threads) while the app is unresponsive. The classic cause is **sync-over-async** (`.Result`/`.Wait()` blocking pool threads). Confirm with a dump showing threads blocked on `.Result`.

---

### Q9. What does dotnet-trace capture and how do you analyze it?

A **trace** of EventPipe events — CPU samples (statistical, low overhead), allocation/GC/exception/custom EventSource events — of a running process. Analyze the `.nettrace` in Visual Studio, PerfView, or Speedscope; read the **flame graph** (widest frames = optimize) and distinguish inclusive vs exclusive time. Capture under realistic load.

---

### Q10. CPU profiling vs allocation profiling — when each?

**CPU profiling** when time/compute is the bottleneck (high CPU). **Allocation profiling** when GC pressure is the problem (high allocation rate/Gen2 from counters). Match the mode to the symptom; don't CPU-profile a memory problem.

---

### Q11. What is dotnet-dump for?

Capturing a **memory snapshot** (heap, thread stacks) and analyzing with SOS commands (`dumpheap -stat`, `dumpobj`, `gcroot`, `clrstack`) — for **leaks** (what's on the heap, why it's rooted) and **hangs/crashes** (what threads are doing). A snapshot, not a time profile.

---

### Q12. How do you find a leak with a dump?

`dumpheap -stat` to see which type is eating the heap, then **`gcroot <address>`** to find the **retention path** — the long-lived root (static collection, un-unsubscribed event, unbounded cache) keeping objects alive. A .NET "leak" is unintended reachability, not unfreed memory. Compare two dumps over time to see what's growing.

---

### Q13. What is PerfView and why is it powerful?

Microsoft's free, **Windows-only**, **ETW**-based profiler — the **deepest** .NET analysis, especially for **GC/allocation** (GCStats, GC Heap Alloc Stacks) and CPU stacks with **grouping/folding** to collapse framework noise. Steep/dated UI, but unmatched depth; complementary to the cross-platform `dotnet-*` tools.

---

### Q14. What's the Visual Studio profiler best at?

Convenience: in-IDE profiling with **click-to-source**, bundling CPU Usage, .NET Object Allocation, Memory Usage (snapshot diffing + paths-to-root), and the **Database** tool (surfaces N+1/slow queries instantly). Ideal for everyday dev-time profiling on Windows. Profile Release for accurate numbers.

---

### Q15. What do JetBrains dotTrace/dotMemory add?

Polished, cross-platform (Rider) dedicated profilers. **dotTrace**'s **Timeline** mode visualizes threads/async/lock-contention/starvation over time; **dotMemory** excels at leaks (snapshot diff, graphical **retention paths**, automatic inspections like un-unsubscribed event handlers). Best for Rider teams and deep memory analysis.

---

### Q16. ETW vs EventPipe?

**ETW** (Windows-only, kernel-level) sees .NET **+ OS** events — powers PerfView/WPA (deepest Windows analysis). **EventPipe** (built into the runtime, **cross-platform**) carries the same EventSource events everywhere — powers the `dotnet-*` tools (production/Linux/containers). Both consume the same `EventSource` events.

---

### Q17. What is EventSource's role?

It's the .NET API for emitting structured, low-overhead trace events; the runtime/BCL emit GC/JIT/thread-pool/HTTP/EF Core events via it, which ETW/EventPipe carry to tools. You can write your own `EventSource` so domain events appear in the same traces.

---

### Q18. What's the #1 .NET performance issue, and how do you fix it?

**Excessive allocations** in hot paths driving GC pressure. Fix by allocating less: `Span<T>`/`Memory<T>` (slice without copying), `ArrayPool<T>` (reuse buffers), `StringBuilder`/`string.Join` (not `+=` in loops), avoiding LINQ/boxing on hot paths. Find offenders with allocation profiling.

---

### Q19. What's the most common whole-app bottleneck in data apps?

**Database access** — especially **N+1 queries** (1 + N round-trips from lazy loading per item; fix with `Include`/projection) and over-fetching (project to DTOs, `AsNoTracking` for reads, add indexes). DB wins usually dwarf CPU/allocation micro-tuning. The VS Database profiler exposes N+1 instantly.

---

### Q20. What causes "the app is slow/unresponsive under load" most often?

**Thread-pool starvation** from **sync-over-async** (`.Result`/`.Wait()` blocking pool threads). Counters show growing thread count + queue; a dump shows threads blocked on `.Result`. Fix: async all the way.

---

### Q21. When should you cache, and what's the risk?

Cache things profiling shows are **repeatedly recomputed/refetched** (in-memory for hot per-instance data, distributed/HybridCache for shared/scaled, output caching for responses). The risk is **stale data** from wrong invalidation and memory growth — cache deliberately with correct expiry/invalidation.

---

### Q22. Why are exceptions-as-control-flow a performance problem?

Throwing/catching is expensive (stack capture, etc.) — high exception count shows in counters. For *expected* cases use `Try...` patterns/result types, reserving exceptions for genuinely exceptional conditions.

---

### Q23. Which tool for a Linux container vs deep Windows GC analysis?

**Linux/container** → EventPipe-based `dotnet-trace/counters/dump` (ETW/PerfView aren't available). **Deepest Windows GC/allocation** → PerfView (ETW). Match the tool to the platform and the question.

---

### Q24. After applying a fix, what must you do?

**Re-measure** against the baseline to confirm the change actually improved the target metric (and didn't regress elsewhere). An optimization that doesn't move the metric should be reverted — keep only proven wins, then stop when the goal is met.
