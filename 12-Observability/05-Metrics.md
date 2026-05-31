# Metrics

## The metrics pillar

Metrics are numeric measurements aggregated over time — requests/sec, error rate, p99 latency, queue depth, active connections — the cheap, always-on pillar for dashboards, alerting, and trends ([01-WhatIsObservability.md](01-WhatIsObservability.md)). .NET's metrics API is **`System.Diagnostics.Metrics`** (`Meter`, `Counter`, `Histogram`, gauges), which **is** the OpenTelemetry metrics API ([Ch02 §08](../02-BCL/08-Diagnostics.md)) — so instrumenting with it is OTel-ready.

```csharp
private static readonly Meter Meter = new("Shop.Orders", "1.0");
private static readonly Counter<long> OrdersPlaced = Meter.CreateCounter<long>("orders.placed", "{orders}");
private static readonly Histogram<double> OrderDuration = Meter.CreateHistogram<double>("orders.duration", "ms");

public async Task PlaceAsync(Order o) {
    var sw = Stopwatch.StartNew();
    await ProcessAsync(o);
    OrdersPlaced.Add(1, new KeyValuePair<string, object?>("region", o.Region));   // with a dimension
    OrderDuration.Record(sw.Elapsed.TotalMilliseconds);
}
```

> The instrument types and basics are in [Ch02 §08](../02-BCL/08-Diagnostics.md). This file focuses on metrics as an **observability** pillar: instrument types, dimensions, cardinality, and what to measure.

---

## Instrument types

| Instrument | Behavior | Use for |
|---|---|---|
| **`Counter<T>`** | monotonically increasing sum | totals: requests, errors, bytes, orders |
| **`UpDownCounter<T>`** | goes up and down | current count: active connections, items in flight |
| **`Histogram<T>`** | distribution of recorded values | latencies, sizes → percentiles (p50/p99) |
| **`ObservableCounter/Gauge/UpDownCounter`** | sampled via a callback | values you read on demand: memory, queue depth, pool size |

- **Counters** for "how many total" (rate is derived by the backend).
- **Histograms** for distributions — essential for latency (an *average* latency hides the slow tail; a histogram gives **percentiles** like p99, which is what users feel).
- **Observable** instruments sample a value via a callback when collected (for gauges like queue depth or memory you don't increment per event):

```csharp
Meter.CreateObservableGauge("orders.queue.depth", () => _queue.Count);   // sampled on collection
```

Create `Meter`s and instruments **once** (static), like sources ([Ch02 §08](../02-BCL/08-Diagnostics.md)); they cost ~nothing when no collector is listening.

---

## Dimensions (tags) — slicing metrics

A metric can carry **dimensions** (tags) so you can slice it — error rate *by* endpoint, latency *by* region:

```csharp
OrdersPlaced.Add(1,
    new("region", order.Region),
    new("payment_method", order.PaymentMethod));
// → query "orders.placed by region" or "by payment_method" in the backend
```

Dimensions turn one metric into a queryable cube (by region, status, route, etc.). But this is also the source of the **cardinality** trap.

---

## Cardinality — the cardinal metrics pitfall

**Cardinality** is the number of distinct dimension-value combinations — and it's the #1 metrics mistake. Each unique combination creates a separate **time series** in the backend. Low-cardinality dimensions (region: ~5 values, status: ~10) are fine. **High-cardinality** dimensions (user id, order id, request id — potentially millions of values) explode into millions of series, overwhelming and bankrupting the metrics backend:

```csharp
// ✗ — user_id is HIGH cardinality → millions of time series → backend overload/cost explosion
OrdersPlaced.Add(1, new("user_id", order.UserId));

// ✓ — low-cardinality dimensions only
OrdersPlaced.Add(1, new("region", order.Region), new("tier", order.CustomerTier));
```

**Rule: only use low-cardinality (bounded, small-set) values as metric dimensions** — region, status code, route *template* (`/orders/{id}`, not `/orders/42`), method, tenant tier. Put high-cardinality data (specific ids) in **logs** or **trace attributes** instead, where per-event detail belongs ([01-WhatIsObservability.md](01-WhatIsObservability.md)). Exploding cardinality is how teams get surprise observability bills and crashed metric backends.

---

## What to measure — the RED/USE methods

Two well-known frameworks for *which* metrics to collect:

- **RED** (for request-driven services): **R**ate (requests/sec), **E**rrors (failure rate), **D**uration (latency distribution). The core SLIs for an API.
- **USE** (for resources): **U**tilization, **S**aturation, **E**rrors — for queues, connection pools, thread pools, CPU/memory.

```
Service metrics (RED):   request rate, error rate, latency histogram (per route/status)
Resource metrics (USE):  pool utilization, queue saturation/depth, GC time, thread-pool queue
Business metrics:        orders placed, revenue, signups (low-cardinality dimensions)
```

Measure RED for your services (rate/errors/duration — the basis of SLOs and alerts), USE for resources (catch saturation before it cascades — [Ch11 §01](../11-Resilience/01-WhyResilience.md)), and meaningful business metrics. Much of RED/USE is provided automatically by built-in instrumentation ([08-AspNetTelemetry.md](08-AspNetTelemetry.md)).

---

## Viewing metrics

```bash
dotnet-counters monitor -p <pid> Shop.Orders Microsoft.AspNetCore.Hosting   # live, near-zero overhead
```

Locally, **`dotnet-counters`** ([09-Profiling.md](09-Profiling.md)) shows live metrics from any `Meter`. In production, the **OpenTelemetry SDK** ([07-OpenTelemetry.md](07-OpenTelemetry.md)) collects and exports metrics to Prometheus/Grafana, Application Insights, Datadog, etc., where you build dashboards and alerts (e.g., alert when p99 > 500ms, or error rate > 1%).

---

## Common gotchas

### High-cardinality dimensions

Tagging metrics with user/request/order ids explodes time-series count → overwhelmed/expensive backend. Use only **low-cardinality** dimensions; put ids in logs/traces.

### Averaging latency instead of histograms

An average latency hides the slow tail (a few very-slow requests). Use a **`Histogram`** and look at **percentiles** (p95/p99) — that's what users experience.

### Creating `Meter`/instruments per call

Wasteful and fragments the metric. Create them **once** (static), like sources.

### Using metrics for per-event detail

Metrics are aggregates — they tell you *that* something changed, not *which* request. For per-event detail, use logs/traces.

### Route metrics with raw paths

Tagging by the actual URL (`/orders/42`) is high-cardinality. Use the **route template** (`/orders/{id}`).

### No business metrics

Only infra metrics miss what matters (orders, revenue). Add meaningful business metrics (with low-cardinality dimensions).

---

## Summary

- **Metrics** (`System.Diagnostics.Metrics`: `Meter` + `Counter`/`Histogram`/gauges — also the OTel API) are cheap, aggregated numbers over time for dashboards, alerts, and trends.
- Use **counters** for totals, **histograms** for distributions (→ **percentiles** like p99, not averages — averages hide the tail), **observable** instruments for sampled gauges; create them **once** (static).
- **Dimensions** slice metrics — but **cardinality is the #1 pitfall**: only use **low-cardinality** dimensions (region, status, route template, tier); high-cardinality ids (user/request id) explode time series → put those in **logs/traces**.
- Measure **RED** (rate/errors/duration) for services and **USE** (utilization/saturation/errors) for resources, plus business metrics; view live with `dotnet-counters`, export via **OpenTelemetry** to your backend.

→ Next: [06-Activities.md](06-Activities.md)
