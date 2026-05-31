# OpenTelemetry

## The unifying observability standard

**OpenTelemetry (OTel)** is the vendor-neutral, open standard for generating, collecting, and exporting all three pillars — logs, metrics, traces — with broad backend support. The crucial .NET fact: the BCL's observability primitives (`ILogger`, `Meter`, `ActivitySource` — [02](02-ILogger.md)/[05](05-Metrics.md)/[06](06-Activities.md)) **are** the OTel APIs. So you instrument with the standard .NET types, and the **OTel SDK** collects and **exports** that telemetry to any compatible backend — instrument once, send anywhere, no vendor lock-in.

```csharp
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService("OrderApi"))
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()      // auto-trace incoming requests
        .AddHttpClientInstrumentation()       // auto-trace outbound HTTP
        .AddEntityFrameworkCoreInstrumentation()  // auto-trace EF queries
        .AddSource("Shop.Orders")             // YOUR ActivitySource
        .AddOtlpExporter())                    // → an OTel collector / backend
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()          // GC, thread pool, etc.
        .AddMeter("Shop.Orders")              // YOUR Meter
        .AddOtlpExporter());

builder.Logging.AddOpenTelemetry(o => o.AddOtlpExporter());   // logs too
```

That wires up the whole pipeline: automatic instrumentation for ASP.NET Core/HttpClient/EF, your custom sources, and export of all three pillars.

---

## The architecture

```
Your app:  ILogger + Meter + ActivitySource  (instrumentation — the OTel APIs)
              ↓ collected by
           OpenTelemetry SDK (in-process: batching, sampling, resource attributes)
              ↓ exports via OTLP
           OpenTelemetry Collector (optional, out-of-process: receive, process, route)
              ↓
           Backend(s): Jaeger/Tempo (traces), Prometheus (metrics), Loki (logs),
                       or Application Insights / Datadog / Grafana Cloud / etc.
```

- **Instrumentation** — your code + auto-instrumentation libraries produce telemetry via the .NET APIs.
- **SDK** — collects it in-process, applies sampling/batching, attaches resource attributes (service name, version, instance), and exports.
- **Exporters** — send telemetry out, most commonly via **OTLP** (the OTel wire protocol) to a collector or directly to a backend.
- **Collector** (optional but recommended) — a separate process that receives OTLP, processes/filters/routes telemetry, and forwards to one or more backends — decoupling your app from backend specifics.

---

## Auto-instrumentation (huge value)

OTel **instrumentation libraries** automatically produce telemetry for common components — you get rich traces and metrics with almost no code:

```csharp
.AddAspNetCoreInstrumentation()       // spans + metrics for every incoming request (RED metrics free!)
.AddHttpClientInstrumentation()        // spans for outbound calls (+ traceparent propagation)
.AddEntityFrameworkCoreInstrumentation() // spans for DB queries (see the slow query in the trace)
.AddRuntimeInstrumentation()           // GC, thread pool, exceptions metrics
.AddSqlClientInstrumentation()
```

These give you, for free: traces of incoming requests and outbound HTTP/DB calls (with automatic W3C context propagation — [06-Activities.md](06-Activities.md)), and RED/runtime metrics ([05-Metrics.md](05-Metrics.md)). So a request's trace shows the ASP.NET span → HttpClient spans → EF query spans automatically — you only add **custom** spans/metrics for your business operations. This auto-instrumentation is a major reason OTel adoption is easy. (More in [08-AspNetTelemetry.md](08-AspNetTelemetry.md).)

---

## Exporters & backends

OTel is **backend-agnostic** — the same instrumentation exports to whatever you choose:

| Exporter / backend | For |
|---|---|
| **OTLP** | the standard protocol → a collector or any OTLP backend (recommended) |
| **Prometheus** | metrics (scrape endpoint) |
| **Jaeger / Zipkin / Tempo** | traces |
| **Azure Monitor / Application Insights** | all three (Azure) |
| **Datadog / Grafana Cloud / Honeycomb / etc.** | commercial platforms (via OTLP) |
| **Console** | dev/debugging |

