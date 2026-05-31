# Activities & Distributed Tracing

## The traces pillar

A **trace** follows one request across services, recording each step (**span**) with timing and parent/child relationships — the pillar that shows *where time went* and *where a failure occurred* in a distributed call ([01-WhatIsObservability.md](01-WhatIsObservability.md)). In .NET, a span is an **`Activity`**, created from an **`ActivitySource`** — which **is** the OpenTelemetry tracing API ([Ch02 §08](../02-BCL/08-Diagnostics.md)).

```csharp
private static readonly ActivitySource Source = new("Shop.Orders", "1.0");

public async Task<Order> PlaceOrderAsync(OrderRequest req) {
    using var activity = Source.StartActivity("PlaceOrder");   // start a span
    activity?.SetTag("order.customer", req.CustomerId);
    try {
        var order = await ProcessAsync(req);                    // child spans nest automatically
        activity?.SetTag("order.id", order.Id);
        activity?.SetStatus(ActivityStatusCode.Ok);
        return order;
    } catch (Exception ex) {
        activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
        throw;
    }
}
```

> The `Activity`/`ActivitySource` mechanics (null-when-no-listener, tags, events, status) are in [Ch02 §08](../02-BCL/08-Diagnostics.md). This file focuses on **distributed tracing**: spans, context propagation, and reading traces.

---

## Spans nest into a trace tree

Each `Activity` is a **span**; spans nest to form a **trace** — a tree showing the request's flow and where time was spent:

```
Trace (TraceId: abc123)
  span "PlaceOrder"                    [240ms]  (root)
    span "ValidateOrder"               [10ms]
    span "DB: load customer"           [20ms]
    span "HTTP: inventory service"     [180ms]   ← the slow span
      span "DB: check stock"           [170ms]   ← the actual culprit (in the inventory service)
    span "publish OrderPlaced"         [8ms]
```

A child `Activity` automatically links to the current one (`Activity.Current`, flowed via `AsyncLocal` — [Ch02 §12](../02-BCL/12-Threading.md)), building the tree. Each span has a **TraceId** (shared across the whole trace) and a **SpanId** (unique per span), plus timing, tags (attributes), events, and status. Reading the tree, you see exactly which operation dominated the latency or failed — the core value of tracing.

---

## Trace context propagation (across services)

The magic of *distributed* tracing: a trace spans **multiple services**. When service A calls service B, the **trace context** must propagate so B's spans join A's trace. This uses the **W3C Trace Context** standard — the `traceparent` HTTP header carrying the TraceId and the parent SpanId:

```
Service A (span X, trace abc) → HTTP call with header: traceparent: 00-abc...-X-01
Service B receives it → its spans are children of X, in trace abc → ONE end-to-end trace across A and B
```

In .NET this is **automatic**: `HttpClient` injects `traceparent` on outbound calls, and ASP.NET Core extracts it on inbound requests (when tracing is enabled — [07-OpenTelemetry.md](07-OpenTelemetry.md)) — so a request's trace flows seamlessly across service boundaries with no manual work. For non-HTTP transports (messaging), you propagate the context in message headers ([Ch07](../07-Messaging/README.md)) so a trace continues through a queue. This propagation is what makes a microservices request observable end-to-end.

---

## Tags, events, status

A span carries structured detail:

```csharp
activity?.SetTag("http.method", "POST");                          // attributes (queryable dimensions)
activity?.SetTag("order.total", order.Total);
activity?.AddEvent(new ActivityEvent("cache.miss"));               // timestamped points within the span
activity?.SetStatus(ActivityStatusCode.Error, "payment declined"); // span outcome
```

- **Tags** (attributes) — key/value detail on the span (HTTP method, status, db statement, ids). Unlike metrics, span tags **can be high-cardinality** (ids are useful here — [05-Metrics.md](05-Metrics.md)).
- **Events** — timestamped points within a span (a cache miss, a retry).
- **Status** — `Ok`/`Error` with a message; mark failed spans so they stand out in trace views.

