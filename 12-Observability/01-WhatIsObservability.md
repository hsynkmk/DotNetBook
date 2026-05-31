# What Is Observability

## Knowing what your system is doing

**Observability** is the ability to understand a system's internal state from its external outputs — to answer "what is it doing right now?", "why is it slow?", "what failed and why?" without attaching a debugger. In distributed systems where you can't step through code across services, observability is how you operate, debug, and trust production. It rests on **three pillars**: **logs**, **metrics**, and **traces**.

```
Logs    — discrete events ("order 42 failed validation")        → what happened, in detail
Metrics — aggregated numbers over time (requests/sec, p99)       → how the system is performing, trends
Traces  — the path of one request across services + timing       → where time went / where it failed
```

Each answers a different question; together they let you detect, diagnose, and understand problems.

---

## Monitoring vs observability

- **Monitoring** — watching **known** signals for **known** problems ("alert if CPU > 80%", "alert if error rate spikes"). You decide in advance what to watch.
- **Observability** — being able to ask **arbitrary, new** questions about the system's behavior, including ones you didn't anticipate ("why are *these specific* requests from *this* region slow *today*?").

Monitoring tells you *that* something is wrong; observability lets you figure out *why* — including for novel problems you didn't predict. A well-instrumented system supports both: dashboards/alerts (monitoring) *and* the rich, queryable data (logs/metrics/traces with dimensions) to investigate the unexpected (observability).

---

## The three pillars

### Logs — discrete events ([02-ILogger.md](02-ILogger.md))

A log is a timestamped record of a specific event: "user logged in", "order failed: insufficient stock", an exception with a stack trace. **Structured** logs (named key/value fields, not just text) are queryable: "show all logs where `OrderId = 42`". Logs are detailed and high-fidelity per event but high-volume and expensive to store/search at scale.

```csharp
logger.LogError(ex, "Order {OrderId} failed for {Customer}", order.Id, order.Customer);
//   structured: OrderId and Customer become searchable fields
```

**Use logs for**: detailed event records, errors with context, audit trails, debugging specific occurrences.

### Metrics — aggregated measurements ([05-Metrics.md](05-Metrics.md))

A metric is a numeric measurement aggregated over time: requests per second, error rate, p50/p99 latency, queue depth, active connections. Metrics are **cheap** (aggregated, not per-event), ideal for dashboards, alerting, and spotting trends — but they're aggregates, so they tell you *that* latency rose, not *which* request was slow.

```csharp
ordersPlaced.Add(1, new("region", "us"));        // a counter with a dimension
orderDuration.Record(elapsed.TotalMilliseconds);  // a histogram → percentiles
```

**Use metrics for**: performance trends, SLIs/SLOs, alerting thresholds, capacity planning, dashboards.

### Traces — request flow across services ([06-Activities.md](06-Activities.md))

A trace follows **one request** as it flows through the system — across services, databases, and queues — recording each step (**span**) with timing and parent/child relationships. It shows *where time went* and *where a failure occurred* in a distributed call.

```
Trace for request abc123:
  [API: handle order            240ms] ────────────────────────────
    [→ Auth service             15ms]  ──
    [→ DB: load customer        20ms]    ──
    [→ Inventory service        180ms]      ──────────────  ← the slow span!
    [→ Publish OrderPlaced      8ms]                       ─
```

**Use traces for**: diagnosing latency ("which service/span was slow?"), following a request across service boundaries, understanding dependencies, root-causing distributed failures.

---

## How the pillars work together

The pillars are complementary — a typical investigation uses all three:

```
1. METRICS alert: p99 latency on /checkout spiked at 14:00 (you know SOMETHING is wrong)
2. TRACES show: the slow checkouts spend 2s in the "inventory" span (you know WHERE)
3. LOGS reveal: inventory service logs "DB connection pool exhausted" at 14:00 (you know WHY)
```

