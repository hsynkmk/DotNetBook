# Log Providers

## Where your logs go

`ILogger` writes log entries; **logging providers** decide *where* they go — console, debug output, files, a structured store (Seq/Elasticsearch), a cloud service (Application Insights), or an OpenTelemetry exporter. The same logging code ([02-ILogger.md](02-ILogger.md)) routes to whichever providers you configure — your code logs through `ILogger`; *where* it lands is configuration.

```csharp
builder.Logging.ClearProviders();
builder.Logging.AddConsole();         // or AddJsonConsole() for structured output
builder.Logging.AddDebug();
// + third-party / OTel providers (below)
```

---

## Built-in providers

| Provider | Output | Use |
|---|---|---|
| **Console** | stdout (text) | dev; **containers** (the orchestrator collects stdout) |
| **JSON Console** (`AddJsonConsole`) | stdout (structured JSON) | containers + log aggregators that parse JSON |
| **Debug** | the debugger output window | local debugging |
| **EventSource** | ETW/EventPipe | `dotnet-trace` capture, low-overhead diagnostics ([09-Profiling.md](09-Profiling.md)) |
| **EventLog** (Windows) | Windows Event Log | Windows services |

For **containers/Kubernetes**, log to **stdout** (console) — the platform collects stdout and ships it to your logging backend. Use **`AddJsonConsole`** so the output is structured JSON that aggregators (Loki, ELK, Datadog) parse into queryable fields — preserving the structured logging from [02-ILogger.md](02-ILogger.md). Plain-text console loses the structure at the aggregator.

---

## Multiple providers & per-provider filtering

You can attach **multiple** providers simultaneously, and filter per provider/category:

```csharp
builder.Logging
    .AddConsole()
    .AddApplicationInsights();        // also send to App Insights

// Filter per provider + category
builder.Logging.AddFilter<ConsoleLoggerProvider>("Microsoft", LogLevel.Warning);
builder.Logging.AddFilter<ApplicationInsightsLoggerProvider>("MyApp", LogLevel.Information);
```

Each log entry goes to all registered providers (each applying its own level filter). So you might log verbosely to the console in dev but send only Warnings+ to a paid backend, or route different categories differently. Configure filters in `appsettings.json` under `Logging` (per provider via the provider alias).

---

## Third-party / structured backends

For production observability, route logs to a **structured store** that supports querying and correlation:

| Backend | Notes |
|---|---|
| **Serilog** | a logging *framework* with rich enrichment + many sinks ([04-Serilog.md](04-Serilog.md)) |
| **Seq** | structured log server (great dev/staging experience) |
| **Elasticsearch / OpenSearch (ELK)** | search/analytics over logs |
| **Grafana Loki** | log aggregation, pairs with Grafana/Prometheus |
| **Application Insights** | Azure's APM (logs + metrics + traces) |
| **Datadog / Splunk / etc.** | commercial observability platforms |

These ingest **structured** logs (the named fields from [02-ILogger.md](02-ILogger.md)) so you can search/aggregate by field and correlate with traces. **Serilog** ([04-Serilog.md](04-Serilog.md)) is a popular choice that plugs into `ILogger` and adds enrichment + a huge sink ecosystem.

---

## OpenTelemetry logging (the converging path)

The modern, vendor-neutral route: export logs via **OpenTelemetry** ([07-OpenTelemetry.md](07-OpenTelemetry.md)), so logs flow alongside metrics and traces to any OTLP-compatible backend:

```csharp
builder.Logging.AddOpenTelemetry(o => {
    o.IncludeScopes = true;                 // include log scopes (and the trace correlation)
    o.AddOtlpExporter();                     // → an OTel collector / backend
});
```

This unifies the three pillars under one pipeline and **automatically correlates logs with traces** (logs carry the current TraceId/SpanId — [02-ILogger.md](02-ILogger.md), [06-Activities.md](06-Activities.md)). It's the recommended direction: instrument with `ILogger`/`Meter`/`ActivitySource`, export everything via OTel, send to whichever backend you choose — no vendor lock-in.

---

## Choosing providers per environment

```
Development → Console (readable) + Debug; maybe Seq for structured browsing
Containers  → JSON Console (stdout, collected by the platform) → aggregator
Production  → OpenTelemetry → OTLP collector → your backend (Loki/App Insights/Datadog/...)
              (or Serilog → a structured sink)
```

Match providers to the environment: human-readable console in dev, structured stdout in containers, and a structured backend via OTel/Serilog in production. Keep the *logging code* unchanged — only the providers differ.

---

## Common gotchas

### Plain-text console in containers/production

Loses structure at the aggregator → not queryable by field. Use `AddJsonConsole` (or OTel/Serilog) so structured fields survive.

### Logging to files in containers

Container filesystems are ephemeral and files aren't collected by the orchestrator. Log to **stdout** (console); let the platform ship it.

### Same verbosity everywhere

Verbose logging to a paid backend is expensive; too quiet in dev hampers debugging. Filter per provider/category/level.

### Not correlating with traces

Logs without TraceId/SpanId can't be linked to traces. Use OTel logging (or include the trace id) so you can pivot between pillars ([06-Activities.md](06-Activities.md)).

### Vendor lock-in

Coding directly to a vendor's logging SDK couples you to it. Log through `ILogger` and export via **OpenTelemetry** — swap backends without changing code.

---

## Summary

- **Logging providers** route `ILogger` output to destinations; the same logging code feeds whatever providers you configure (built-in Console/JSON-Console/Debug/EventSource, or third-party/OTel).
- For **containers**, log to **stdout** with **`AddJsonConsole`** (structured JSON the platform/aggregator parses); don't log to files.
- Attach **multiple providers** with **per-provider/category filtering**; route to **structured backends** (Serilog, Seq, ELK, Loki, App Insights) so logs are queryable by field.
- Prefer **OpenTelemetry logging** (`AddOpenTelemetry().AddOtlpExporter()`) — unifies the three pillars, auto-correlates logs with traces, and avoids vendor lock-in.
- Match providers to the environment (readable in dev, structured stdout in containers, OTel/structured backend in prod) without changing logging code.

→ Next: [04-Serilog.md](04-Serilog.md)
