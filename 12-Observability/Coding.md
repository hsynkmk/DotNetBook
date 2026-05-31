# Chapter 12 — Observability — Coding Problems

Instrument a service with structured logging, metrics, and traces, wire up OpenTelemetry, and use diagnostic tools.

---

## Problem 1: Structured logging done right

Convert string-interpolated logs to structured logging with correct levels.

<details><summary>Solution</summary>

```csharp
// ✗ Before — interpolation (no structured fields, formats even when filtered)
logger.LogInformation($"Order {order.Id} placed for {order.Customer} at {DateTime.UtcNow}");
logger.LogError("Failed: " + ex.Message);

// ✓ After — message templates (named, queryable fields) + exception object
logger.LogInformation("Order {OrderId} placed for {Customer}", order.Id, order.Customer);
logger.LogError(ex, "Order {OrderId} failed", order.Id);   // pass the exception → preserves the stack
```

Named placeholders → structured fields (search "OrderId = 42"); pass the exception object (not `.Message`) so the stack is captured. ([02-ILogger.md](02-ILogger.md).)

</details>

---

## Problem 2: Log scope for correlation

Attach order/tenant context to all logs in an operation.

<details><summary>Solution</summary>

```csharp
public async Task ProcessAsync(Order order, string tenantId) {
    using (logger.BeginScope("Order {OrderId} for tenant {TenantId}", order.Id, tenantId)) {
        logger.LogInformation("validating");   // carries OrderId + TenantId (and the request's TraceId)
        await ValidateAsync(order);
        logger.LogInformation("charging");      // also carries them — no repetition
    }
}
```

`BeginScope` adds context to every log within it (flowing across awaits); ASP.NET Core also adds the TraceId/SpanId automatically, correlating these logs with the request's trace. ([02-ILogger.md](02-ILogger.md), [06-Activities.md](06-Activities.md).)

</details>

---

## Problem 3: High-performance logging with [LoggerMessage]

Replace a hot-path log with an allocation-free source-generated method.

<details><summary>Solution</summary>

```csharp
public partial class OrderService(ILogger<OrderService> logger) {
    [LoggerMessage(Level = LogLevel.Information, Message = "Order {OrderId} placed for {Amount}")]
    private partial void LogOrderPlaced(int orderId, decimal amount);

    [LoggerMessage(Level = LogLevel.Warning, Message = "Order {OrderId} retry {Attempt}")]
    private partial void LogRetry(int orderId, int attempt);

    public void Place(Order o) => LogOrderPlaced(o.Id, o.Total);   // no boxing, no template parse, level-checked
}
```

Source-gen logging avoids boxing the `int`/`decimal` args and parsing the template per call — ideal for frequently-hit logs. ([02-ILogger.md](02-ILogger.md).)

</details>

---

## Problem 4: Custom metrics (counter + histogram)

Add business metrics for order throughput and latency.

<details><summary>Solution</summary>

```csharp
public class OrderMetrics {
    private readonly Counter<long> _placed;
    private readonly Histogram<double> _duration;
    public OrderMetrics(IMeterFactory factory) {
        var meter = factory.Create("Shop.Orders");
        _placed = meter.CreateCounter<long>("orders.placed", "{orders}");
        _duration = meter.CreateHistogram<double>("orders.duration", "ms");
    }
    public void RecordPlaced(string region) =>
        _placed.Add(1, new KeyValuePair<string, object?>("region", region));   // LOW-cardinality dimension
    public void RecordDuration(double ms) => _duration.Record(ms);
}
builder.Services.AddSingleton<OrderMetrics>();
```

Counter for throughput, histogram for latency (→ percentiles). Dimension by `region` (low-cardinality) — **not** order/user id (would explode series). Use `IMeterFactory` for DI-friendly meter creation. ([05-Metrics.md](05-Metrics.md).)

</details>

---

## Problem 5: Custom trace spans

Add a business span with tags and status around an operation.

