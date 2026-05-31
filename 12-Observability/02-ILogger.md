# ILogger & Structured Logging

## The logging pillar

`ILogger<T>` is .NET's logging abstraction and the foundation of the **logs** pillar. It's introduced in [Ch03 §09](../03-HostingAndDI/09-LoggingPrimer.md) (everyday usage); this file goes deeper on **structured logging**, scopes, the `[LoggerMessage]` source generator, and how logs fit observability (correlation with traces).

```csharp
public class OrderService(ILogger<OrderService> logger) {
    public async Task PlaceAsync(Order order) {
        logger.LogInformation("Placing order {OrderId} for {Customer}", order.Id, order.Customer);
        // ...
    }
}
```

Inject `ILogger<T>`; the `T` is the **category** for filtering. `ILogger` is the logs API that OpenTelemetry collects ([07-OpenTelemetry.md](07-OpenTelemetry.md)) — so logging through it is OTel-ready.

---

## Structured logging — the non-negotiable

The single most important logging practice: **use message templates with named placeholders, not string interpolation.**

```csharp
// ✓ STRUCTURED — OrderId and Amount become named, queryable fields
logger.LogInformation("Order {OrderId} placed for {Amount:C}", order.Id, order.Total);

// ✗ INTERPOLATION — collapses to a flat string; no structured fields; formats even when filtered out
logger.LogInformation($"Order {order.Id} placed for {order.Total:C}");
```

With the template form, the placeholders (`OrderId`, `Amount`) are captured as **structured key/value pairs**. A structured logging backend (Seq, Elasticsearch, Loki, Application Insights) stores them as searchable fields — so you can query "all logs where `OrderId = 42`" or "errors grouped by `Customer`" across services. Interpolation throws that away (it's just a string) and wastes CPU formatting logs that may be filtered out by level. **Always use named placeholders + arguments** — this is what makes logs an observability asset rather than a text dump.

---

## Log levels & filtering

```csharp
logger.LogTrace(...);        // 0 — firehose (dev only)
logger.LogDebug(...);        // 1 — diagnostics
logger.LogInformation(...);  // 2 — significant events
logger.LogWarning(...);      // 3 — handled-but-notable
logger.LogError(ex, ...);    // 4 — failures (always pass the exception!)
logger.LogCritical(ex, ...); // 5 — catastrophic
```

Levels are filtered **per category** in config, *before* the message is formatted (so a filtered-out `LogDebug` costs ~nothing — another reason templates beat interpolation):

```json
{ "Logging": { "LogLevel": {
    "Default": "Information",
    "Microsoft.AspNetCore": "Warning",
    "MyApp.OrderService": "Debug"
} } }
```

Choose levels deliberately: Information for significant business events, Warning for handled-but-notable issues, Error/Critical for failures (always pass the **exception object** — it preserves the stack and is captured structurally). Don't flood Information with routine noise or bury real errors at Debug.

---

## Log scopes — attaching context

A **scope** attaches structured properties to *every* log written within it — invaluable for correlation:

```csharp
using (logger.BeginScope("Processing order {OrderId} for {TenantId}", order.Id, tenantId)) {
    logger.LogInformation("validating");    // automatically carries OrderId + TenantId
    logger.LogInformation("charging");      // ...so does this, without repeating them
}
```

Scopes flow via `AsyncLocal` ([Ch02 §12](../02-BCL/12-Threading.md)) — so every log in the operation (across awaits) shares the context. ASP.NET Core automatically creates a per-request scope with the request id, **and correlates logs with the current trace** (the `Activity` — [06-Activities.md](06-Activities.md)): logs written during a request carry its **TraceId/SpanId**, so you can pivot from a trace to its logs and back. This correlation across pillars is a core observability capability.

---

## `[LoggerMessage]` — high-performance logging

For hot paths, the `[LoggerMessage]` source generator emits **allocation-free**, pre-compiled logging methods (no boxing of value-type args, no per-call template parsing):

```csharp
public partial class OrderService {
    [LoggerMessage(Level = LogLevel.Information, Message = "Order {OrderId} placed for {Amount}")]
    private partial void LogOrderPlaced(int orderId, decimal amount);

    public void Place(Order o) => LogOrderPlaced(o.Id, o.Total);   // fast, structured, no boxing
}
```

The generator (CSharpBook Ch12 §05) produces a method that checks the level first (skipping all work if filtered out) and avoids the allocations the generic `Log*` extension methods incur (boxing `int`/`decimal` args, parsing the template each call). Use `[LoggerMessage]` for frequently-hit log statements — it's the recommended high-performance, AOT-friendly logging approach.

---

## Logs in the observability picture

Logs are the **detailed-event** pillar ([01-WhatIsObservability.md](01-WhatIsObservability.md)). To make them useful for observability:
- **Structure them** (named fields) so they're queryable.
- **Correlate them with traces** (the per-request scope adds TraceId/SpanId automatically) so you can jump from a slow trace to its logs.
- **Route them** to a structured backend ([03-LogProviders.md](03-LogProviders.md), [04-Serilog.md](04-Serilog.md)) — console JSON in containers, Seq/ELK/App Insights/Loki in production, ideally via OpenTelemetry ([07-OpenTelemetry.md](07-OpenTelemetry.md)).
- **Control volume** — log meaningful events, use levels, sample if needed; logs are the most expensive pillar at scale.
- **Never log secrets/PII** — logs are widely accessible and retained.

---

## Common gotchas

### String interpolation instead of templates

Loses structured fields and formats even when filtered out. Use `"...{Named}...", arg` — the cardinal logging rule.

### Not passing the exception

`LogError("failed: " + ex.Message)` loses the stack trace. Pass the exception object: `LogError(ex, "failed for {Id}", id)`.

### Logging the same error at every layer

Re-logging an exception as it bubbles floods logs with duplicates. Log once, at the handling boundary (CSharpBook Ch17 §13).

### High-volume logging without levels/sampling

Logging everything at Information at scale is expensive and drowns the signal. Use levels, log meaningful events, sample if needed.

### Logging secrets/PII

Don't log passwords, tokens, or full PII — logs are broadly accessible. Redact or omit.

### Allocations on hot log paths

The generic `Log*` methods box value-type args and parse templates per call. Use `[LoggerMessage]` for frequently-hit logs.

---

## Summary

- **`ILogger<T>`** is the logs pillar (and the OTel logs API); inject it with `T` as the filter category.
- **Use message templates with named placeholders** (`"...{OrderId}...", id`) — never interpolation — so logs become **structured, queryable fields** (the non-negotiable rule) and aren't formatted when filtered out.
- Choose **levels** deliberately (Information/Warning/Error, always pass the **exception**); filter per category in config.
- Use **scopes** (`BeginScope`) to attach correlation context across an operation; ASP.NET Core auto-correlates logs with the current **trace** (TraceId/SpanId) — pivot between logs and traces.
- Use **`[LoggerMessage]`** source-gen for allocation-free hot-path logging; route logs to a structured backend (via OTel), control volume, and never log secrets.

→ Next: [03-LogProviders.md](03-LogProviders.md)
