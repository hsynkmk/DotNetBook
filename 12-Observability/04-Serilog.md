# Serilog

## Structured logging, done richly

Serilog is the most popular third-party logging framework for .NET. It plugs into `ILogger` (so your logging code is unchanged — [02-ILogger.md](02-ILogger.md)) but adds **first-class structured logging**, **enrichment** (automatically attaching context to every log), and a huge ecosystem of **sinks** (output destinations). When you want richer structured logging than the built-in providers offer, Serilog is the common choice.

```csharp
// Program.cs
builder.Services.AddSerilog((services, config) => config
    .ReadFrom.Configuration(builder.Configuration)   // configure from appsettings
    .ReadFrom.Services(services)
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .WriteTo.Console(formatter: new JsonFormatter())  // structured JSON to stdout
    .WriteTo.Seq("http://seq:5341"));                  // and to Seq
```

You still log via `ILogger<T>`; Serilog is the provider underneath, applying enrichment and writing to its sinks.

---

## Structured events (Serilog's core idea)

Serilog treats every log as a **structured event** — a message template plus named property values, preserved as data (not flattened to text):

```csharp
logger.LogInformation("Order {OrderId} shipped to {Country} in {Days} days", order.Id, country, days);
//   Serilog captures: { OrderId: 42, Country: "US", Days: 3, message template, timestamp, level }
```

This is the same structured-logging principle as [02-ILogger.md](02-ILogger.md), but Serilog was built around it from the start — properties are first-class, queryable, and preserved through the pipeline to structured sinks. Serilog can also serialize complex objects as structured data:

```csharp
logger.LogInformation("Processing {@Order}", order);   // @ = serialize the object's properties as structured data
//   vs {Order} which would just call ToString()
```

The `@` (destructuring) operator captures an object's structure (not just its `ToString()`), so the sink stores queryable fields for each property.

---

## Enrichment — context on every log

**Enrichers** automatically attach properties to *every* log event — so you don't repeat common context:

```csharp
config
    .Enrich.FromLogContext()              // properties pushed via LogContext (e.g., per-request)
    .Enrich.WithMachineName()             // adds MachineName to every event
    .Enrich.WithThreadId()
    .Enrich.WithProperty("Service", "OrderApi")
    .Enrich.WithSpan();                    // adds TraceId/SpanId (correlation with traces!)

// Push per-operation context (like a scope) — every log within carries CorrelationId
using (LogContext.PushProperty("CorrelationId", correlationId)) {
    logger.LogInformation("processing");   // automatically tagged with CorrelationId
}
```

Enrichment is Serilog's standout feature: machine name, environment, service name, thread, **trace id** (for correlation with traces — [06-Activities.md](06-Activities.md)), and per-request context all attach automatically. This makes every log self-describing and correlatable without manually adding the same fields everywhere. `LogContext.PushProperty` is Serilog's richer take on log scopes.

---

## Sinks — output destinations

A **sink** is where events are written. Serilog has a vast ecosystem:

| Sink | Destination |
|---|---|
| `WriteTo.Console` | stdout (text or JSON) |
| `WriteTo.File` | rolling files |
| `WriteTo.Seq` | Seq structured log server |
| `WriteTo.Elasticsearch` / `OpenSearch` | ELK |
| `WriteTo.ApplicationInsights` | Azure App Insights |
| `WriteTo.GrafanaLoki` | Loki |
| `WriteTo.OpenTelemetry` | OTLP → any OTel backend |

You can write to **multiple sinks** simultaneously (console + Seq + a file), each with its own level/format:

```csharp
.WriteTo.Console()                                            // everything to console (dev)
.WriteTo.Seq("http://seq:5341", restrictedToMinimumLevel: LogEventLevel.Information)
.WriteTo.File("logs/errors.txt", restrictedToMinimumLevel: LogEventLevel.Error)   // errors to a file
```

The huge sink ecosystem (and per-sink configuration) is why Serilog is popular — route the same structured events to wherever you need, in the format each backend expects.

---

## Serilog vs built-in vs OpenTelemetry

| | Built-in `ILogger` providers | Serilog | OpenTelemetry logging |
|---|---|---|---|
| Structured logging | yes (basic) | **yes (rich, destructuring)** | yes |
| Enrichment | scopes | **rich enrichers** | resource/scope attributes |
| Sinks/destinations | few built-in | **huge ecosystem** | OTLP → any backend |
| Vendor neutrality | provider-specific | sink-specific | **fully neutral** |

- **Built-in** providers ([03-LogProviders.md](03-LogProviders.md)) — fine for simple needs (console, basic structured JSON).
- **Serilog** — when you want rich enrichment and a specific sink ecosystem; mature and widely used.
- **OpenTelemetry logging** ([07-OpenTelemetry.md](07-OpenTelemetry.md)) — the converging, vendor-neutral standard that unifies logs with metrics/traces.

These aren't mutually exclusive — Serilog has an **OpenTelemetry sink** (`WriteTo.OpenTelemetry`), so you can use Serilog's enrichment *and* export via OTel. A common modern setup: Serilog for rich structured logging + enrichment, exporting through OTel to your backend.

---

## Common gotchas

### Not destructuring complex objects

`{Order}` calls `ToString()` (a flat string); `{@Order}` captures the object's properties as structured data. Use `@` when you want queryable fields from an object.

### Logging to files in containers

Container files are ephemeral and uncollected. Use a console/OTel sink (stdout) in containers; file sinks suit traditional servers.

### No trace correlation

Without an enricher adding TraceId/SpanId (`Enrich.WithSpan()` or OTel integration), logs can't be linked to traces. Add it for cross-pillar correlation.

### Over-enriching

Attaching expensive or high-cardinality enrichers to every event adds cost. Enrich with cheap, useful context (machine, service, trace id), not heavy computations.

### Logging secrets via destructuring

`{@User}` serializes *all* of an object's properties — including passwords/PII if present. Destructure deliberately; exclude sensitive fields.

### Vendor lock-in via a single sink

Coupling to one vendor's sink ties you to it. Use the OpenTelemetry sink (or `ILogger` + OTel) for neutrality.

---

## Summary

- **Serilog** plugs into `ILogger` and adds rich **structured logging** (first-class properties, `{@obj}` destructuring), **enrichment** (auto-attach machine/service/thread/**trace id**/per-request context to every event), and a huge **sink** ecosystem.
- **Enrichment** is its standout feature — self-describing, correlatable logs without repeating fields; `LogContext.PushProperty` adds per-operation context.
- Write to **multiple sinks** (console/Seq/ELK/Loki/App Insights/OTLP) simultaneously, each with its own level/format.
- It complements OpenTelemetry (use the **OpenTelemetry sink** to get Serilog's enrichment *and* OTel export); add **trace correlation** (`Enrich.WithSpan()`), log to **stdout/OTel** in containers, and avoid destructuring secrets.

→ Next: [05-Metrics.md](05-Metrics.md)