This is where per-request detail (specific ids, the failing operation) belongs — complementing low-cardinality metrics.

---

## Correlation with logs

Because logs written during a request carry the current span's **TraceId/SpanId** (ASP.NET Core's per-request log scope adds them — [02-ILogger.md](02-ILogger.md)), you can **pivot between traces and logs**: from a slow/failed span in the trace view, jump to the logs for that exact TraceId to see the detailed error. This cross-pillar correlation (trace ↔ logs, via the shared TraceId) is a core observability capability — it's how you go from "this span failed" to "here's the exception and context."

```
Trace shows: span "inventory" failed (TraceId abc123)
   → query logs WHERE TraceId = abc123 → "DB connection pool exhausted" with full context
```

---

## Near-zero cost when uncollected

`StartActivity` returns **`null`** when no listener (tracing/OTel) is subscribed — so the `activity?.` calls no-op, and instrumentation costs ~nothing when not collecting ([Ch02 §08](../02-BCL/08-Diagnostics.md)). This means you can **instrument liberally** — add spans around meaningful operations — without worrying about overhead in environments where tracing is off. Create the `ActivitySource` **once** (static).

---

## Sampling

At high volume, recording *every* request's trace is expensive. **Sampling** records a representative fraction (e.g., 10%, or "always sample errors/slow requests"). It's configured in the OTel SDK ([07-OpenTelemetry.md](07-OpenTelemetry.md)):

```csharp
// e.g., sample 10% of traces (head sampling), or use tail sampling to keep all errors/slow traces
tracing.SetSampler(new TraceIdRatioBasedSampler(0.1));
```

Sampling controls trace cost while keeping enough data to diagnose problems. **Tail-based sampling** (decide after the trace completes — keep all errors and slow traces, sample the rest) is ideal but needs a collector; head sampling (decide upfront) is simpler. Tune sampling to your volume/budget.

---

## Common gotchas

### Not propagating context across non-HTTP boundaries

HTTP propagates `traceparent` automatically; **messaging/queues don't** — you must propagate the context in message headers, or the trace breaks at the queue ([Ch07](../07-Messaging/README.md)).

### Creating `ActivitySource` per operation

Wasteful and fragments tracing. Create it **once** (static).

### Not marking span status on failure

A failed span without `SetStatus(Error)` doesn't stand out in trace views. Set status on exceptions so failures are visible.

### Forgetting `activity?` (null when uncollected)

`StartActivity` can return null (no listener); always null-check (`activity?.SetTag`). This is also the feature that makes instrumentation free when off.

### No sampling at high volume

Recording every trace at scale is expensive. Sample (keeping errors/slow traces) to control cost.

### Putting span-level detail in metrics

High-cardinality ids belong in span **tags** (and logs), not metric dimensions ([05-Metrics.md](05-Metrics.md)). Use spans for per-request detail.

---

## Summary

- A **trace** = one request's path across services as a tree of **spans** (`Activity` from `ActivitySource` — the OTel tracing API); it shows *where time went* and *where it failed*.
- Spans **nest automatically** (via `Activity.Current`/`AsyncLocal`) into a trace tree with a shared **TraceId**; **W3C Trace Context** (`traceparent`) **propagates** the trace across services — **automatic over HTTP** (HttpClient/ASP.NET Core), **manual over messaging** (message headers).
- Spans carry **tags** (can be high-cardinality — ids belong here, unlike metrics), **events**, and **status** (mark failures); logs carry the **TraceId/SpanId**, enabling **trace ↔ log correlation**.
- Instrumentation is **near-free when uncollected** (`StartActivity` → null) — instrument liberally; create sources **once**; **sample** at high volume (keep errors/slow traces) to control cost.

→ Next: [07-OpenTelemetry.md](07-OpenTelemetry.md)
