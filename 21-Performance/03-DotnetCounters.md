# dotnet-counters

## Live metrics from a running process

**`dotnet-counters`** is a command-line tool that shows **live performance counters** from a running .NET process — GC activity, exception rate, thread-pool state, heap size, JIT stats, and any custom `Meter`-based counters ([Ch12 §05](../12-Observability/05-Metrics.md)). It's the **first tool to reach for** when a live app is misbehaving ("it's slow / using too much memory / pegging CPU") because it gives an immediate, low-overhead, near-real-time view of *what the runtime is doing* — without attaching a debugger or stopping the process. It answers "what's happening *right now*?" before you dig deeper with a trace ([04-DotnetTrace.md](04-DotnetTrace.md)) or dump ([05-DotnetDump.md](05-DotnetDump.md)).

```bash
dotnet-counters monitor --process-id 1234                          # live dashboard
dotnet-counters monitor -p 1234 --counters System.Runtime,Microsoft.AspNetCore.Hosting
dotnet-counters collect -p 1234 --format csv -o counters.csv       # record to a file
```

---

## What it shows

`dotnet-counters monitor` displays a live-updating table. The `System.Runtime` provider exposes the most useful runtime counters:

| Counter | Tells you |
|---|---|
| **CPU Usage (%)** | how busy the process is |
| **GC Heap Size** | managed memory in use (rising steadily → possible leak — [05-DotnetDump.md](05-DotnetDump.md)) |
| **Gen 0/1/2 GC Count** | collection frequency (frequent Gen2 → memory pressure) |
| **Allocation Rate** | bytes allocated/sec (high → GC pressure, a top culprit — [10-CommonWins.md](10-CommonWins.md)) |
| **Exception Count** | exceptions/sec (high → exceptions used for control flow / a failing path) |
| **ThreadPool Thread Count** | growing → thread-pool starvation (sync-over-async — [10-CommonWins.md](10-CommonWins.md)) |
| **ThreadPool Queue Length** | work waiting for threads (starvation signal) |
| **Monitor Lock Contention Count** | lock contention ([CSharpBook Ch08](../../CSharpBook/08-Concurrency/README.md)) |

Reading these quickly narrows the problem: high allocation rate + frequent Gen2 → a memory/GC issue; growing thread-pool count + queue → starvation; high exception count → an error path or exceptions-as-control-flow.

---

## Diagnosing common symptoms

`dotnet-counters` maps symptoms to likely causes fast:

- **Memory climbing** → watch **GC Heap Size** and **Allocation Rate**. Steady growth that GC doesn't reclaim suggests a **leak** (objects rooted somewhere — investigate with a dump — [05-DotnetDump.md](05-DotnetDump.md)). A high allocation rate suggests churn (optimize hot allocations — [10-CommonWins.md](10-CommonWins.md)).
- **App unresponsive / requests queuing** → watch **ThreadPool Thread Count** and **Queue Length**. Climbing both = **thread-pool starvation**, classically from **sync-over-async** (`.Result`/`.Wait()` blocking pool threads — [10-CommonWins.md](10-CommonWins.md)).
- **High CPU** → confirm with the **CPU %** counter, then capture a **CPU trace** ([04-DotnetTrace.md](04-DotnetTrace.md)) to find which methods.
- **Errors/latency** → **Exception Count** spiking points to a failing path or exceptions misused for flow control (expensive).

It tells you the *category* of problem; you then use the deeper tool (trace/dump) to find the *exact* cause.

---

## Custom metrics

`dotnet-counters` shows any counters published via the `Meter` API ([Ch12 §05](../12-Observability/05-Metrics.md)) — so your app's own business metrics appear alongside runtime ones:

```bash
dotnet-counters monitor -p 1234 --counters MyApp.Orders   # your custom Meter
```

If you've instrumented an `orders.placed` counter or a queue-depth gauge, you can watch it live without a full observability backend — handy for local diagnosis and quick production checks.

---

## Why it's a good first tool

- **Low overhead** — it reads counters via EventPipe ([09-ETW-EventPipe.md](09-ETW-EventPipe.md)), safe to run on production processes.
- **No restart / no code change** — attach to an already-running PID.
- **Cross-platform** — works on Windows, Linux, macOS, containers.
- **Immediate signal** — seconds to see whether it's GC, threads, CPU, or exceptions.

It's the triage step: get the *shape* of the problem cheaply, then escalate to a trace (CPU/allocation hot spots) or dump (heap contents) for the detail.

---

## Common gotchas

### Jumping to a profiler before triage

Capturing a full trace/dump first is heavyweight. Start with `dotnet-counters` to *categorize* the problem (GC? threads? CPU? exceptions?), then use the targeted tool.

### Misreading climbing memory as always a leak

A high allocation *rate* (churn) differs from a *leak* (unreclaimed growth). Watch whether GC reclaims it; if heap grows and stays, it's a leak (use a dump — [05-DotnetDump.md](05-DotnetDump.md)).

### Ignoring thread-pool counters

Unresponsiveness is often **thread-pool starvation** (growing thread count + queue), not CPU. Check the thread-pool counters, not just CPU.

### Not collecting over time

A single snapshot misses trends. Use `monitor` to watch live, or `collect` to record over a period for analysis.

---

## Summary

- **`dotnet-counters`** shows **live runtime counters** (GC heap/collections, allocation rate, CPU, exceptions, **thread-pool count/queue**, lock contention) plus custom `Meter` counters — the **low-overhead first tool** for a misbehaving live process (no restart, cross-platform, production-safe).
- It **categorizes** the problem fast: climbing heap/allocation → GC/leak; growing thread-pool count+queue → **starvation** (often sync-over-async); high CPU → capture a trace; spiking exceptions → a failing path.
- It reads via **EventPipe** ([09-ETW-EventPipe.md](09-ETW-EventPipe.md)) and shows your app's **custom metrics** ([Ch12 §05](../12-Observability/05-Metrics.md)) live.
- Use it for **triage**, then escalate to **dotnet-trace** (CPU/alloc hot spots — [04](04-DotnetTrace.md)) or **dotnet-dump** (heap/leaks — [05](05-DotnetDump.md)) for the exact cause.

→ Next: [04-DotnetTrace.md](04-DotnetTrace.md)
