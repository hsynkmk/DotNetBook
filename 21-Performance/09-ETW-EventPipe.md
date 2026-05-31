# ETW and EventPipe

## The tracing infrastructure underneath

Every diagnostic tool in this chapter sits on top of a **tracing infrastructure** that emits and collects runtime events. On Windows that's **ETW (Event Tracing for Windows)**; cross-platform it's **EventPipe** (built into the .NET runtime). Understanding this layer explains *how* the tools work, *why* some are Windows-only (PerfView — [06-PerfView.md](06-PerfView.md)) and others cross-platform (`dotnet-trace` — [04-DotnetTrace.md](04-DotnetTrace.md)), and how to emit your **own** events that show up in traces. The runtime and BCL publish a rich stream of events (GC, JIT, exceptions, thread-pool, HTTP, EF Core) via **EventSource** ([Ch12 §08](../02-BCL/08-Diagnostics.md)), which these pipes carry to collectors.

```
Your code / runtime / BCL  ── emit events via ──▶  EventSource
                                                      │
                          ┌───────────────────────────┴───────────────┐
                       ETW (Windows)                            EventPipe (cross-platform)
                          │                                            │
                    PerfView, WPA, ...                    dotnet-trace/counters/dump, VS, ...
```

---

## EventSource — the source of events

**`EventSource`** ([Ch12 §08](../02-BCL/08-Diagnostics.md)) is the .NET API for emitting structured, low-overhead trace events. The runtime and framework use it extensively — there are well-known EventSources for the **GC**, **JIT**, **thread pool**, **exceptions**, **`HttpClient`**, **EF Core**, **ASP.NET Core**, and more. Each emits named events with typed payloads. The tracing pipes (ETW/EventPipe) **subscribe** to selected EventSources/keywords and route their events to collectors. This is why `dotnet-trace`/PerfView can show GC pauses, JIT activity, or HTTP calls: the runtime already publishes them via EventSource; the tool just listens.

You can write your **own** `EventSource` so your app's domain events appear in the same traces:

```csharp
[EventSource(Name = "MyApp-Orders")]
public sealed class OrderEventSource : EventSource {
    public static readonly OrderEventSource Log = new();
    [Event(1)] public void OrderPlaced(int id) => WriteEvent(1, id);
}
// OrderEventSource.Log.OrderPlaced(42);  → visible in dotnet-trace/PerfView
```

---

## ETW — Windows event tracing

**ETW** is Windows' high-performance, kernel-level tracing system — it can correlate **.NET events with OS-level events** (disk I/O, network, context switches, CPU sampling by the kernel). This breadth is why **PerfView** ([06-PerfView.md](06-PerfView.md)) (and Windows Performance Analyzer) are so powerful on Windows: they tap ETW to see the whole system, not just the managed runtime. The cost: ETW is **Windows-only** and requires Windows tracing infrastructure — so ETW-based tools don't run on Linux/macOS/containers.

---

## EventPipe — cross-platform tracing

**EventPipe** is the .NET runtime's **built-in, cross-platform** tracing mechanism — it carries EventSource events on **any** platform (Windows, Linux, macOS, containers) without OS-specific infrastructure. It's what powers the **`dotnet-*` diagnostic tools**:

- **dotnet-counters** ([03](03-DotnetCounters.md)) — subscribes to counter events via EventPipe.
- **dotnet-trace** ([04](04-DotnetTrace.md)) — collects EventPipe events (CPU samples, allocations, GC, custom EventSources) into `.nettrace`.
- **dotnet-dump** ([05](05-DotnetDump.md)) — uses the diagnostics IPC channel (related infrastructure).

Because EventPipe is in the runtime itself, these tools work **identically across platforms** — the key reason they're the go-to for **production, Linux, and container** diagnostics where ETW/PerfView can't run.

---

## ETW vs EventPipe

| | ETW (Windows) | EventPipe (cross-platform) |
|---|---|---|
| Platform | Windows only | **all** (Windows/Linux/macOS) |
| Scope | .NET **+ OS/kernel** events | .NET runtime events |
| Tools | PerfView, WPA, xperf | dotnet-trace/counters/dump, VS |
| Best for | deepest whole-system analysis on Windows | cross-platform, production, containers |

Both consume the **same `EventSource` events**; they differ in reach (ETW sees the OS too) and portability (EventPipe runs everywhere). A `dotnet-trace` capture (EventPipe) can be converted and opened in PerfView for its richer analysis — bridging the two.

---

## Why this matters in practice

- **Choosing tools**: on a **Linux container**, you must use EventPipe-based tools (`dotnet-trace/counters/dump`) — ETW/PerfView aren't options. On **Windows** with the hardest GC/whole-system problems, ETW/PerfView see more.
- **Custom diagnostics**: writing an `EventSource` ([Ch12 §08](../02-BCL/08-Diagnostics.md)) lets your domain events flow into the same traces as runtime events — correlate "order placed" with GC/HTTP activity in one trace.
- **Low overhead**: both are designed for production use (structured, buffered, sampling-friendly), so you can diagnose live systems without crippling them.

---

## Common gotchas

### Expecting ETW tools on Linux

ETW (and PerfView) are **Windows-only**. On Linux/macOS/containers, use **EventPipe**-based `dotnet-*` tools. Trying to use ETW cross-platform simply won't work.

### Not knowing events already exist

The runtime/BCL already emit GC, JIT, thread-pool, HTTP, and EF Core events via EventSource — you usually don't need custom instrumentation to diagnose runtime behavior; just subscribe (via dotnet-trace/PerfView) to the right provider/keywords.

### Custom EventSource mistakes

Hand-written `EventSource` types have rules (event IDs, `WriteEvent` argument order). Follow the pattern (or use the source-generated approach) so events are emitted correctly ([Ch12 §08](../02-BCL/08-Diagnostics.md)).

### Over-collecting

Enabling verbose providers/keywords floods the trace and adds overhead. Select only the providers/keywords you need for the investigation.

---

## Summary

- The diagnostic tools sit on a **tracing infrastructure**: **ETW** (Windows, kernel-level, sees .NET **+ OS** events) and **EventPipe** (built into the runtime, **cross-platform**). Both carry **`EventSource`** events ([Ch12 §08](../02-BCL/08-Diagnostics.md)) that the runtime/BCL emit for GC, JIT, thread pool, exceptions, HTTP, EF Core, etc.
- **ETW** powers **PerfView/WPA** (deepest **Windows** whole-system analysis); **EventPipe** powers the **`dotnet-*`** tools (`counters`/`trace`/`dump`) that work **identically cross-platform** — the go-to for **production, Linux, and containers**.
- Both consume the **same events**; ETW reaches further (OS), EventPipe runs everywhere — and a `dotnet-trace` capture can be analyzed in PerfView, bridging them.
- Write your own **`EventSource`** to surface domain events in the same traces; the runtime already emits rich events (no custom code needed to diagnose it), and both pipes are **low-overhead** for production — but select only the providers you need.

→ Next: [10-CommonWins.md](10-CommonWins.md)
