# JetBrains dotTrace and dotMemory

## Polished, dedicated profilers

JetBrains' **dotTrace** (performance/CPU profiler) and **dotMemory** (memory profiler) are commercial, polished, dedicated profiling tools — part of the JetBrains .NET suite (and integrated with Rider and Visual Studio). They overlap with the Visual Studio profiler ([07-VisualStudioProfiler.md](07-VisualStudioProfiler.md)) but are often praised for their **UX, analysis depth, and memory-investigation workflows** — and they're **cross-platform** (Rider runs on Windows/macOS/Linux), unlike PerfView. For teams using Rider, or wanting best-in-class memory analysis, they're a strong choice.

---

## dotTrace — performance profiling

**dotTrace** profiles where time goes, with several profiling modes that trade detail vs overhead:

| Mode | What |
|---|---|
| **Sampling** | low-overhead statistical CPU (like dotnet-trace — [04](04-DotnetTrace.md)) — the default |
| **Tracing** | precise call counts/timing (higher overhead) |
| **Line-by-line** | per-line timing within methods (highest detail) |
| **Timeline** | time-correlated view of threads, async, I/O, GC, lock contention |

The **Timeline** mode is a standout — it shows activity over time across threads, making **async/await** flow, **lock contention**, **thread-pool starvation**, and I/O waits visible in a way pure call-tree profilers don't. Combined with strong call-tree/flame-graph views and source navigation, dotTrace is excellent for diagnosing both CPU hot spots and *concurrency*/latency issues ([CSharpBook Ch08](../../CSharpBook/08-Concurrency/README.md)).

---

## dotMemory — memory profiling

**dotMemory** is widely regarded as best-in-class for .NET memory analysis. It captures **heap snapshots** and provides rich analysis to find leaks and excessive allocations:

- **Snapshot comparison** — diff two snapshots to see exactly what grew between them (the leak-hunting workflow — [05-DotnetDump.md](05-DotnetDump.md)).
- **Retention paths** — for any object, see *what's keeping it alive* (the chain to the GC root — the `gcroot` equivalent, but graphical) — the key to finding *why* something leaks.
- **Automatic inspections** — it flags common problems: objects rooted by **event handlers** (un-unsubscribed — [CSharpBook Ch05](../../CSharpBook/05-DelegatesEvents/README.md)), strings/duplicates, sparse arrays, leaked disposables.
- **Allocation profiling** — what allocates and where, to cut GC pressure ([10-CommonWins.md](10-CommonWins.md)).

Its graphical retention-path analysis and automatic leak inspections make memory investigations much faster than command-line SOS ([05-DotnetDump.md](05-DotnetDump.md)) for most developers.

---

## How they compare

| | JetBrains dotTrace/dotMemory | VS Profiler ([07](07-VisualStudioProfiler.md)) | PerfView ([06](06-PerfView.md)) | dotnet-* ([03](03-DotnetCounters.md)–[05](05-DotnetDump.md)) |
|---|---|---|---|---|
| Cost | commercial | included with VS | free | free |
| Platform | cross-platform (Rider) | Windows/VS | Windows | cross-platform |
| UX | polished, strong memory UX | convenient, in-IDE | powerful but steep | CLI |
| Best for | Rider users, deep memory analysis | everyday VS dev | deepest GC/CLR (Windows) | production/containers, triage |

All find the same problems; the choice is about **ecosystem and workflow**. Rider users get dotTrace/dotMemory integrated; VS users have the built-in profiler; everyone has the free cross-platform `dotnet-*` tools for production. dotMemory's memory-analysis UX is a frequent reason teams pay for it.

---

## When they shine

- **Memory leaks / retention puzzles** — dotMemory's graphical retention paths + automatic inspections quickly answer "why is this still alive?" (faster than SOS for most).
- **Concurrency/latency issues** — dotTrace's **Timeline** mode exposes async waits, lock contention, and thread-pool starvation visually.
- **Rider-based teams** — integrated profiling without leaving the IDE, cross-platform.

For production/Linux/container triage you'll still use the **`dotnet-*`** tools ([03](03-DotnetCounters.md)–[05](05-DotnetDump.md)); JetBrains tools shine at the developer's desk for deep, comfortable analysis.

---

## Common gotchas

### Profiling Debug builds

Same rule as every profiler: measure **Release** for accurate numbers ([01-Mindset.md](01-Mindset.md)).

### Choosing the wrong dotTrace mode

Sampling (low overhead) for general CPU; tracing/line-by-line (high overhead) only when you need precise counts. High-overhead modes distort timing — don't use them for steady-state measurement.

### Expecting them on production servers

These are developer-desk tools (commercial, GUI). For production/containers, use the cross-platform **`dotnet-*`** tools and analyze offline.

### Ignoring automatic inspections

dotMemory's automatic inspections flag common leaks (event handlers, etc.) — don't skip them; they often point straight at the problem.

### Snapshot timing for leaks

As with any heap analysis, compare **two snapshots over time** to find what's growing — a single snapshot shows state, not growth ([05-DotnetDump.md](05-DotnetDump.md)).

---

## Summary

- **dotTrace** (CPU/performance) and **dotMemory** (memory) are JetBrains' polished, **cross-platform** (Rider) dedicated profilers — overlapping with the VS profiler but praised for UX and depth, especially **memory analysis**.
- **dotTrace** offers sampling/tracing/line-by-line plus a standout **Timeline** mode that visualizes threads, **async**, **lock contention**, and **thread-pool starvation** over time — great for concurrency/latency issues.
- **dotMemory** excels at leaks: **snapshot comparison**, graphical **retention paths** (why an object is alive), and **automatic inspections** (e.g., un-unsubscribed event handlers) — faster than CLI SOS for most ([05-DotnetDump.md](05-DotnetDump.md)).
- All profilers find the same problems; choose by **ecosystem/workflow** (Rider → JetBrains, VS → built-in, deepest Windows GC → PerfView, production/Linux → `dotnet-*`). Profile **Release**, pick the right overhead mode, and diff snapshots over time.

→ Next: [09-ETW-EventPipe.md](09-ETW-EventPipe.md)
