# Built-in ASP.NET Core Telemetry

## You get a lot for free

Modern .NET frameworks are **instrumented out of the box** — ASP.NET Core, `HttpClient`, EF Core, and the runtime itself emit metrics and activities via the standard APIs ([05](05-Metrics.md)/[06](06-Activities.md)). Once you wire up OpenTelemetry ([07-OpenTelemetry.md](07-OpenTelemetry.md)), you get rich traces and metrics with **almost no custom code** — you only add instrumentation for your *business* operations.

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()       // incoming request spans
        .AddHttpClientInstrumentation()        // outbound HTTP spans (+ context propagation)
        .AddEntityFrameworkCoreInstrumentation()) // DB query spans
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()        // request metrics (RED)
        .AddHttpClientInstrumentation()         // outbound HTTP metrics
        .AddRuntimeInstrumentation());          // GC, thread pool, exceptions
```

---

## Built-in metrics (RED + runtime, for free)

The frameworks emit standardized metrics ([05-Metrics.md](05-Metrics.md))) — much of the **RED** (Rate/Errors/Duration) and **USE** picture without writing any:

| Source | Metrics (examples) |
|---|---|
| **ASP.NET Core** (`Microsoft.AspNetCore.Hosting`) | request count, request duration (histogram → p99), active requests, by route/status/method |
| **Kestrel** | active connections, queued connections, TLS handshakes |
| **HttpClient** (`System.Net.Http`) | outbound request duration, active requests, connection pool usage |
| **Runtime** (`System.Runtime`) | GC heap size, gen-0/1/2 counts, time-in-GC, thread-pool queue length, exception count |
| **EF Core / SqlClient** | query duration, active connections |

So **request rate, error rate, and latency percentiles per route** — the core service SLIs — come for free, dimensioned by **route template** (`/orders/{id}`, low-cardinality — [05-Metrics.md](05-Metrics.md)), status, and method. The runtime metrics surface GC pressure and thread-pool saturation ([Ch01](../01-Runtime/README.md)) — early-warning signals for cascading failure ([Ch11 §01](../11-Resilience/01-WhyResilience.md)). View them live with `dotnet-counters` ([09-Profiling.md](09-Profiling.md)) or export via OTel.

---

## Built-in tracing (automatic spans + propagation)

The framework instrumentation produces a trace tree automatically for each request:

```
[ASP.NET Core: POST /orders]                    ← AddAspNetCoreInstrumentation (incoming request span)
   [HttpClient: GET inventory-svc/stock]        ← AddHttpClientInstrumentation (outbound span)
   [EF Core: SELECT ... FROM Orders]            ← AddEntityFrameworkCoreInstrumentation (DB span)
   [your custom span: "ApplyDiscount"]          ← YOUR ActivitySource
```

Crucially, `HttpClient` instrumentation **propagates W3C trace context** (`traceparent`) automatically ([06-Activities.md](06-Activities.md)), so a request's trace flows across service boundaries — the inventory service's spans become children of this trace. You get end-to-end distributed traces of incoming → outbound → DB with **zero manual span code**; you add custom spans only for meaningful business steps.

---

## What you still instrument: business telemetry

The built-ins cover *technical* operations (HTTP, DB, runtime). They **don't** know your *domain* — so you add:

```csharp
private static readonly Meter Meter = new("Shop.Orders");
private static readonly Counter<long> OrdersPlaced = Meter.CreateCounter<long>("orders.placed");
private static readonly ActivitySource Source = new("Shop.Orders");

public async Task PlaceAsync(Order o) {
    using var activity = Source.StartActivity("PlaceOrder");      // business span
    activity?.SetTag("order.tier", o.CustomerTier);
    // ... business logic ...
    OrdersPlaced.Add(1, new("region", o.Region));                 // business metric
}

// Register YOUR sources with OTel so they're exported:
.WithTracing(t => t.AddSource("Shop.Orders"))
.WithMetrics(m => m.AddMeter("Shop.Orders"));
```

Add **business metrics** (orders placed, revenue, signups — low-cardinality dimensions) and **business spans** (meaningful operations within a request). Remember to **register your custom sources/meters** (`AddSource`/`AddMeter`) — auto-instrumentation only covers framework components ([07-OpenTelemetry.md](07-OpenTelemetry.md)).

---

## Enriching framework telemetry

You can enrich the auto-generated spans/metrics with custom tags:

```csharp
.AddAspNetCoreInstrumentation(o => {
    o.EnrichWithHttpRequest = (activity, req) => activity.SetTag("tenant", req.Headers["X-Tenant"]);
    o.RecordException = true;   // attach exception details to the span
});
```

Enrichment callbacks let you add domain context (tenant, user tier) to the framework's request spans, or capture exception details — without writing your own instrumentation. Useful for adding the few extra attributes the framework can't know about.

---

## The "free observability" payoff

The combination is powerful: wire up OTel + the instrumentation libraries, and you immediately get:
- **RED metrics** per route (rate, error rate, latency percentiles) → dashboards/alerts.
- **Runtime metrics** (GC, thread pool) → catch saturation early.
- **Distributed traces** (request → HTTP → DB) with cross-service propagation → diagnose latency.
- **Correlated logs** (TraceId on every log — [02-ILogger.md](02-ILogger.md)).

…for a few lines of setup. Then you layer **business** telemetry on top. This low-effort, high-value baseline is why modern .NET observability is approachable — most of the work is configuration, not instrumentation code.

---

## Common gotchas

### Not registering custom sources/meters

Auto-instrumentation covers framework components; your custom `ActivitySource`/`Meter` need `AddSource`/`AddMeter` or your business telemetry isn't exported.

### Expecting business telemetry for free

The built-ins cover HTTP/DB/runtime, not your domain. Add business metrics/spans for orders, revenue, etc.

### Ignoring runtime metrics

GC time and thread-pool queue length are early-warning signals for the cascading failures from [Ch11 §01](../11-Resilience/01-WhyResilience.md). Watch them.

### High-cardinality enrichment

Enriching framework spans/metrics with high-cardinality tags (on *metrics* especially) explodes series. Low-cardinality on metrics; ids are fine on spans ([05-Metrics.md](05-Metrics.md)).

### Not enabling instrumentation libraries

`AddOpenTelemetry()` alone doesn't auto-instrument — you must add the instrumentation (`AddAspNetCoreInstrumentation`, etc.) to get the framework telemetry.

---

## Summary

- ASP.NET Core, **HttpClient**, **EF Core**, and the **runtime** are instrumented out of the box — wire up OTel + the instrumentation libraries and get **RED metrics** (rate/errors/latency per route), **runtime metrics** (GC, thread pool), and **distributed traces** (request → HTTP → DB, with **automatic W3C context propagation**) for almost no code.
- Add **business telemetry** yourself — custom `Meter` metrics (orders, revenue — low-cardinality) and `ActivitySource` spans for domain operations — and **register them** (`AddSource`/`AddMeter`).
- **Enrich** framework spans/metrics with domain tags (tenant, user tier) via enrichment callbacks; capture exceptions on spans.
- Logs are auto-**correlated** with traces (TraceId). The payoff: a high-value observability baseline for a few lines of setup, then layer business telemetry on top.

→ Next: [09-Profiling.md](09-Profiling.md)
