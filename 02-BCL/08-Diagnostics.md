# Diagnostics — Tracing, Metrics, Activities

## The observability primitives in the BCL

Long before you add a logging library or an APM vendor, the BCL ships the primitives that all of them build on: **`Activity`/`ActivitySource`** (distributed tracing), **`Meter`** (metrics), **`EventSource`** (high-performance structured events), plus `Stopwatch`, `Process`, `Debug`, and `Trace`. These are **vendor-neutral** and **OpenTelemetry-compatible** — instrument with them and any backend (OpenTelemetry, Application Insights, Prometheus, Jaeger) can consume the data.

> Chapter 12 (Observability) covers the full pipeline (OpenTelemetry, exporters, Serilog, dashboards). This file covers the **BCL building blocks** those rest on — understand these and the observability chapter is mostly configuration.

```
Your code instruments with BCL primitives:
   ActivitySource  → spans/traces (the "where did time go across services" story)
   Meter           → metrics (counters, gauges, histograms)
   ILogger         → logs (structured)   [Ch12]
   EventSource     → low-overhead structured events (ETW/EventPipe)
        ↓ collected by ↓
   OpenTelemetry SDK / dotnet-trace / dotnet-counters / APM vendor
```

---

## `Activity` & `ActivitySource` — distributed tracing

An **`Activity`** represents a unit of work (an operation) with a start/stop time, a unique ID, a link to its parent, and tags. Activities chain across method calls **and across services** (via propagated trace context), producing a **distributed trace** — the timeline of a request as it flows through your system.

```csharp
using System.Diagnostics;

// Define a source once (typically static per library/component)
private static readonly ActivitySource Source = new("MyCompany.OrderService", "1.0.0");

public async Task<Order> PlaceOrderAsync(OrderRequest req) {
    using Activity? activity = Source.StartActivity("PlaceOrder");   // start a span
    activity?.SetTag("order.customer", req.CustomerId);
    activity?.SetTag("order.itemCount", req.Items.Count);

    try {
        var order = await ProcessAsync(req);                          // child activities nest automatically
        activity?.SetTag("order.id", order.Id);
        activity?.SetStatus(ActivityStatusCode.Ok);
        return order;
    } catch (Exception ex) {
        activity?.SetStatus(ActivityStatusCode.Error, ex.Message);    // mark the span failed
        throw;
    }
}
```

Key points:
- `StartActivity` returns `null` if **no listener** is subscribed — so instrumentation is **near-zero cost when nobody's collecting** (the `?.` calls no-op). You can instrument liberally.
- Activities **nest**: a child `StartActivity` automatically links to the current one (via `Activity.Current`, flowed by `AsyncLocal` — see [12-Threading.md](12-Threading.md)), building the trace tree.
- **Tags** (`SetTag`) attach structured attributes; **events** (`AddEvent`) attach timestamped points; **status** marks success/failure.
- This *is* the OpenTelemetry tracing API surface — `Activity` = an OTel "span", `ActivitySource` = an OTel "tracer". OTel's .NET SDK just listens to your `ActivitySource`s and exports them.

### Trace context propagation

Across services (HTTP/gRPC/messaging), the **W3C Trace Context** (`traceparent` header) carries the trace ID and parent span ID so spans from different processes join one trace. `HttpClient` and ASP.NET Core propagate it automatically when tracing is enabled — your downstream service's activities become children of the caller's. (Wired up in [Ch12](../12-Observability/README.md).)

---

## `Meter` — metrics (`System.Diagnostics.Metrics`)

A **`Meter`** creates instruments that record numeric measurements over time — counters, gauges, histograms — the metrics half of observability:

```csharp
using System.Diagnostics.Metrics;

private static readonly Meter Meter = new("MyCompany.OrderService", "1.0.0");

// Counter — monotonically increasing (e.g., total orders)
private static readonly Counter<long> OrdersPlaced =
    Meter.CreateCounter<long>("orders.placed", unit: "{orders}");

// Histogram — distribution of values (e.g., request durations)
private static readonly Histogram<double> OrderDuration =
    Meter.CreateHistogram<double>("orders.duration", unit: "ms");

// ObservableGauge — sampled on demand (e.g., current queue depth)
private static readonly ObservableGauge<int> QueueDepth =
    Meter.CreateObservableGauge("orders.queue.depth", () => _queue.Count);

public async Task PlaceAsync(Order o) {
    var sw = Stopwatch.StartNew();
    await ProcessAsync(o);
    OrdersPlaced.Add(1, new KeyValuePair<string, object?>("region", o.Region));  // with a dimension/tag
    OrderDuration.Record(sw.Elapsed.TotalMilliseconds);
}
```

Instrument types:
- **`Counter<T>`** — sum that only goes up (requests, errors, bytes sent).
- **`UpDownCounter<T>`** — can go up or down (active connections, items in flight).
- **`Histogram<T>`** — record a distribution (latencies, sizes) → percentiles.
- **`ObservableCounter/Gauge/UpDownCounter`** — sampled via a callback (memory usage, queue depth).

Like activities, metrics cost ~nothing if no listener subscribes. **Tags/dimensions** (the `KeyValuePair` args) let you slice metrics by region, status, route, etc. This is the OpenTelemetry **metrics** API — the OTel SDK or `dotnet-counters` listens and exports.

```bash
dotnet-counters monitor -p <pid> MyCompany.OrderService   # watch your custom Meter live
```

---

## `EventSource` — high-performance structured events