<details><summary>Solution</summary>

```csharp
public class OrderService {
    private static readonly ActivitySource Source = new("Shop.Orders");

    public async Task<Order> PlaceAsync(OrderRequest req, CancellationToken ct) {
        using var activity = Source.StartActivity("PlaceOrder");   // null if no listener → near-free
        activity?.SetTag("order.customer", req.CustomerId);         // tags CAN be high-cardinality (ids ok here)
        activity?.SetTag("order.itemCount", req.Items.Count);
        try {
            var order = await ProcessAsync(req, ct);                // child spans (HTTP/DB) nest automatically
            activity?.SetTag("order.id", order.Id);
            activity?.SetStatus(ActivityStatusCode.Ok);
            return order;
        } catch (Exception ex) {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);   // mark failure → stands out in traces
            throw;
        }
    }
}
```

A business span with tags (ids fine on spans) and status; child HTTP/DB spans nest under it automatically. Create the source once (static). ([06-Activities.md](06-Activities.md).)

</details>

---

## Problem 6: Wire up OpenTelemetry (all three pillars)

Configure OTel for traces, metrics, and logs with auto + custom instrumentation.

<details><summary>Solution</summary>

```csharp
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService("OrderApi", serviceVersion: "1.0")
        .AddAttributes([new("deployment.environment", builder.Environment.EnvironmentName)]))
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()                 // incoming requests (auto)
        .AddHttpClientInstrumentation()                  // outbound HTTP + context propagation (auto)
        .AddEntityFrameworkCoreInstrumentation()         // DB queries (auto)
        .AddSource("Shop.Orders")                        // YOUR spans — must register
        .SetSampler(new TraceIdRatioBasedSampler(0.1))   // sample 10%
        .AddOtlpExporter())
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()                  // RED metrics (auto)
        .AddRuntimeInstrumentation()                     // GC, thread pool (auto)
        .AddMeter("Shop.Orders")                         // YOUR metrics — must register
        .AddOtlpExporter());

builder.Logging.AddOpenTelemetry(o => { o.IncludeScopes = true; o.AddOtlpExporter(); });
```

Auto-instrumentation gives request/HTTP/DB traces + RED/runtime metrics for free; register your custom `Shop.Orders` source/meter; export everything via OTLP (→ a collector → any backend). All three pillars, correlated. ([07-OpenTelemetry.md](07-OpenTelemetry.md), [08-AspNetTelemetry.md](08-AspNetTelemetry.md).)

</details>

---

## Problem 7: Fix the cardinality bug

This metric crashes the backend. Fix it.

```csharp
ordersPlaced.Add(1, new("user_id", order.UserId), new("order_id", order.Id.ToString()));
```

<details><summary>Solution</summary>

```csharp
// ✗ — user_id and order_id are HIGH cardinality → millions of time series → backend overload/cost explosion

// ✓ — low-cardinality dimensions only
ordersPlaced.Add(1, new("region", order.Region), new("customer_tier", order.CustomerTier));

// Put the high-cardinality ids in a TRACE TAG or LOG instead (per-event detail belongs there):
activity?.SetTag("order.id", order.Id);
activity?.SetTag("order.user", order.UserId);
logger.LogInformation("Order {OrderId} for {UserId} placed", order.Id, order.UserId);
```

Metric dimensions must be low-cardinality (bounded sets). Ids (unbounded) belong in span tags and logs, where per-event detail is the point. ([05-Metrics.md](05-Metrics.md), [06-Activities.md](06-Activities.md).)

</details>

---

## Problem 8: Diagnose a production memory leak

Memory is climbing in production. Outline the steps.

<details><summary>Solution</summary>