Metrics detect the problem (broad, cheap, always-on), traces localize it (which service/operation), logs explain it (the specific error/context). Instrument all three and **correlate** them (a shared **trace id** links a trace to its logs — [02-ILogger.md](02-ILogger.md), [06-Activities.md](06-Activities.md)) so you can pivot between them. Relying on only one leaves blind spots: metrics without traces can't localize; logs without metrics can't show trends; traces without logs lack detail.

---

## OpenTelemetry — the unifying standard

The industry has converged on **OpenTelemetry (OTel)** — a vendor-neutral, open standard for generating and collecting all three pillars, with broad backend support (Jaeger, Prometheus, Grafana, Application Insights, Datadog, etc.). Crucially, **.NET's built-in observability primitives *are* the OTel APIs**: `ILogger` (logs), `Meter` (metrics — [05-Metrics.md](05-Metrics.md)), and `ActivitySource` (traces — [06-Activities.md](06-Activities.md)). So instrumenting with the BCL primitives is automatically OTel-compatible — the OTel SDK ([07-OpenTelemetry.md](07-OpenTelemetry.md)) just collects and exports them to any backend. Instrument once, send anywhere, no vendor lock-in.

---

## Cost and signal

Observability has real costs (storage, ingestion, processing) — you can't log everything at full fidelity at scale. The trade-offs:
- **Logs** — most detailed, most expensive. Log meaningful events; control verbosity by level ([02-ILogger.md](02-ILogger.md)); don't log noise or secrets.
- **Metrics** — cheap (aggregated) — but **cardinality** matters: a metric tagged with unbounded values (user ids) explodes into millions of series and overwhelms the backend ([05-Metrics.md](05-Metrics.md)). Use low-cardinality dimensions.
- **Traces** — moderate cost — often **sampled** (record a fraction of requests) at high volume to control cost while keeping representative data.

Good observability is about capturing the **right signal** (high-value events, meaningful metrics, representative traces), not maximum volume.

---

## Common gotchas

### Relying on one pillar

Metrics alone can't tell you *why*; logs alone can't show trends; traces alone lack detail. Use all three and **correlate** them (shared trace id).

### Unstructured logs

Plain-text logs ("Order 42 failed") aren't queryable. Use **structured** logging (named fields) so you can search/aggregate ([02-ILogger.md](02-ILogger.md)).

### High-cardinality metric tags

Tagging metrics with user/request ids creates millions of series, crippling the backend. Use low-cardinality dimensions (region, status, route template) ([05-Metrics.md](05-Metrics.md)).

### No correlation between pillars

If logs, traces, and metrics can't be linked (no shared trace id), you can't pivot from "metric spiked" to "this trace" to "this log". Ensure correlation flows ([06-Activities.md](06-Activities.md)).

### Confusing monitoring with observability

Pre-defined dashboards/alerts (monitoring) catch known problems; you also need rich, queryable data (observability) to investigate the unexpected.

### Logging everything (or nothing)

Logging at full verbosity at scale is expensive and noisy; logging too little leaves you blind. Capture the right signal — meaningful events, the right level, representative traces.

---

## Summary

- **Observability** = understanding a system's internal state from external outputs, to answer "what/why" in production without a debugger — essential for distributed systems.
- Three pillars: **logs** (discrete, detailed events — *what happened*), **metrics** (cheap aggregates over time — *how it's performing, trends*), **traces** (one request's path across services — *where time went / where it failed*).
- They're **complementary**: metrics **detect** (alert), traces **localize** (which span), logs **explain** (the error) — instrument all three and **correlate** via a shared trace id.
- **OpenTelemetry** is the converged industry standard; .NET's built-in primitives (`ILogger`/`Meter`/`ActivitySource`) **are** the OTel APIs — instrument once, export anywhere.
- Mind **cost/signal**: structured logs (not noise/secrets), low-cardinality metrics, sampled traces — capture the *right* signal, not maximum volume. **Monitoring** watches known problems; **observability** lets you investigate unknown ones.

→ Next: [02-ILogger.md](02-ILogger.md)
