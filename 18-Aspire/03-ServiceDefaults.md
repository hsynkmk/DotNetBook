# ServiceDefaults

## Cross-cutting concerns, by convention

Every service in a distributed app needs the same plumbing: **OpenTelemetry** (traces/metrics/logs), **health checks**, **HTTP resilience**, and **service discovery**. Wiring these per service, consistently, is tedious and error-prone — one service forgets tracing, another misconfigures health checks, and your observability has holes. Aspire's answer is the **ServiceDefaults** project: a shared library with a single extension method (`AddServiceDefaults()`) that wires all of it by convention. Every service calls it once in `Program.cs`, and they're all instrumented and configured identically.

```csharp
// In each service's Program.cs:
var builder = WebApplication.CreateBuilder(args);
builder.AddServiceDefaults();   // OpenTelemetry + health checks + resilience + service discovery

var app = builder.Build();
app.MapDefaultEndpoints();      // /health and /alive endpoints
app.Run();
```

---

## What `AddServiceDefaults()` wires

The generated `ServiceDefaults` project (added by the Aspire template) contains an `Extensions.cs` you own and can customize. By default `AddServiceDefaults()` sets up four things:

1. **OpenTelemetry** ([Ch12](../12-Observability/README.md)) — traces, metrics, and logs with the standard ASP.NET Core/HttpClient/runtime instrumentation, exporting via OTLP to the dashboard (and to your APM in production).
2. **Health checks** ([Ch04 §16](../04-AspNetCore/16-HealthChecks.md)) — a default liveness check, exposed via `MapDefaultEndpoints()` as `/health` (readiness) and `/alive` (liveness).
3. **HTTP resilience** ([Ch11](../11-Resilience/README.md)) — the standard resilience handler on `HttpClient`s (retry, circuit breaker, timeouts) by default.
4. **Service discovery** ([04-ServiceDiscovery.md](04-ServiceDiscovery.md)) — the service-discovery client so `HttpClient`s can resolve `http://servicename` to real endpoints.

One call gives every service production-grade observability, resilience, and discovery — the conventions that would otherwise be copy-pasted (and drift) across services.

---

## It's *your* project — customize it

Unlike a black-box NuGet package, ServiceDefaults is **scaffolded into your solution as source** — you own `Extensions.cs` and edit it to fit your conventions:

```csharp
public static TBuilder AddServiceDefaults<TBuilder>(this TBuilder builder)
    where TBuilder : IHostApplicationBuilder {
    builder.ConfigureOpenTelemetry();
    builder.AddDefaultHealthChecks();

    builder.Services.AddServiceDiscovery();
    builder.Services.ConfigureHttpClientDefaults(http => {
        http.AddStandardResilienceHandler();        // resilience on all HttpClients
        http.AddServiceDiscovery();                 // discovery on all HttpClients
    });
    return builder;
}
```

You can add company-wide defaults here — extra telemetry attributes, a custom health check, organization resilience policies, additional exporters — so *every* service in the solution inherits them by calling `AddServiceDefaults()`. It's the one place to enforce cross-cutting standards.

---

## `ConfigureOpenTelemetry` — the observability core

The bulk of ServiceDefaults is OpenTelemetry setup ([Ch12 §07](../12-Observability/07-OpenTelemetry.md)): it registers tracing (ASP.NET Core incoming requests, outgoing `HttpClient` calls, and library instrumentation like EF Core), metrics (runtime, ASP.NET Core, HttpClient meters), and logging, all exporting via **OTLP**. Aspire sets the OTLP endpoint to the **dashboard** locally (so traces/metrics show up live — [06-Dashboard.md](06-Dashboard.md)), and to your real backend in production. This is why Aspire apps have **distributed tracing across services for free** — every service is instrumented the same way and exports to the same place, so a request flowing API → worker → database shows as one correlated trace.

---

## `MapDefaultEndpoints` — health endpoints

`app.MapDefaultEndpoints()` maps the health-check endpoints:

- **`/health`** — *readiness*: are all checks (including dependencies) healthy? Used to decide whether to route traffic.
- **`/alive`** — *liveness*: is the process up? Used to decide whether to restart.

This distinction matches the liveness-vs-readiness model ([Ch04 §16](../04-AspNetCore/16-HealthChecks.md)) that orchestrators (Kubernetes, Container Apps) consume. By default these are mapped only outside Development for security; customize as needed.

---

## How AppHost and ServiceDefaults fit together

Two complementary pieces ([01-WhatIsAspire.md](01-WhatIsAspire.md)):

- The **AppHost** ([02-AppHost.md](02-AppHost.md)) composes the graph and injects connection info — *outside* the services.
- **ServiceDefaults** configures each service's *internals* (telemetry, health, resilience, discovery) — *inside* each service.

Together: the AppHost wires services to resources and to each other; ServiceDefaults ensures each service is observable, resilient, and discovery-aware. Neither replaces the other.

---

## Common gotchas

### Forgetting to call `AddServiceDefaults()`

A service that doesn't call it misses telemetry, health, resilience, and discovery — a blind spot in your distributed app. Every service should call it (and `MapDefaultEndpoints()`).

### Expecting it to be a sealed library

ServiceDefaults is **source in your solution** — it's meant to be edited. Customize `Extensions.cs` for org-wide conventions instead of fighting defaults.

### Health endpoints exposed in production unintentionally

By default detailed health endpoints are mapped carefully (often Dev-only or secured). Don't expose internal health detail publicly — review what `MapDefaultEndpoints` exposes for your environment.

### Duplicating telemetry setup

Adding your own OpenTelemetry config on top of ServiceDefaults can double-register instrumentation. Configure telemetry *in* ServiceDefaults (the single place) rather than scattering it.

### Assuming resilience/discovery without it

Service discovery (`http://servicename`) and HTTP resilience come from ServiceDefaults' `ConfigureHttpClientDefaults`. Without it, named-endpoint resolution and the standard resilience handler aren't applied to your clients.

---

## Summary

- **ServiceDefaults** is a shared project (scaffolded as **source you own**) whose `AddServiceDefaults()` wires four cross-cutting concerns into every service **by convention**: **OpenTelemetry** (traces/metrics/logs), **health checks**, **HTTP resilience** (standard handler), and **service discovery**.
- Call **`builder.AddServiceDefaults()`** in each service's `Program.cs` and **`app.MapDefaultEndpoints()`** to expose **`/health`** (readiness) and **`/alive`** (liveness) — matching the orchestrator liveness/readiness model.
- Its OpenTelemetry setup exports via **OTLP** to the **dashboard** locally (and your APM in prod), giving **distributed tracing across services for free** ([Ch12](../12-Observability/README.md)).
- It's **editable source** — add org-wide defaults (extra telemetry, custom checks, resilience policies) in one place; it complements the **AppHost** (which wires the graph *outside* services) by configuring each service's *internals*.

→ Next: [04-ServiceDiscovery.md](04-ServiceDiscovery.md)
