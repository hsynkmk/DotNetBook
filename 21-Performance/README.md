# Chapter 21 — Performance & Tooling

> Measuring, finding, and fixing performance problems. BenchmarkDotNet for the small, dotnet-counters / dotnet-trace / dotnet-dump for the live.

**Prerequisites**: CSharpBook Chapter 09 (Memory & Performance).

**Time to read**: ~6-8 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-Mindset.md](01-Mindset.md) | "Measure, don't guess." Profile-first methodology, common bias traps. |
| [02-BenchmarkDotNet.md](02-BenchmarkDotNet.md) | Setup, [MemoryDiagnoser], [Params], baselines, common pitfalls (dead-code elimination, JIT-warmup). |
| [03-DotnetCounters.md](03-DotnetCounters.md) | Live perf counters: GC, exceptions, thread pool, custom Meter-based counters. |
| [04-DotnetTrace.md](04-DotnetTrace.md) | CPU sampling, allocation traces, EventPipe events. |
| [05-DotnetDump.md](05-DotnetDump.md) | Heap dumps, SOS-style analysis (dumpheap, dumpobj, gcroot). |
| [06-PerfView.md](06-PerfView.md) | The veteran Windows-only profiler — still the most powerful for in-depth CLR analysis. |
| [07-VisualStudioProfiler.md](07-VisualStudioProfiler.md) | The integrated experience. |
| [08-JetBrainsTools.md](08-JetBrainsTools.md) | dotMemory, dotTrace — when they shine. |
| [09-ETW-EventPipe.md](09-ETW-EventPipe.md) | Lower-level tracing infrastructure on Windows and cross-platform. |
| [10-CommonWins.md](10-CommonWins.md) | The "usual suspects": string concat, async-over-sync, eager EF queries, allocations in hot loops. |
| [Questions.md](Questions.md) | Drilling. |
| [Coding.md](Coding.md) | Profile a sample app, find and fix a regression. |

→ Begin: [01-Mindset.md](01-Mindset.md)
