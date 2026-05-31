# Logging Primer

## `ILogger<T>` — the logging abstraction

The host wires up logging so you can inject **`ILogger<T>`** anywhere and write structured, level-filtered, multi-destination logs. This primer covers everyday usage; **Chapter 12 (Observability)** covers the full pipeline (providers, OpenTelemetry, Serilog, distributed tracing).

```csharp
public class OrderService(ILogger<OrderService> logger) {
    public void Place(Order order) {
        logger.LogInformation("Placing order {OrderId} for {Customer}", order.Id, order.Customer);
        try { /* ... */ }
        catch (Exception ex) {
            logger.LogError(ex, "Failed to place order {OrderId}", order.Id);   // pass the exception!
            throw;
        }
    }
}
```

`ILogger<T>` is registered by the host; the `T` becomes the log **category** (typically the class name), so you can filter by namespace/type. Just inject it — no setup per class.

---

## Structured logging — use placeholders, not interpolation

The single most important logging habit: **use message templates with named placeholders**, not string interpolation.

```csharp
// ✓ — structured: "OrderId" and "Amount" become queryable fields, not just text
logger.LogInformation("Order {OrderId} placed for {Amount:C}", order.Id, order.Total);

// ✗ — interpolation: collapses to a flat string; no structured fields; formats even when filtered out
logger.LogInformation($"Order {order.Id} placed for {order.Total:C}");
```

With the template form, structured-logging backends (Seq, Elasticsearch, Application Insights) capture `OrderId` and `Amount` as **named, searchable properties** — you can query "all logs where OrderId = 42" across services. Interpolation throws that away (just a string) and pays formatting cost even when the level is filtered out. Always use `{Named}` placeholders with arguments.

---

## Log levels

```csharp
logger.LogTrace("very detailed diagnostic");        // Trace   (0) — firehose, dev only
logger.LogDebug("debugging info");                   // Debug   (1) — dev/diagnostics
logger.LogInformation("normal operation");           // Information (2) — significant events
logger.LogWarning("recoverable/notable issue");      // Warning (3) — something off but handled
logger.LogError(ex, "operation failed");             // Error   (4) — failures needing attention
logger.LogCritical(ex, "system unusable");           // Critical (5) — catastrophic
```

Choose levels deliberately: **Information** for significant business events, **Warning** for handled-but-notable issues, **Error/Critical** for failures (always pass the **exception object** — preserves the stack). Trace/Debug are verbose and usually filtered out in production. Don't log routine success at Information if it floods; don't bury real errors at Debug.

---

## Level filtering & configuration

Logging is filtered by level **per category**, configured in `appsettings.json`:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",        // quiet the framework
      "Microsoft.EntityFrameworkCore": "Warning",
      "MyApp.OrderService": "Debug"              // verbose for one component
    }
  }
}
```

Filtering happens **before** the message is formatted, so a filtered-out `LogDebug` costs almost nothing (another reason to use templates, not interpolation — interpolation formats regardless). You can tune verbosity per namespace/type without code changes.

---

## Log scopes — attach context to a group of logs

A **scope** attaches structured properties to every log written within it — invaluable for correlating logs of one operation:

```csharp
using (logger.BeginScope("Processing order {OrderId} for {TenantId}", order.Id, tenantId)) {
    logger.LogInformation("validated");      // both lines carry OrderId + TenantId
    logger.LogInformation("charged");        // automatically, without repeating them
}
```

Scopes flow via `AsyncLocal` ([Ch02 §12](../02-BCL/12-Threading.md)), so all logs in the operation (across awaits) share the context. ASP.NET Core automatically creates a scope per request with the request ID, and correlates with the current trace (`Activity` — [Ch02 §08](../02-BCL/08-Diagnostics.md)). Use scopes for request/operation/tenant correlation instead of repeating IDs in every message.

---

## High-performance logging: `LoggerMessage` source generator

For hot paths, the `[LoggerMessage]` source generator emits allocation-free, pre-compiled logging methods (no boxing, no template parsing per call):

```csharp
public partial class OrderService {
    [LoggerMessage(Level = LogLevel.Information, Message = "Order {OrderId} placed for {Amount}")]
    private partial void LogOrderPlaced(int orderId, decimal amount);

    public void Place(Order o) => LogOrderPlaced(o.Id, o.Total);   // fast, structured, no boxing
}
```

This is the recommended approach for frequently-hit log statements — it avoids the per-call allocation/formatting of the generic `Log*` extension methods and is AOT-friendly. (Source generators: CSharpBook Ch12 §05.)

---

## Providers (preview)

`ILogger` writes to configured **providers** — Console (default in dev), Debug, EventSource, and many third-party (Serilog, Seq, Application Insights, OpenTelemetry):

```csharp
builder.Logging.ClearProviders();
builder.Logging.AddConsole();
builder.Logging.AddJsonConsole();   // structured JSON output (good for log aggregators)
```

In production you typically route to a structured sink (Serilog → Seq/Elasticsearch, or OpenTelemetry → your APM). The provider/exporter pipeline is [Chapter 12](../12-Observability/README.md). The point here: your code logs through `ILogger`; *where* it goes is configuration.

---

## Common gotchas

### String interpolation instead of templates

`LogInformation($"...{x}...")` loses structured fields and formats even when filtered out. Use `LogInformation("...{X}...", x)`.

### Not passing the exception

`logger.LogError("failed: " + ex.Message)` loses the stack trace. Pass the exception object: `logger.LogError(ex, "failed for {Id}", id)`.

### Logging the same error at every layer

Re-logging an exception as it bubbles floods logs with duplicates. Log once, at the boundary that handles it (CSharpBook Ch17 §13).

### Wrong levels

Routine success at Information floods; real errors at Debug get filtered out and missed. Match level to significance.

### Logging secrets/PII

Don't log passwords, tokens, full PII. Redact or omit — logs are widely accessible and retained.

### Allocations on hot log paths

The generic `Log*` methods box value-type args and parse templates per call. Use `[LoggerMessage]` source-gen for high-frequency logging.

---

## Summary

- Inject **`ILogger<T>`** anywhere (the host registers it); `T` is the log **category** for filtering.
- **Use message templates with named placeholders** (`"...{OrderId}..."` + args), never string interpolation — preserves structured, queryable fields and avoids formatting filtered-out logs.
- Choose **levels** deliberately (Information for events, Warning for notable, Error/Critical for failures — always pass the **exception**); filter per category in config.
- Use **scopes** (`BeginScope`) to attach correlation context (request/order/tenant IDs) to a group of logs; they flow across awaits and correlate with traces.
- For hot paths use the **`[LoggerMessage]`** source generator (allocation-free, AOT-friendly).
- Don't log secrets; log errors once at the handling boundary. Full observability pipeline: [Chapter 12](../12-Observability/README.md).

→ Next: [10-Validation.md](10-Validation.md)