`EventSource` emits **strongly-typed, low-overhead structured events** consumed by ETW (Windows), EventPipe (cross-platform), and `dotnet-trace`. The runtime and BCL use it heavily (GC, JIT, thread pool, `HttpClient` all emit EventSource events — which is what `dotnet-counters`/`-trace` read).

```csharp
[EventSource(Name = "MyCompany-OrderService")]
public sealed class OrderEventSource : EventSource {
    public static readonly OrderEventSource Log = new();

    [Event(1, Level = EventLevel.Informational)]
    public void OrderPlaced(int orderId, string region) => WriteEvent(1, orderId, region);

    [Event(2, Level = EventLevel.Error)]
    public void OrderFailed(int orderId, string reason) => WriteEvent(2, orderId, reason);
}

// Usage — extremely cheap when no one is listening
OrderEventSource.Log.OrderPlaced(order.Id, order.Region);
```

`EventSource` is lower-level than `ILogger` and faster (no string formatting unless a listener is attached, binary serialization). Most app code uses `ILogger` (which can route to EventSource); reach for `EventSource` directly for very high-frequency diagnostics you want collectable by EventPipe tools without log overhead. The companion `EventCounter`/`Meter` expose live metrics to `dotnet-counters`.

---

## `Stopwatch` — precise elapsed time

```csharp
var sw = Stopwatch.StartNew();
DoWork();
sw.Stop();
Console.WriteLine($"{sw.Elapsed.TotalMilliseconds} ms");
long ticks = Stopwatch.GetTimestamp();                 // raw high-res timestamp
TimeSpan elapsed = Stopwatch.GetElapsedTime(startTicks); // .NET 7+: elapsed from a timestamp
```

`Stopwatch` uses a high-resolution monotonic timer (`QueryPerformanceCounter`/`clock_gettime`) — accurate for measuring durations and **unaffected by clock changes** (unlike `DateTime.Now` subtraction). Use it for ad-hoc timing; for rigorous micro-benchmarks use **BenchmarkDotNet** ([Ch21](../21-Performance/README.md), CSharpBook Ch16 §07), which handles warmup/statistics that a raw `Stopwatch` can't.

---

## `Process` — process & environment info

```csharp
using System.Diagnostics;
var proc = Process.GetCurrentProcess();
proc.WorkingSet64;            // memory
proc.TotalProcessorTime;      // CPU time used
proc.Threads.Count;

// Launch and capture an external process
var psi = new ProcessStartInfo("git", "rev-parse HEAD") { RedirectStandardOutput = true };
using var p = Process.Start(psi)!;
string sha = await p.StandardOutput.ReadToEndAsync();
await p.WaitForExitAsync();
```

`Process` inspects the current process or launches/controls external ones (capturing stdout/stderr, waiting, killing). Useful for tooling and orchestration; be careful with untrusted arguments (command injection) and always read redirected streams to avoid deadlocks.

---

## `Debug` & `Trace`

```csharp
Debug.Assert(index >= 0, "index must be non-negative");   // Debug builds only (removed in Release)
Debug.WriteLine($"value = {value}");                       // Debug builds only
Trace.WriteLine("always present");                          // both, if Trace listeners configured
Trace.TraceError("something failed");
```

`Debug.*` calls are compiled out in Release ( `[Conditional("DEBUG")]` — CSharpBook Ch12 §03), making them a free invariant-checking tool during development. `Trace.*` works in all builds via configured `TraceListener`s. Modern apps mostly use `ILogger` instead, but `Debug.Assert` remains a handy dev-time invariant check.

---

## Common gotchas

### Forgetting `StartActivity` returns null

When no listener is registered, `StartActivity` returns `null` — guard with `?.`. (This is the feature that makes instrumentation free when uncollected, not a bug.)

### Creating `ActivitySource`/`Meter` per call

Create them **once** (static) per library/component. Creating per operation is wasteful and fragments the telemetry namespace.

### Using `DateTime` subtraction for elapsed time

`DateTime.Now` can jump (NTP sync, DST) → negative or wrong durations. Use `Stopwatch` (monotonic).

### High-cardinality metric tags

Tagging metrics with unbounded values (user IDs, request IDs) explodes the metric series count and overwhelms backends. Use low-cardinality dimensions (region, status, route template).

### Not reading redirected `Process` output

A child process can deadlock if its stdout/stderr buffer fills and you're not reading. Read both streams (async) while waiting.

### Reinventing tracing/metrics with logs

Logging "started/finished" lines to compute durations is fragile. Use `Activity` (traces) and `Meter` (metrics) — purpose-built and OTel-compatible.

---

## Summary

- The BCL ships **vendor-neutral, OpenTelemetry-compatible** observability primitives: **`ActivitySource`/`Activity`** (distributed tracing/spans), **`Meter`** (metrics: counters/histograms/gauges), **`EventSource`** (low-overhead structured events).
- They cost **~nothing when no listener subscribes** (`StartActivity` returns null; metrics no-op) — instrument freely; create sources/meters **once** (static).
- Activities **nest** (via `Activity.Current`/`AsyncLocal`) and propagate across services via **W3C Trace Context**, forming distributed traces.
- Use **`Stopwatch`** (monotonic) for elapsed time, **`Process`** for process control, **`Debug.Assert`** for dev-time invariants.
- Avoid high-cardinality metric tags; don't compute durations from `DateTime`. The full pipeline (OTel, exporters, dashboards) is [Chapter 12](../12-Observability/README.md).

→ Next: [09-Reflection.md](09-Reflection.md)
