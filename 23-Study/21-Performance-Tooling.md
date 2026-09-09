# 21 — Performance & Tooling

## ⚡ 30-second answer

**Measure, don't guess** — intuition about bottlenecks is reliably wrong, and the hot path is
usually concentrated somewhere non-obvious. The methodology: define the goal and metric → take a
baseline → profile → change **one** thing → re-measure → stop when the goal is met. Match the
tool to the question: **`dotnet-counters`** for live triage, **`dotnet-trace`** for where CPU and
allocations go, **`dotnet-dump`**/**`dotnet-gcdump`** for heap and leaks, **BenchmarkDotNet** for
"is A faster than B". These are **EventPipe**-based, so they work identically on Linux and in
containers; PerfView uses **ETW** and goes deepest on Windows. "Premature optimization" means
avoid *speculative* micro-tuning before measuring — it does **not** mean ignore performance: get
the algorithms and architecture right early, defer the micro work. And for data-driven apps,
**check the database first** — an N+1 dwarfs any amount of allocation tuning.

---

## Core mechanics

### The triage workflow

```text
alert / user report
      ↓
dotnet-counters   ← what KIND of problem is this?
      ↓
   ┌──────────────┬──────────────────┬─────────────────┐
CPU high      heap climbing     thread pool queue    exceptions/sec
   ↓              ↓              growing               ↓
dotnet-trace  dotnet-gcdump   → STARVATION          dotnet-trace
(hot stacks)  ×2 + gcroot       (sync-over-async)   (exception events)
              (retention path)
```

`dotnet-counters` categorizes fast:

| Signal | Likely cause |
|---|---|
| Climbing heap, rising Gen 2 | leak or unbounded cache |
| High allocation rate, high % time in GC | allocation pressure |
| **Thread-pool count *and* queue growing** | **starvation — sync-over-async** |
| High CPU, low allocation | genuine compute, or a hot loop |
| High exceptions/sec | exceptions used as control flow |

### The tools

| Tool | Answers | Platform |
|---|---|---|
| **`dotnet-counters`** | "what's happening right now?" — **first triage** | all |
| **`dotnet-trace`** | "where does the time / allocation go?" (CPU sampling) | all |
| **`dotnet-gcdump`** | "what's on the heap?" — leak hunting | all |
| **`dotnet-dump`** + SOS | "what is every thread doing?" — hangs, deadlocks | all |
| **`dotnet-stack`** | quick stack snapshot | all |
| **BenchmarkDotNet** | "is A faster than B?" — micro comparison | all |
| **PerfView** (ETW) | deepest GC + allocation analysis | Windows |
| **VS Profiler** | click-to-source, tight dev loop | Windows/IDE |
| **dotTrace / dotMemory** | polished UX; **Timeline** shows async + contention | cross-platform |

```bash
dotnet-counters monitor -p <pid> --counters System.Runtime,Microsoft.AspNetCore.Hosting
dotnet-trace collect -p <pid> --profile cpu-sampling --duration 00:00:30
dotnet-gcdump collect -p <pid>          # twice, minutes apart, then diff
dotnet-dump collect -p <pid> && dotnet-dump analyze core_dump
```

**Leak hunt**: two `gcdump` snapshots minutes apart → diff to find the type that's growing → in
`dotnet-dump`, `dumpheap -stat` then **`gcroot`** to reveal the **retention path** — the
long-lived root holding it. It's nearly always a static collection, an unbounded cache, or an
event handler never unsubscribed.

**Hang or deadlock**: `dotnet-dump` → `clrthreads` and `clrstack`. Threads blocked on `.Result` are
visible directly, which is how you *prove* thread-pool starvation rather than guessing.

### BenchmarkDotNet

```csharp
[MemoryDiagnoser]
public class Bench
{
    [Params(10, 1000)] public int N;
    [GlobalSetup] public void Setup() { … }              // untimed
    [Benchmark(Baseline = true)] public string Old() => …;   // ← return the result
    [Benchmark] public string New() => …;
}
```

It handles what naive `Stopwatch` timing gets wrong: **JIT warmup** (your first calls run
unoptimized Tier 0 code — [01](01-Runtime-GC-JIT.md)), **dead-code elimination** (why you must
return results), GC noise, and statistics. Read **Mean/StdDev** (low StdDev = trustworthy),
**Ratio** vs baseline, and **Allocated/Gen0** — excess allocation is the most common .NET culprit.

**Always Release, never with a debugger attached.**

It is the **micro** tool: profile the app first to find the hot path, benchmark alternatives for
it, then re-profile to confirm the win in context.

### ETW and EventPipe

The tools sit on tracing infrastructure. **ETW** is Windows kernel-level and sees .NET **plus OS**
events — it powers PerfView and goes deepest. **EventPipe** is built into the runtime and is
**cross-platform** — it powers the `dotnet-*` tools, which is why they work identically in a Linux
container. Both consume the same events, and a `dotnet-trace` capture can be opened in PerfView.

Both are **low-overhead**, so `dotnet-counters` and `dotnet-trace` are production-safe. Writing
your own **`EventSource`** puts domain events into the same traces
([02](02-BCL-Essentials.md), [12](12-Observability.md)).

### The five recurring bottlenecks

1. **Excessive allocations** → GC pressure. `Span<T>`, `ArrayPool<T>`, `StringBuilder`; avoid
   boxing and LINQ chains on hot paths.
2. **Database** — N+1, missing indexes, over-fetching. For data apps, **look here first**; it
   usually dwarfs any CPU micro-tuning ([05](05-EFCore.md)).
3. **Blocking / sync-over-async** → thread-pool starvation.
4. **Chatty I/O** — many small round-trips where latency, not throughput, is the cost.
5. **Missing caching** — recomputing or refetching the same thing
   ([06](06-DataAccess-Caching.md)).

These are **likely suspects to recognize once profiling points somewhere** — not permission to
skip measuring.

---

## 🪤 Traps & gotchas

- **Optimizing without profiling** — you'll rewrite the elegant 2% and leave the 60% untouched.
  Every experienced answer to "how do you improve performance?" starts with measurement.
- **Benchmarking in Debug or with the debugger attached** — optimizations are off and the numbers
  are meaningless. BenchmarkDotNet warns; people ignore it.
- **Benchmarking cold code** — the first calls run Tier 0. A hand-rolled `Stopwatch` loop with no
  warmup measures the JIT.
- **A benchmark that discards its result** — the JIT eliminates the computation as dead code and
  you time an empty loop.
- **Setup inside the benchmark method** — you're timing the setup too. `[GlobalSetup]`.
- **Trusting a single run** — high StdDev means the number is noise. Look at the distribution
  before you believe a 5% improvement.
- **Changing several things at once** — you can't attribute the change, and if it got slower you
  don't know which part.
- **Micro-optimizing when the database is the bottleneck** — shaving allocations off a handler that
  spends 400ms in an N+1 query.
- **Profiling on a developer laptop and shipping to a container** with a 1-CPU limit and different
  GC mode — different bottleneck entirely.
- **Reading averages instead of percentiles** — a good mean hides the p99 that's actually paging
  someone ([12](12-Observability.md)).
- **Capturing a trace for 2 seconds under no load** — sample under **realistic** load, for a
  representative window.
- **Taking one heap snapshot to find a leak** — a leak is *growth*. You need two snapshots over
  time and a diff.
- **Finding the big type but not the root** — `dumpheap -stat` says "1.2 GB of `byte[]`", which
  isn't actionable. `gcroot` tells you what's holding them.
- **Forgetting that a dump pauses the process** — and that a dump of a large heap is a large file.
  Know the cost before you take one in production.
- **`GC.Collect()` "to fix" a memory problem** — it promotes survivors and makes the next full GC
  worse ([01](01-Runtime-GC-JIT.md)).
- **Assuming high memory is a leak** — .NET holds memory it may reuse. The question is whether the
  *live* heap grows across collections, which is exactly what `gcdump` diffing answers.
- **Treating "premature optimization is the root of all evil" as license to ignore performance** —
  the quote is about speculative micro-tuning. Algorithmic and architectural choices are the
  expensive ones to change later.

---

## ❓ Likely questions

**Q: How do you approach a performance problem?**
A: Define the goal and the metric first, take a baseline, then profile to find the actual
bottleneck rather than guessing. Change one thing, re-measure against the baseline, and stop when
the goal is met. The order matters — intuition about where time goes is reliably wrong.

**Q: Which tool for which question?**
A: `dotnet-counters` for live triage — it tells you what *kind* of problem you have.
`dotnet-trace` for where CPU or allocations go. `dotnet-gcdump`/`dotnet-dump` for heap contents,
leaks, and thread state. BenchmarkDotNet for comparing two implementations. PerfView when you need
the deepest GC analysis on Windows.

**Q: How do you find a memory leak in production?**
A: Confirm it's real with `dotnet-counters` — the heap climbing across Gen 2 collections, not just
high memory. Then take two `dotnet-gcdump` snapshots minutes apart and diff them to see which type
is growing. Then `gcroot` on an instance to get the retention path — the long-lived root holding
it, usually a static collection, an unbounded cache, or an un-unsubscribed event handler.

**Q: How do you diagnose thread-pool starvation?**
A: `dotnet-counters` shows the thread-pool thread count *and* queue length both climbing while CPU
sits low — that combination is the signature. Confirm with `dotnet-dump` and `clrthreads`: you'll
see threads blocked on `.Result` or `.Wait()`. The fix is async all the way
([01](01-Runtime-GC-JIT.md)).

**Q: Why does BenchmarkDotNet exist?**
A: Because a naive timing loop measures the wrong thing — cold Tier 0 code, dead-code elimination
of results you never use, GC noise, and no statistical treatment. It handles warmup, runs enough
iterations for confidence, prevents elimination, and reports mean, standard deviation and
allocations.

**Q: Why does `[MemoryDiagnoser]` matter as much as the timing?**
A: Because in .NET, allocation is usually the cost. Allocation itself is a pointer bump, but the
survivors create GC work, and a change that halves allocations often beats one that shaves
nanoseconds off the arithmetic.

**Q: ETW vs EventPipe?**
A: ETW is Windows kernel-level tracing that sees both .NET and OS events, and it's what PerfView
uses for the deepest analysis. EventPipe is built into the runtime and works cross-platform,
which is what makes the `dotnet-*` tools work identically in a Linux container. Both emit the same
runtime events, and a `dotnet-trace` file can be analyzed in PerfView.

**Q: Are these tools safe to run in production?**
A: `dotnet-counters` and `dotnet-trace` are low-overhead and designed for it. A full `dotnet-dump`
pauses the process and produces a file the size of the heap, so it's a deliberate action rather
than a routine one.

**Q: What are the usual bottlenecks in a .NET web app?**
A: In rough order of how often they're the real answer: the database (N+1, missing indexes,
over-fetching), missing caching, blocking calls causing thread-pool starvation, chatty I/O, and
then allocation pressure. For a data-driven app the database dominates, which is why you check it
first.

**Q: What does "premature optimization is the root of all evil" actually mean?**
A: Avoid speculative micro-tuning before you've measured — not "ignore performance". Algorithms,
data access patterns and architecture are expensive to change later and deserve thought up front;
the micro work is what you defer until a profiler points at it.

**Q: How do you know your optimization worked?**
A: Re-measure against the same baseline, under the same conditions, and confirm it in the real
application rather than only in the microbenchmark — a change that's 3× faster in isolation can be
invisible if that method is 1% of the request.

---

## 🎓 Senior Extra

- **Profile the shape of the workload, not one request.** A single trace shows one path; the
  interesting question is which of the top five endpoints consumes the fleet's CPU, and that comes
  from metrics first ([12](12-Observability.md)).
- **Latency is usually not CPU.** Most .NET web latency is waiting — database round-trips, HTTP
  calls, lock contention. That's why `dotnet-trace`'s CPU profile can look flat on a service that's
  visibly slow, and why the Timeline view in dotTrace (which shows async waits and contention) is
  often more revealing.
- **Allocation cost is about survivors, not allocations.** An object that dies in Gen 0 is nearly
  free; one that gets promoted costs a copy and eventually a Gen 2 trace. So the metric to watch is
  Gen 2 collection frequency, not the raw allocation counter.
- **Continuous profilers** (Application Insights Profiler, Datadog, Pyroscope) close the gap
  between "the alert says CPU is high" and "here is the method", without needing someone to be
  logged in while it happens.
- **Benchmark the runtime upgrade.** BenchmarkDotNet jobs can compare .NET 9 vs 10 on the same
  code — often the cheapest performance win available, and worth quantifying rather than assuming.
- **Server GC on a small container** commits far more than you'd expect from per-core heaps; DATAS
  is the modern mitigation and is on by default, but it's worth knowing the knob exists when a pod
  keeps getting OOM-killed ([01](01-Runtime-GC-JIT.md)).
- **Write your own `EventSource`** for domain events — near-zero cost when nothing listens, and it
  puts "order validated / payment called / receipt written" into the same trace as the runtime
  events, which makes a production trace legible.
- **Set a performance budget, not a target.** "p99 under 300ms at 500 rps" is testable, gates a
  release, and stops the endless-tuning failure mode that "make it faster" invites.
- **Regression-test performance in CI** by comparing trends rather than absolutes — CI machines are
  noisy, so an alert on a 20% drift is meaningful where an absolute threshold is just flaky.
- **The best optimization is usually deletion**: not making the call, not fetching the column, not
  serializing the field, caching the whole response. Reach for those before you reach for
  `Span<T>`.

→ Deeper: [`../21-Performance/`](../21-Performance/README.md)