Prefer **OTLP** → an **OpenTelemetry Collector**, which then routes to your backends. This decouples your app from backend choices: switch from Jaeger to Tempo, or add Datadog, by reconfiguring the collector — **no app code changes**. This vendor neutrality (avoid lock-in) is OTel's headline benefit.

---

## Resource attributes & correlation

OTel attaches **resource attributes** identifying *what* produced the telemetry — service name, version, instance id, environment, region:

```csharp
.ConfigureResource(r => r
    .AddService(serviceName: "OrderApi", serviceVersion: "1.2.0")
    .AddAttributes([new("deployment.environment", builder.Environment.EnvironmentName)]));
```

These let you filter/group telemetry by service/version/environment across the whole system. And because all three pillars flow through OTel, they're **correlated**: a trace's TraceId appears on its logs and links to relevant metrics — so you pivot seamlessly between pillars (the cross-pillar correlation from [01-WhatIsObservability.md](01-WhatIsObservability.md)) in a backend like Grafana or Application Insights.

---

## Sampling & cost control

Configure sampling in the SDK to control trace volume/cost ([06-Activities.md](06-Activities.md)):

```csharp
.WithTracing(t => t.SetSampler(new TraceIdRatioBasedSampler(0.1)));   // sample 10% of traces
```

For richer control, **tail sampling** in the collector keeps all error/slow traces while sampling the rest. Metrics are cheap (aggregated) but watch cardinality ([05-Metrics.md](05-Metrics.md)); logs are the most expensive — control verbosity by level. OTel's pipeline (SDK + collector) is where you tune the cost/signal trade-off centrally.

---

## Why OTel (the case for adopting it)

- **Vendor-neutral** — instrument once, export to any backend; switch backends without code changes (no lock-in).
- **Unified** — logs, metrics, traces in one pipeline, **correlated**.
- **Built into .NET** — the BCL primitives *are* the OTel APIs; auto-instrumentation covers ASP.NET Core/HttpClient/EF for free.
- **Industry standard** — broad ecosystem and backend support; the converged direction.

This is why OTel is the recommended observability approach for modern .NET — and why instrumenting with `ILogger`/`Meter`/`ActivitySource` (rather than a vendor SDK) is the right default.

---

## Common gotchas

### Coding to a vendor SDK instead of OTel

Couples you to one backend. Instrument with the .NET/OTel APIs and export via OTLP — swap backends freely.

### Forgetting to register your sources/meters

Auto-instrumentation covers framework components, but your **custom** `ActivitySource`/`Meter` must be added (`AddSource`/`AddMeter`) or your business telemetry isn't exported.

### No sampling at high volume

Exporting every trace at scale is expensive. Sample (keep errors/slow traces) in the SDK/collector.

### High-cardinality metric dimensions

OTel won't save you from cardinality explosion — keep metric dimensions low-cardinality ([05-Metrics.md](05-Metrics.md)).

### Exporting directly to backends from every app

Direct export couples apps to backends and duplicates config. Export **OTLP → a collector** that routes to backends — central, swappable.

### Missing resource attributes

Without service name/version/environment, telemetry is hard to attribute. Configure resource attributes.

---

## Summary

- **OpenTelemetry** is the vendor-neutral standard unifying logs, metrics, and traces; .NET's `ILogger`/`Meter`/`ActivitySource` **are** the OTel APIs — instrument once, export anywhere.
- The **SDK** collects in-process (sampling, batching, resource attributes) and **exports** (preferably **OTLP** → an **OTel Collector** → your backends), decoupling your app from backend choices.
- **Auto-instrumentation** (`AddAspNetCoreInstrumentation`/`AddHttpClientInstrumentation`/`AddEntityFrameworkCoreInstrumentation`/`AddRuntimeInstrumentation`) gives rich traces + RED/runtime metrics + automatic context propagation **for free**; add your custom sources/meters (`AddSource`/`AddMeter`).
- All three pillars flow through OTel **correlated** (shared TraceId); configure **resource attributes** (service/version/env) and **sampling** (keep errors/slow traces) for attribution and cost control.
- OTel's vendor neutrality, unification, and built-in .NET support make it the recommended observability approach.

→ Next: [08-AspNetTelemetry.md](08-AspNetTelemetry.md)
