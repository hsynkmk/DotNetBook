# Chapter 12 — Observability

> The three pillars: logs, metrics, traces. Plus distributed tracing, structured logging, and OpenTelemetry — the open standard the entire industry is converging on.

**Prerequisites**: Chapter 03 (Hosting & DI), Chapter 09 (HTTP).

**Time to read**: ~6-8 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-WhatIsObservability.md](01-WhatIsObservability.md) | Logs vs metrics vs traces — what each is for. |
| [02-ILogger.md](02-ILogger.md) | `ILogger<T>`, structured logging, log levels, scopes, `[LoggerMessage]` source-gen. |
| [03-LogProviders.md](03-LogProviders.md) | Console, Debug, EventSource, file (third-party), Azure Application Insights, Serilog. |
| [04-Serilog.md](04-Serilog.md) | Serilog's enrichment, sinks, structured event design. |
| [05-Metrics.md](05-Metrics.md) | `System.Diagnostics.Metrics` — `Meter`, `Counter<T>`, `Histogram<T>`, OTel-compatible. |
| [06-Activities.md](06-Activities.md) | `Activity`, `ActivitySource` — distributed tracing primitives. |
| [07-OpenTelemetry.md](07-OpenTelemetry.md) | OTel SDK for .NET — exporters (OTLP, Jaeger, Zipkin), instrumentation packages. |
| [08-AspNetTelemetry.md](08-AspNetTelemetry.md) | Built-in metrics and activities in ASP.NET Core, HttpClient, EF Core. |
| [09-Profiling.md](09-Profiling.md) | dotnet-counters, dotnet-trace, PerfView, Continuous Profiler (Azure). |
| [10-HealthChecks.md](10-HealthChecks.md) | Health endpoints, readiness vs liveness, custom checks. |
| [Questions.md](Questions.md) | Drilling. |
| [Coding.md](Coding.md) | Add structured logging + OTel traces + custom metrics. |

→ Begin: [01-WhatIsObservability.md](01-WhatIsObservability.md)
