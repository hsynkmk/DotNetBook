# Profiling & Diagnostic Tools

## Looking inside a running process

Observability (logs/metrics/traces) tells you *what* and *where*; **profiling and diagnostic tools** let you look inside a live or captured process to answer *why* at a deeper level — CPU hot spots, allocations, GC behavior, thread stalls, memory leaks. .NET's cross-platform **`dotnet-*` diagnostic tools** work in production (no debugger), and complement the always-on telemetry pillars.

> These tools are covered in depth in **CSharpBook Chapter 15 §08** (profiling) and §07 (debugging/dumps). This file frames them as the **production-diagnostics** complement to observability and recaps the essentials.

---

## The diagnostic CLI tools

Install as global tools; all are cross-platform and low-overhead (production-safe):

```bash
dotnet tool install -g dotnet-counters
dotnet tool install -g dotnet-trace
dotnet tool install -g dotnet-dump
dotnet tool install -g dotnet-gcdump
```

| Tool | Purpose | Use when |
|---|---|---|
| **`dotnet-counters`** | live metrics (CPU, GC, alloc rate, thread-pool, your `Meter`s) | first triage — *which direction* is the problem? |
| **`dotnet-trace`** | CPU/event trace (`.nettrace`) | find CPU hot spots / event sequences |
| **`dotnet-gcdump`** | managed heap snapshot | hunt memory leaks (object retention) |
| **`dotnet-dump`** | full process dump + analysis (SOS) | crashes, hangs, deep state inspection |
| **`dotnet-stack`** | print managed stacks of all threads | diagnose a hang/deadlock quickly |

These read the runtime's **EventPipe** (cross-platform) — the same source the BCL's metrics/EventSource feed ([Ch02 §08](../02-BCL/08-Diagnostics.md)) — so they observe a process **without** attaching a debugger, safe to run against production.

---

## Triage workflow (telemetry → tools)

Observability and diagnostics work together:

```
1. METRICS alert (telemetry): p99 latency up, or memory climbing  → SOMETHING is wrong
2. dotnet-counters: triage — is it CPU? GC? thread-pool starvation? exceptions?  → which DIRECTION
3a. CPU-bound  → dotnet-trace → analyze hot stacks (PerfView/speedscope/VS)       → which METHOD
3b. Memory     → dotnet-gcdump (compare snapshots) → object retention paths        → what's LEAKING
3c. Hang/crash → dotnet-dump → clrstack/clrthreads/gcroot/pstacks                  → WHERE it's stuck
4. Fix, verify with a benchmark (BenchmarkDotNet), re-measure in production
```

`dotnet-counters` is the **first** tool — it tells you whether the problem is CPU, GC pressure, thread-pool starvation ([Ch01 §08](../01-Runtime/08-Threading.md)), or an exception storm, pointing you to the right deeper tool. (Full workflow + commands: CSharpBook Ch15 §08.)

---

## Production memory-leak hunt (the classic)

```bash
dotnet-gcdump collect -p <pid>      # snapshot the managed heap (twice, over time, to compare)
# open the .gcdump in Visual Studio → compare snapshots → find what's growing → retention path

# Or in a dump:
dotnet-dump collect -p <pid>
dotnet-dump analyze core_dump
  > dumpheap -stat            # objects by type/count/size — what dominates the heap
  > gcroot <address>          # WHY is this object still alive? (the rooting reference)
```

`dotnet-gcdump` (lightweight) + comparing snapshots over time reveals what's **growing** (the leak); `dumpheap -stat` + `gcroot` finds **what keeps objects alive** (an unintended root — a static event, a never-evicting cache — [Ch01 §04](../01-Runtime/04-GCDeepDive.md), CSharpBook Ch09 §13). This is the standard production memory-leak investigation.

---

## CPU profiling

```bash
dotnet-trace collect -p <pid> --profile cpu-sampling   # low-overhead CPU sampling
# → .nettrace, view in PerfView (Windows), Visual Studio, or convert for speedscope.app
```

