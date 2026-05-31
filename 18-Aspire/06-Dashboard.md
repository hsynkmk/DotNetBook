# The Developer Dashboard

## Observability out of the box

When you run the AppHost, Aspire launches a **developer dashboard** — a web UI that shows your entire running app graph: every resource's state, **live logs**, **distributed traces**, **metrics**, and the actual environment variables/connection strings each service received. This is observability ([Ch12](../12-Observability/README.md)) with zero setup: because ServiceDefaults ([03-ServiceDefaults.md](03-ServiceDefaults.md)) exports OpenTelemetry via OTLP to the dashboard, you get a correlated, real-time view of distributed behavior locally — the thing that's usually hardest to see when developing multi-service apps.

```
dotnet run   (the AppHost)
  → starts your services + containers
  → opens the dashboard at https://localhost:17xxx
       Resources | Console logs | Structured logs | Traces | Metrics
```

---

## Resources view

The landing view lists every resource in the app graph ([02-AppHost.md](02-AppHost.md)) — projects, containers, executables — with:

- **State** (starting / running / unhealthy / exited) and **health** status,
- **Endpoints** (clickable URLs for each service),
- **Source** (image/project) and **start time**,
- **Environment variables** — including the **injected connection strings and discovery endpoints** ([04-ServiceDiscovery.md](04-ServiceDiscovery.md)), so you can *see exactly* what config a service received.

This is invaluable for debugging wiring: if a service can't reach its database, the Resources view shows whether the connection string was injected and whether the dependency is healthy.

---

## Logs — console and structured

Two log views:

- **Console logs** — the raw stdout/stderr of each resource (what you'd see in a terminal), per service, live-streaming.
- **Structured logs** — the **`ILogger`** structured logs ([Ch12 §02](../12-Observability/02-ILogger.md)) from all services in one place, **filterable** by service, level, and attributes, with each log's structured properties expandable.

Crucially, structured logs are **correlated by trace** — a log emitted during a request carries the trace/span id, so you can jump from a log line to the full distributed trace it belongs to. No more SSHing into multiple containers to grep logs — they're aggregated, searchable, and linked.

---

## Distributed traces — the killer feature

The **Traces** view is the dashboard's standout. Because every service is instrumented identically and exports to the dashboard, a single request that flows **API → worker → database → cache** appears as **one correlated trace** — a waterfall of spans across service boundaries:

```
Trace: POST /orders                                    [========================] 240ms
  ├─ orderapi: POST /orders                            [====================]     210ms
  │    ├─ orderapi → ordersdb: INSERT orders           [===]                        18ms
  │    ├─ orderapi → cache: SET order:123              [=]                            4ms
  │    └─ orderapi → broker: publish OrderPlaced       [==]                          12ms
  └─ orderworker: handle OrderPlaced                   [=========]                   95ms
       └─ orderworker → ordersdb: UPDATE inventory     [====]                        40ms
```

You can click any span to see its attributes, duration, and associated logs. This makes **latency and failures across services obvious** — you see *which* span is slow or errored, across the whole call chain, without building a tracing backend. It's the local equivalent of what you'd see in Jaeger/Application Insights in production ([Ch12 §06](../12-Observability/06-Activities.md)).

---

## Metrics

The **Metrics** view shows the OpenTelemetry **meters** ([Ch12 §05](../12-Observability/05-Metrics.md)) each service emits — request rates, durations, HttpClient metrics, runtime metrics (GC, thread pool), and any custom meters — as live charts per service. You can watch request throughput, error rates, and resource usage in real time while exercising the app, spotting regressions or hot paths during development.

---

## Why it matters for development

The dashboard closes the biggest gap in distributed development: **you can normally only see one service at a time.** With it:

- A failing request's **full path** is one trace away — no guessing which service erred.
- **Logs across services** are aggregated and trace-correlated.
- **Wiring problems** (missing connection string, unhealthy dependency) are visible in the Resources view.
- **Performance** is observable live via metrics.

It turns "it works on my machine, somewhere in these five services" into a concrete, navigable picture — and it's the *same telemetry shape* you'll send to your production APM, so what you learn locally transfers.

---

## Standalone and production note

The dashboard can also run **standalone** (a container) to receive OTLP telemetry from any app — not just Aspire-orchestrated ones — useful as a lightweight local telemetry viewer. In **production**, you don't use the dev dashboard as your monitoring tool; you export the same OpenTelemetry to a real backend (Application Insights, Grafana/Tempo/Prometheus, Datadog — [Ch12 §07](../12-Observability/07-OpenTelemetry.md)). The dashboard is the *development* observability surface; production uses durable, scalable APM fed by the identical instrumentation.

---

## Common gotchas

### Treating the dev dashboard as production monitoring

The dashboard is for local development — it's in-memory and not a durable, scalable monitoring system. In production, export OpenTelemetry to a real APM ([Ch12](../12-Observability/README.md)); don't rely on the dev dashboard.

### No traces showing up

If a service doesn't call `AddServiceDefaults()` ([03-ServiceDefaults.md](03-ServiceDefaults.md)), it isn't exporting OTLP to the dashboard — its spans/logs/metrics won't appear. Ensure every service wires the defaults.

### Confusing console vs structured logs

Console logs are raw stdout; structured logs are `ILogger` events with properties and trace correlation. For debugging distributed flows, use **structured logs** (filterable, trace-linked), not just console output.

### Sensitive values visible in the dashboard

The Resources view shows injected env vars including connection strings/secrets — fine locally, but be mindful when screen-sharing, and never expose the dev dashboard publicly.

### Expecting historical data

The dev dashboard shows the current run's telemetry (limited retention); it's not a long-term store. For history/alerting, use a production backend.

---

## Summary

- Running the AppHost launches the **developer dashboard**: a live web UI of the whole app graph — **resource state/health**, **console + structured logs**, **distributed traces**, **metrics**, and the **injected env vars/connection strings** — with **zero setup** (ServiceDefaults exports OTLP to it).
- **Structured logs** are aggregated across services and **trace-correlated**; the **Traces** view shows a single request as **one correlated waterfall across services** (API → worker → DB → cache), making cross-service latency/failures obvious — the standout feature.
- The **Resources** view exposes wiring (connection strings, discovery endpoints, health) for debugging; **Metrics** shows live OpenTelemetry meters per service.
- It's the **development** observability surface (in-memory, current run) using the **same telemetry shape** you export to a real **APM in production** ([Ch12](../12-Observability/README.md)) — not a replacement for production monitoring; every service must call `AddServiceDefaults()` to appear.

→ Next: [07-ConfigAndSecrets.md](07-ConfigAndSecrets.md)
