# 12 — Observability

## ⚡ 30-second answer

**Observability** is understanding a system's internal state from its external outputs — answering
"what happened and why" in production without a debugger. Three pillars, and the useful framing is
what each one is *for*: **metrics detect** (cheap aggregates → dashboards and alerts), **traces
localize** (one request's path across services → where the time went), **logs explain** (the
actual error). You correlate them with a shared **TraceId**. **OpenTelemetry** is the converged
standard, and the key .NET fact is that **`ILogger`, `Meter` and `ActivitySource` *are* the OTel
APIs** — instrument once, export anywhere. Two rules carry most of the value: **always use log
message templates, never interpolation** (that's what makes fields queryable), and **keep metric
dimensions low-cardinality** — a user id as a metric tag is a cardinality explosion; it belongs on
a trace span or in a log.

---

## Core mechanics

### The three pillars

| | Logs | Metrics | Traces |
|---|---|---|---|
| What | discrete detailed events | aggregated numbers over time | one request's path across services |
| Answers | **why** — the error, the context | **how** — rate, latency, saturation | **where** — which hop, which span |
| Cost | high per event | very low | medium (sampled) |
| Cardinality | unlimited | **must be low** | high is fine (ids belong here) |
| Role | explain | **detect / alert** | localize |

They're complementary: an alert fires on a **metric**, you open the **trace** to find which span
is slow, and you read the **log** for that span to see the exception.

### Logging

```csharp
_logger.LogInformation("Order {OrderId} shipped to {Region}", id, region);   // ✅ structured
_logger.LogError(ex, "Failed to ship order {OrderId}", id);                  // ✅ pass the exception

using var scope = _logger.BeginScope(new Dictionary<string, object>
{
    ["TenantId"] = tenantId                  // attached to every log inside this scope
});
```

**The non-negotiable rule**: message templates with named placeholders, never string
interpolation. The template preserves `OrderId` as a **queryable structured field**, and the
arguments aren't formatted at all if the level is filtered out.

Choose levels deliberately, always pass the exception object, and filter per category in
configuration. Use **`[LoggerMessage]`** source generation for hot paths — allocation-free and
AOT-friendly.

**In containers, log JSON to stdout** (`AddJsonConsole`) and let the platform collect it. Never
log to files.

### Metrics

```csharp
private static readonly Meter Meter = new("MyApp.Orders");           // static, created once
private static readonly Counter<long> Placed = Meter.CreateCounter<long>("orders.placed");
private static readonly Histogram<double> Duration =
    Meter.CreateHistogram<double>("orders.duration", "ms");

Placed.Add(1, new KeyValuePair<string, object?>("region", region));  // low-cardinality tag only
```

**Counters** for totals, **histograms** for distributions (→ **percentiles**, because averages
hide the tail), **observable** instruments for sampled gauges.

**Cardinality is the number-one pitfall.** Each unique combination of dimension values creates a
separate time series. `region` (10 values) is fine; `user_id` (2 million) will take down your
metrics backend. High-cardinality identifiers belong on **trace spans** or in **logs**.

What to measure: **RED** for services (**R**ate, **E**rrors, **D**uration) and **USE** for
resources (**U**tilization, **S**aturation, **E**rrors), plus business metrics.

### Traces

```csharp
private static readonly ActivitySource Source = new("MyApp.Orders");
using var activity = Source.StartActivity("PlaceOrder");
activity?.SetTag("order.id", id);            // high cardinality is FINE on a span
activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
```

A **trace** is one request as a tree of **spans**. Spans nest automatically via `Activity.Current`
(an `AsyncLocal`, so it flows across `await`), sharing a **TraceId**.

**W3C Trace Context** (`traceparent` header) propagates the trace across services —
**automatically over HTTP** (HttpClient and ASP.NET Core instrumentation), but **manually over
messaging**: you must put the context in the message headers and restore it in the consumer, or
your trace stops dead at the publish ([07](07-Messaging.md)).

Instrumentation is **near-free when nothing is collecting** — `StartActivity` returns `null`. So
instrument liberally. **Sample** at high volume, keeping errors and slow traces.

### OpenTelemetry

```csharp
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService("catalog-api", serviceVersion: version))
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddEntityFrameworkCoreInstrumentation()
        .AddSource("MyApp.Orders")            // ← your custom sources must be registered
        .AddOtlpExporter())
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()
        .AddRuntimeInstrumentation()          // GC, thread pool
        .AddMeter("MyApp.Orders")
        .AddOtlpExporter());
```

**Auto-instrumentation gives you a genuinely good baseline for almost no code**: RED metrics per
route, runtime metrics (GC, thread pool), and distributed traces spanning request → outbound HTTP
→ database, with automatic context propagation. Export via **OTLP** to a **Collector**, which
decouples your app from the choice of backend.

Set **resource attributes** (service name, version, environment) — without them you can't tell
which deployment a signal came from.

### Serilog

Plugs into `ILogger` and adds richer structured logging (`{@obj}` destructuring), a large **sink**
ecosystem, and its standout feature — **enrichment**, automatically attaching machine, service,
thread and **trace id** to every event without repeating them at call sites. Use the OpenTelemetry
sink to get Serilog's enrichment *and* OTel export.

### Profiling — the code-level "why"

Observability tells you *what* and *where*; profiling tells you *why* at the code level. The
cross-platform, production-safe `dotnet-*` tools:

| Tool | For |
|---|---|
| **`dotnet-counters`** | live metrics — **first triage** (CPU, GC, thread pool, exceptions) |
| **`dotnet-trace`** | CPU/event traces → hot stacks |
| **`dotnet-gcdump`** | heap snapshots → **leak hunting** (two snapshots + `gcroot`) |
| **`dotnet-dump`** / `dotnet-stack` | crashes, hangs, thread state |

Workflow: alert → `dotnet-counters` triage → the right deep tool → fix → verify
([21](21-Performance-Tooling.md)).

### Health checks

The coarse, always-on signal for orchestrators: **liveness** (alive? → restart, checks *only the
app*) vs **readiness** (can serve? → stop routing, checks dependencies). Conflating them causes
restart storms ([04](04-AspNetCore.md)). A passing health check is not a statement about
performance.

---

## 🪤 Traps & gotchas

- **String interpolation in logs** — `LogInformation($"Order {id}")` produces one opaque string
  with no queryable `OrderId` field, and formats the message even when the level is disabled. The
  single most common logging mistake.
- **Not passing the exception object** — `LogError("failed: " + ex.Message)` loses the stack trace
  and inner exceptions. The exception is the first parameter for a reason.
- **High-cardinality metric dimensions** — user id, request id, full URL with ids. Each unique
  value is a new time series; this takes down the metrics backend, not your app. Put them on spans.
- **Alerting on averages** — an average latency of 200ms can hide a p99 of 8 seconds. Alert on
  percentiles.
- **Logging secrets** — tokens, passwords, connection strings, PII. Once it's in the log
  aggregator it's in backups and probably in a third-party SaaS.
- **Logging at Information in a hot loop** — a per-request debug log at 10k rps is a bigger cost
  than the work being logged, and the bill is real.
- **Logging the same error at every layer** — one exception becomes five entries and the alert
  count is meaningless. Log once, at the boundary where it's handled.
- **Creating `Meter` or `ActivitySource` per request** — they're meant to be static and created
  once. Per-instance creation leaks and defeats the aggregation.
- **Forgetting `AddSource`/`AddMeter` for your custom instrumentation** — the code runs, the spans
  and metrics are created, and nothing is exported. Silently no data.
- **Losing trace context across a message broker** — HTTP propagation is automatic, messaging is
  not. Without putting `traceparent` in the message headers, the consumer's work appears as an
  unrelated trace and the end-to-end picture is gone.
- **Losing trace context in a background task** — `Activity.Current` flows across `await` but not
  across a queue. Capture and restore it explicitly
  ([08](08-BackgroundProcessing.md)).
- **No resource attributes** — signals arrive with no service name, version or environment, so you
  can't tell staging from production or which deployment regressed.
- **100% trace sampling in production** — cost and volume that nobody budgeted. Sample, but keep
  all errors and slow traces (tail sampling).
- **Logging to files in a container** — the container is ephemeral, so the logs vanish with it.
  stdout as JSON.
- **Treating a green health check as "healthy"** — it means the process answered, not that latency
  is acceptable or that the error rate is low.
- **Instrumenting everything and monitoring nothing** — telemetry nobody alerts on is a cost
  centre. Start from the alerts you want and work backwards.
- **Profiling in Debug or with a debugger attached** — the numbers are meaningless. Release, not
  attached.

---

## ❓ Likely questions

**Q: What are the three pillars and how do they differ?**
A: Logs are discrete detailed events that explain *why*; metrics are cheap aggregates over time
that tell you *how* the system is performing and are what you alert on; traces show one request's
path across services and tell you *where* the time went. You use all three together, correlated by
a shared TraceId.

**Q: Monitoring vs observability?**
A: Monitoring watches for problems you already predicted — dashboards and alerts on known failure
modes. Observability is the property that lets you investigate a problem you *didn't* predict,
asking new questions of existing telemetry without shipping new code.

**Q: Why must you use log message templates instead of interpolation?**
A: The template keeps the parameters as named structured fields, so you can query "all logs where
OrderId = 123" in your backend. Interpolation collapses everything into one string. It also avoids
paying the formatting cost when the log level is filtered out.

**Q: What is metric cardinality and why does it matter?**
A: Every unique combination of dimension values is a separate time series stored and indexed by
your metrics backend. A low-cardinality dimension like region or status code is fine; a user id or
request id creates millions of series and will bring the backend down. High-cardinality data
belongs on trace spans or in logs.

**Q: Why percentiles rather than averages?**
A: An average is dominated by the bulk of fast requests and hides the tail. p99 latency is what
your unhappiest 1% of users actually experience, and it's usually the thing that's broken.

**Q: How does a trace get from one service to another?**
A: Via **W3C Trace Context** — the `traceparent` header carrying the TraceId and parent SpanId.
Over HTTP, ASP.NET Core and `HttpClient` instrumentation handle it automatically. Over a message
broker it's manual: you put the context in the message headers and restore it in the consumer.

**Q: What's the relationship between .NET's APIs and OpenTelemetry?**
A: They're the same thing. `ILogger`, `Meter` and `ActivitySource` **are** the OTel APIs in .NET —
there's no separate instrumentation library to adopt. You instrument with the BCL types and the
OTel SDK collects and exports them, so you're never locked into a vendor.

**Q: What does OpenTelemetry auto-instrumentation give you?**
A: A lot for a few lines: RED metrics per route from ASP.NET Core, runtime metrics (GC, thread
pool), traces spanning the incoming request through outbound HTTP calls and EF Core queries, and
automatic trace-context propagation. Then you add your own `Meter` and `ActivitySource` for
business telemetry.

**Q: What does instrumentation cost when nothing is collecting?**
A: Essentially nothing — `StartActivity` returns `null` when no listener has subscribed, and
metric instruments no-op. That's deliberate so libraries can be instrumented unconditionally, and
it's why you should instrument liberally.

**Q: What is RED, and USE?**
A: **RED** for services: Rate (requests/sec), Errors (failure rate), Duration (latency
distribution). **USE** for resources: Utilization, Saturation, Errors. Between them they cover
"is the service healthy" and "is the machine healthy".

**Q: How do you correlate a log line with a trace?**
A: ASP.NET Core automatically attaches the current TraceId and SpanId to log entries, so your
backend can pivot from a slow span straight to the logs written inside it — which is the whole
point of running the three pillars through one pipeline.

**Q: How would you debug a slow endpoint in production?**
A: Start with metrics to confirm it's real and scoped (which route, which percentile, since when).
Open a trace for a slow request to see which span dominates — your code, a database query, a
downstream call. Read the logs for that span for the specific error or parameters. If it's CPU or
memory inside your own code, attach `dotnet-counters` for triage and then `dotnet-trace` or
`dotnet-gcdump`.

**Q: How would you find a memory leak?**
A: `dotnet-counters` first to confirm the heap is growing and Gen 2 collections aren't reclaiming
it. Then two `dotnet-gcdump` snapshots minutes apart, diff them to find which type is growing, and
use `gcroot` in `dotnet-dump` to find what's holding the references — usually a static collection,
a cache without a bound, or an event handler never unsubscribed.

**Q: Where should Serilog fit if you're using OpenTelemetry?**
A: Use Serilog for its enrichment and destructuring, and configure the OpenTelemetry sink so those
logs still flow through the OTLP pipeline correlated with your traces. You don't have to choose.

---

## 🎓 Senior Extra

- **Start from the alert, not the instrumentation.** Decide what page you want to wake someone up,
  work backwards to the metric that detects it, then to the trace and log fields you'd need to
  diagnose it. Telemetry designed the other way round is expensive and unactionable.
- **Tail-based sampling** (in the Collector) keeps every errored and slow trace while sampling the
  boring ones, which is what makes sampling acceptable — head-based sampling throws away the
  traces you actually want.
- **The OTel Collector is the right architecture**, not a nicety: your app speaks only OTLP, and
  the Collector handles batching, retry, redaction, sampling and fan-out to backends. Changing
  vendors becomes a config change.
- **`Activity.Current` is `AsyncLocal`**, which is why nesting "just works" across `await` — and
  exactly why it doesn't survive a hand-off through a channel or a `Task.Run` you didn't await.
- **Span tags are the right home for high-cardinality data.** The mental model worth stating:
  metrics for aggregation, spans for a specific request's identifiers, logs for the narrative.
- **Exemplars** link a metric bucket to a sample trace, so you can jump from "p99 is bad" to an
  actual slow trace in one click. Underused and worth knowing about.
- **Semantic conventions matter** — using the OTel standard attribute names (`http.route`,
  `db.system`, `server.address`) is what makes vendor dashboards work out of the box instead of
  requiring custom queries.
- **Log volume is a cost you can architect**: sample debug logs, use `[LoggerMessage]` on hot
  paths, and set per-category levels from configuration so you can raise verbosity for one
  namespace at runtime without redeploying ([03](03-Hosting-DI-Config.md)).
- **Instrument the *absence* of events.** A background job that stops running produces no error and
  no log — the alert has to be on the missing heartbeat, not on a failure
  ([08](08-BackgroundProcessing.md)).
- **Continuous profilers** (Application Insights Profiler, Datadog, Pyroscope) give always-on
  production profiling at low overhead, which closes the gap between "the metric says CPU is high"
  and "here is the method".

→ Deeper: [`../12-Observability/`](../12-Observability/README.md)
