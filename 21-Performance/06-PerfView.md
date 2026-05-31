# PerfView

## The deepest CLR analysis (Windows)

**PerfView** is Microsoft's free, Windows-only profiling tool — the most **powerful** for in-depth CLR analysis. Built on **ETW (Event Tracing for Windows)** ([09-ETW-EventPipe.md](09-ETW-EventPipe.md)), it captures extremely detailed traces (CPU, allocations, GC, JIT, exceptions, thread-pool, locks, and arbitrary ETW events) and provides deep analysis views — most famously **CPU stacks** with grouping/folding and **GC heap analysis**. It has a steep, dated UI, but for serious .NET performance investigation on Windows (especially GC and allocation problems), nothing matches its depth. It's the tool the .NET team itself uses.

```
PerfView → Collect → Run (your app) → Stop → open the resulting .etl
Views: CPU Stacks | GC Heap Alloc | GCStats | Events | ...
```

---

## What makes it powerful

PerfView's strength is **detail + analysis**:

- **CPU Stacks view** — sampled CPU with powerful **grouping and folding** (collapse framework noise, group by module/namespace) so you can see *your* code's cost without drowning in BCL frames. Inclusive/exclusive time, callers/callees, flame graphs.
- **GC analysis** — **GCStats** (pause times, frequency, generation sizes, gen budgets) and **GC Heap Alloc Stacks** (which call stacks allocate what) — the best view into GC pressure ([Ch01 §04](../01-Runtime/04-GCDeepDive.md)) and the allocations driving it.
- **Heap snapshots** — capture and analyze the managed heap, including **reference chains** to find what roots objects (leaks — [05-DotnetDump.md](05-DotnetDump.md)).
- **Any ETW event** — JIT, exceptions, lock contention, disk/file I/O, and custom `EventSource` events ([Ch12 §08](../02-BCL/08-Diagnostics.md)) — correlate across the whole system.

Its grouping/folding is the differentiator: large traces become readable because you can fold away noise and focus on your hot code.

---

## Typical workflows

- **CPU bottleneck**: Collect → reproduce the load → Stop → open **CPU Stacks**, fold framework frames, find the widest (highest exclusive-time) frames in your code.
- **GC/allocation problem**: open **GC Heap Alloc Stacks** to see which call stacks allocate the most (the targets for reduction — [10-CommonWins.md](10-CommonWins.md)), and **GCStats** to see pause impact.
- **Memory leak**: take heap snapshots over time, compare, and follow reference chains to the rooting object.
- **Startup / JIT analysis**: ETW JIT events show what's being compiled and the startup cost.

---

## PerfView vs the cross-platform tools

| | PerfView | dotnet-counters/trace/dump |
|---|---|---|
| Platform | **Windows only** (ETW) | **cross-platform** (EventPipe) |
| Depth | **deepest** (full ETW, rich analysis) | good, more focused |
| UI | powerful but dated/steep | CLI (analyze elsewhere) |
| Best for | serious GC/CPU/allocation deep-dives on Windows | live triage, container/Linux profiling, scripting |

They're complementary: use **dotnet-counters/trace/dump** ([03](03-DotnetCounters.md)–[05](05-DotnetDump.md)) for cross-platform, production, and container scenarios (and quick triage); reach for **PerfView** when you need the deepest analysis on Windows, especially for GC and allocation investigations. A `dotnet-trace` `.nettrace` can even be converted and opened in PerfView for its superior analysis views.

---

## The learning-curve trade-off

PerfView's UI is notoriously unintuitive (it exposes a lot, with terse terminology). The investment pays off for serious performance work — its analysis capabilities (grouping/folding, GC views, reference chains) are unmatched in the free .NET tooling. For casual profiling, the **Visual Studio profiler** ([07-VisualStudioProfiler.md](07-VisualStudioProfiler.md)) or **JetBrains tools** ([08-JetBrainsTools.md](08-JetBrainsTools.md)) are friendlier; for the hardest problems, PerfView is worth learning.

---

## Common gotchas

### Expecting a friendly UI

PerfView's UI is powerful but steep and dated. Budget time to learn its concepts (grouping/folding, the stack views); the depth rewards the effort.

### Not folding framework frames

Raw CPU stacks are dominated by BCL/framework frames. Use **grouping/folding** to collapse noise and surface *your* hot code — this is the key skill.

### Using it cross-platform

PerfView is **Windows-only** (ETW). On Linux/containers, use **dotnet-trace/dump/counters** (EventPipe) and optionally convert traces to analyze in PerfView on Windows.

### Inclusive vs exclusive confusion

As with any profiler, distinguish inclusive (self + callees) from exclusive (self) time when finding the real hot spot ([04-DotnetTrace.md](04-DotnetTrace.md)).

### Huge traces

Detailed ETW collection produces large traces. Capture focused windows under load, and fold/filter during analysis.

---

## Summary

- **PerfView** is Microsoft's free, **Windows-only**, ETW-based ([09](09-ETW-EventPipe.md)) profiler — the **deepest** .NET performance analysis tool, especially for **GC and allocation** problems (GCStats, GC Heap Alloc Stacks) and **CPU stacks** with powerful **grouping/folding**.
- Its differentiator is **detail + analysis**: fold framework noise to see your hot code, analyze GC pauses and allocation stacks, follow heap reference chains to find leak roots, and capture any ETW/`EventSource` event.
- It's **complementary** to the cross-platform `dotnet-*` tools: use those for triage/Linux/containers ([03](03-DotnetCounters.md)–[05](05-DotnetDump.md)); use PerfView for the hardest Windows deep-dives (a `.nettrace` can be converted into it).
- The **UI is steep/dated** — invest in learning grouping/folding; for casual profiling, Visual Studio ([07](07-VisualStudioProfiler.md)) / JetBrains ([08](08-JetBrainsTools.md)) are friendlier.

→ Next: [07-VisualStudioProfiler.md](07-VisualStudioProfiler.md)