```bash
# 1. Confirm it's GC/memory (not just working set) — live triage
dotnet-counters monitor -p <pid> System.Runtime
#    watch gc-heap-size climbing, gen-2 count rising, alloc-rate

# 2. Capture heap snapshots over time and compare to find what's GROWING
dotnet-gcdump collect -p <pid>      # snapshot 1
# ... wait while memory grows ...
dotnet-gcdump collect -p <pid>      # snapshot 2
#    open both in Visual Studio → diff → which type's instance count keeps increasing?

# 3. Find WHAT keeps the leaking objects alive (the unintended root)
dotnet-dump collect -p <pid>
dotnet-dump analyze core_dump
  > dumpheap -stat                  # objects by type/size — confirm the growing type
  > gcroot <address>                # the rooting reference (static event? never-evicting cache?)
```

`dotnet-counters` triages (it's GC/memory), `dotnet-gcdump` snapshots reveal *what's growing*, and `gcroot` finds *what keeps it alive* — typically an unintended root (static event handler, unbounded cache). ([09-Profiling.md](09-Profiling.md), CSharpBook Ch09 §13.)

</details>

---

## Problem 9: Liveness vs readiness health checks

Set up correct liveness and readiness endpoints for Kubernetes.

<details><summary>Solution</summary>

```csharp
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"])   // app-only
    .AddDbContextCheck<AppDbContext>("db", tags: ["ready"])                 // dependency
    .AddCheck<RedisHealthCheck>("redis", tags: ["ready"]);

var app = builder.Build();
// Liveness: ONLY app-self — a DB blip must NOT restart the pod (avoid restart storms)
app.MapHealthChecks("/health/live",  new() { Predicate = c => c.Tags.Contains("live") });
// Readiness: app + dependencies — stop routing traffic if a dependency is down (don't restart)
app.MapHealthChecks("/health/ready", new() { Predicate = c => c.Tags.Contains("ready") });
```

Liveness checks only the app (failing it restarts the pod — must not depend on the DB); readiness checks dependencies (failing it stops traffic without restarting). Conflating them causes restart storms. ([10-HealthChecks.md](10-HealthChecks.md), [Ch04 §16](../04-AspNetCore/16-HealthChecks.md).)

</details>

---

## Problem 10: A full observability investigation

Walk through diagnosing "checkout is slow" using all three pillars + tools.

<details><summary>Solution</summary>

```
1. METRICS (alert):   p99 of /checkout > 2s (was 200ms) at 14:00 — RED metric from AddAspNetCoreInstrumentation.
                      → SOMETHING is wrong on checkout.

2. TRACES (localize): open a slow /checkout trace → the "inventory-service" HTTP span takes 1.8s,
                      and within it the "DB: check stock" span takes 1.7s.
                      → the bottleneck is a DB query IN the inventory service.

3. LOGS (explain):    query inventory-service logs WHERE TraceId = <the slow trace's id> (correlation!)
                      → "Query timeout retrying; connection pool exhausted" at 14:00.
                      → root cause: pool exhaustion / a slow query.

4. PROFILE (confirm): dotnet-counters on inventory-service → thread-pool queue length high, or
                      dotnet-trace → the stock query dominates CPU/time. Confirm the culprit query.

5. FIX + VERIFY:      add an index / fix the query / raise the pool; verify p99 returns to baseline
                      (metrics) and the slow span shrinks (traces).
```

The pillars compose: **metrics detect**, **traces localize** (which span/service), **logs explain** (correlated by TraceId), **profiling confirms** at the code level. This is the canonical observability-driven investigation. ([01-WhatIsObservability.md](01-WhatIsObservability.md), [05](05-Metrics.md)/[06](06-Activities.md)/[09](09-Profiling.md).)

</details>

---

You can now instrument a .NET service across all three pillars — structured logging (with scopes and source-gen), low-cardinality metrics, and trace spans — wire it up with OpenTelemetry (auto + custom + export), avoid the cardinality trap, set correct health checks, and run an observability-driven investigation using metrics → traces → logs → profiling.

→ Back to [Chapter 12 README](README.md) · Next chapter: [Chapter 13 — Configuration](../13-Configuration/README.md)
