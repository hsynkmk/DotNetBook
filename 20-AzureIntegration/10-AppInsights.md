# Application Insights

## Azure's APM, via OpenTelemetry

**Application Insights** is Azure's Application Performance Monitoring (APM) service — the place your app's **logs, metrics, and distributed traces** ([Ch12 Observability](../12-Observability/README.md)) land for querying, dashboards, alerting, and diagnostics in production. The **modern, recommended path** is to instrument your app with **OpenTelemetry** ([Ch12 §07](../12-Observability/07-OpenTelemetry.md)) and export to Application Insights — rather than the legacy Application Insights SDK. This keeps your instrumentation **vendor-neutral** (standard OTel APIs and the `Activity`/`Meter`/`ILogger` primitives — [Ch12](../12-Observability/README.md)) while sending the data to Azure's managed backend.

```csharp
// The modern path: OpenTelemetry exporting to Application Insights
builder.Services.AddOpenTelemetry().UseAzureMonitor();   // Azure.Monitor.OpenTelemetry.AspNetCore

// Or wire the OTLP/Azure Monitor exporter onto your existing OTel setup ([Ch12 §07])
```

---

## Why OpenTelemetry over the legacy SDK

Historically you'd add the Application Insights SDK directly. The modern guidance is **OpenTelemetry → Azure Monitor exporter**, because:

- **Vendor-neutral instrumentation** — you instrument with **standard OTel APIs** (and .NET's built-in `Activity`/`Meter`/`ILogger` — [Ch12 §02–06](../12-Observability/02-ILogger.md)); switching or adding backends (Grafana, Datadog, Jaeger) later doesn't require re-instrumenting.
- **Consistency with the ecosystem** — the same telemetry that flows to the Aspire dashboard locally ([Ch18 §06](../18-Aspire/06-Dashboard.md)) flows to App Insights in production — identical instrumentation, different exporter.
- **Future-proof** — OTel is the industry standard; Microsoft's investment is in the OTel-based distro (`Azure.Monitor.OpenTelemetry.AspNetCore`).

So you write **standard observability code** ([Ch12](../12-Observability/README.md)) and just configure the **Azure Monitor exporter** — Application Insights becomes the *destination*, not a coupling in your code.

---

## What you get in Application Insights

Once telemetry flows in, App Insights provides production observability:

- **Distributed tracing** — requests correlated across services (the `Activity`/trace context — [Ch12 §06](../12-Observability/06-Activities.md)) shown as end-to-end transactions, with the **application map** visualizing service dependencies and their health/latency.
- **Metrics** — request rates, durations, dependency calls, plus your custom `Meter`s ([Ch12 §05](../12-Observability/05-Metrics.md)) — chartable, with **live metrics** (real-time).
- **Logs** — structured `ILogger` logs ([Ch12 §02](../12-Observability/02-ILogger.md)), queryable with **KQL** (Kusto Query Language) for powerful ad-hoc analysis.
- **Failures & performance** — drill into failed requests, exceptions, and slow dependencies; **end-to-end transaction** views correlate a request's logs, trace, and exceptions.
- **Alerts** — fire on metric thresholds, failure rates, or KQL queries.

This is the production counterpart to the Aspire dev dashboard ([Ch18 §06](../18-Aspire/06-Dashboard.md)) — durable, scalable, queryable, with alerting.

---

## KQL — querying telemetry

App Insights stores telemetry in tables you query with **KQL** (Kusto Query Language) — a powerful analytics language for slicing logs/metrics/traces:

```kql
// Slowest endpoints in the last hour:
requests
| where timestamp > ago(1h)
| summarize p95 = percentile(duration, 95), count() by name
| order by p95 desc

// Exceptions correlated to a failing operation:
exceptions
| where timestamp > ago(1h)
| join requests on operation_Id
```

KQL turns your telemetry into queryable data — find the p95 latency per endpoint, correlate exceptions to requests via `operation_Id` (the trace correlation id), build custom dashboards and alerts. This ad-hoc query power is a major reason teams use App Insights.

---

## Correlation and sampling

- **Correlation**: App Insights uses the W3C trace context (`operation_Id`/trace id — [Ch12 §06](../12-Observability/06-Activities.md)) propagated across services, so a request's traces, logs, and exceptions are linked — click a failed request to see its full distributed trace and associated logs.
- **Sampling**: high-volume apps generate enormous telemetry; **sampling** keeps a representative subset to control cost/volume while preserving statistical accuracy (and keeping correlated items together). Configure sampling to balance fidelity vs cost — capturing everything is expensive at scale.

---

## Setup with managed identity

Connect using **managed identity** where possible ([01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)) (or a connection string for the App Insights resource). On **App Service** ([02-AppService.md](02-AppService.md)) and other Azure hosts, App Insights integration is often a toggle that injects the connection string. The data path: your app's OTel instrumentation → Azure Monitor exporter → Application Insights resource → query/dashboard/alert.

---

## Common gotchas

### Using the legacy SDK for new apps

The modern path is **OpenTelemetry + Azure Monitor exporter** (`Azure.Monitor.OpenTelemetry.AspNetCore`), not the classic Application Insights SDK. OTel keeps instrumentation vendor-neutral and future-proof ([Ch12 §07](../12-Observability/07-OpenTelemetry.md)).

### No sampling on high-volume apps

Exporting every telemetry item at scale is costly and can hit limits. Configure **sampling** to keep a representative subset while preserving correlation.

### Broken correlation

If trace context isn't propagated (custom HTTP without the headers, a non-instrumented hop), traces fragment across services. Use the instrumented `HttpClient`/frameworks so W3C trace context flows ([Ch12 §06](../12-Observability/06-Activities.md)).

### Logging sensitive data

Telemetry can capture request bodies/headers/PII. Scrub/redact sensitive data before it reaches App Insights ([Ch12 §02](../12-Observability/02-ILogger.md), [Ch13 §07](../13-Configuration/07-Secrets.md)) — don't ship secrets/PII to the APM.

### Treating it as the only signal

App Insights is the production APM; locally the **Aspire dashboard** ([Ch18 §06](../18-Aspire/06-Dashboard.md)) shows the same telemetry. Use the right surface per environment — same instrumentation, different backend.

---

## Summary

- **Application Insights** is Azure's APM — production **logs, metrics, distributed traces**, dashboards, and alerting; the **modern path** is **OpenTelemetry** instrumentation ([Ch12 §07](../12-Observability/07-OpenTelemetry.md)) exporting via the **Azure Monitor exporter** (`UseAzureMonitor()`), **not** the legacy SDK — keeping instrumentation **vendor-neutral**.
- It provides **distributed tracing** (correlated end-to-end transactions + **application map**), **metrics** (incl. custom `Meter`s + live metrics), and **structured logs** — all queryable with **KQL** for ad-hoc analysis, dashboards, and alerts.
- **Correlation** via W3C trace context links a request's traces/logs/exceptions (`operation_Id`); use **sampling** to control telemetry volume/cost at scale while preserving accuracy.
- Connect via **managed identity**/connection string ([01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)); it's the **production** counterpart to the Aspire dev dashboard ([Ch18 §06](../18-Aspire/06-Dashboard.md)) — same instrumentation, durable scalable backend. Scrub PII/secrets before export.

→ Next: [Questions.md](Questions.md)