`dotnet-trace` samples call stacks to show where CPU time goes (flame-graph-style in the viewers), revealing hot methods. **PerfView** (deep, free, Windows/ETW) and Visual Studio's profiler give richer analysis (CPU stacks, allocation stacks, GC stats). Use these when metrics show a CPU-bound problem and you need to find the hot path. (PerfView/VS profiler details: CSharpBook Ch15 §08.)

---

## Continuous profiling

Beyond on-demand profiling, **continuous profilers** run always-on in production at low overhead, capturing periodic CPU/allocation profiles so you can investigate *past* incidents (not just reproduce them live):

- **Azure Application Insights Profiler** / **Continuous Profiler** — sampled profiles in production.
- **Datadog / Grafana Pyroscope / etc.** — continuous profiling platforms.

Continuous profiling is the production complement to on-demand `dotnet-trace`: it's already running when an incident happens, so you can see what the CPU was doing during the spike retroactively. Increasingly part of a full observability setup alongside logs/metrics/traces.

---

## Profiling vs observability — complementary

| | Observability (logs/metrics/traces) | Profiling/diagnostics |
|---|---|---|
| Always on | yes (continuous) | on-demand (or continuous profiler) |
| Granularity | request/operation level | method/allocation/thread level |
| Answers | what/where/which service | why (CPU/memory/GC/threads) at code level |
| Tools | OTel + backend | `dotnet-*`, PerfView, VS, continuous profiler |

Telemetry detects and localizes (this route is slow, this service, memory is climbing); profiling/diagnostics drill into the *code-level cause* (this method burns CPU, these objects leak, this thread is blocked). You need both: observability for the broad, always-on view; diagnostics for deep root-cause. **Measure, don't guess** (CSharpBook Ch15 §08) — profile to find the real hot path before optimizing.

---

## Common gotchas

### Optimizing without profiling

Guessing the hot path wastes effort on the wrong code. Use `dotnet-counters` → `dotnet-trace` to find the *actual* bottleneck first.

### Heavy profiling in production

Some modes (full instrumentation) add overhead. Prefer low-overhead **sampling** (`dotnet-trace`, `dotnet-counters`, continuous profilers) for production; reserve heavy instrumentation for dev.

### Profiling a Debug build

Debug builds aren't optimized — numbers are meaningless. Profile/benchmark in **Release** (CSharpBook Ch15 §08, Ch16 §07).

### Confusing diagnostics with observability

Tools like `dotnet-dump` are on-demand deep dives, not a substitute for always-on telemetry. Use telemetry to *know* there's a problem, tools to *root-cause* it.

### Ignoring runtime metrics as leak/saturation signals

`dotnet-counters` GC and thread-pool metrics surface leaks and starvation early — watch them, don't wait for a crash.

---

## Summary

- **Profiling/diagnostic tools** look inside a live/captured process to answer *why* at the code level (CPU, allocations, GC, threads, leaks) — complementing the always-on observability pillars.
- The cross-platform **`dotnet-*` tools** (production-safe, EventPipe-based): **`dotnet-counters`** (live metrics — first triage), **`dotnet-trace`** (CPU/event traces → hot stacks), **`dotnet-gcdump`** (heap snapshots → leaks), **`dotnet-dump`**/**`dotnet-stack`** (crashes/hangs/state).
- Workflow: metrics alert → `dotnet-counters` triage (CPU/GC/thread-pool/exceptions) → the right deep tool (`-trace`/`-gcdump`/`-dump`) → fix → verify. Leak hunt = `gcdump` snapshots + `gcroot`.
- **PerfView/VS profiler** for deep CPU/allocation analysis; **continuous profilers** (App Insights/Datadog/Pyroscope) for always-on production profiling.
- Profiling (code-level *why*) complements observability (operation-level *what/where*) — **measure, don't guess**; profile in Release. Full coverage: CSharpBook Ch15 §08.

→ Next: [10-HealthChecks.md](10-HealthChecks.md)
