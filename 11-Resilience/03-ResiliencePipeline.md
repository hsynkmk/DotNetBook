# Microsoft.Extensions.Resilience

## The DI-integrated resilience stack (.NET 8+)

`Microsoft.Extensions.Resilience` is Microsoft's resilience layer built **on Polly v8**, integrated with dependency injection, configuration, and telemetry. It provides the **named pipeline** registration ([02-Polly.md](02-Polly.md)) and the **standard resilience handler** for HTTP ([Ch09 §04](../09-NetworkingAndHttp/04-Resilience.md)). Think of it as "Polly, wired into the .NET host" — the recommended way to apply resilience in modern apps.

```csharp
// Register a named pipeline (DI-integrated, configurable, telemetry-enabled)
builder.Services.AddResiliencePipeline("orders", (builder, context) => {
    builder
        .AddRetry(new RetryStrategyOptions { MaxRetryAttempts = 3, UseJitter = true })
        .AddCircuitBreaker(new CircuitBreakerStrategyOptions { FailureRatio = 0.5 })
        .AddTimeout(TimeSpan.FromSeconds(10));
});

// Resolve and use it
public class OrderService(ResiliencePipelineProvider<string> provider) {
    public Task<Order> GetAsync(int id, CancellationToken ct) =>
        provider.GetPipeline("orders").ExecuteAsync(t => FetchAsync(id, t), ct);
}
```

---

## What it adds over raw Polly

| | Raw Polly | Microsoft.Extensions.Resilience |
|---|---|---|
| Pipeline registration | manual instance | **DI-registered, named** (`AddResiliencePipeline`) |
| Configuration | code | bind from `IConfiguration` / options |
| Telemetry | callbacks | **built-in** metrics + traces (OpenTelemetry-ready) |
| HTTP integration | manual handler | **`AddStandardResilienceHandler`** |
| Strategy reuse | manual | named, shared (with shared circuit state) |

It's the same Polly strategies, but managed by the host: pipelines are named/reusable, configurable via options, and emit telemetry automatically ([Ch12](../12-Observability/README.md)). For a modern .NET app, use this rather than constructing Polly pipelines by hand — you get DI, config, and observability for free.

---

## The standard resilience handler (HTTP)

For `HttpClient`, the package's headline feature is the pre-configured handler ([Ch09 §04](../09-NetworkingAndHttp/04-Resilience.md)):

```csharp
builder.Services.AddHttpClient<ApiClient>()
    .AddStandardResilienceHandler();   // total timeout → retry → circuit breaker → per-attempt timeout
```

This is a Microsoft.Extensions.Resilience pipeline tuned for HTTP with sensible defaults, applied as a `DelegatingHandler`. It's the recommended one-liner for outbound HTTP resilience. Customize via options if needed.

---

## Configuration-bound resilience

Pipeline settings can come from configuration, so you tune resilience without recompiling:

```json
// appsettings.json
{ "Resilience": { "Retry": { "MaxRetryAttempts": 5 }, "CircuitBreaker": { "FailureRatio": 0.3 } } }
```

```csharp
builder.Services.AddResiliencePipeline("orders", (builder, context) => {
    var options = context.GetOptions<MyResilienceOptions>("Resilience");   // bound from config
    builder.AddRetry(new RetryStrategyOptions { MaxRetryAttempts = options.Retry.MaxRetryAttempts });
});
```

Binding resilience parameters to configuration ([Ch03 §07](../03-HostingAndDI/07-Configuration.md)) lets you adjust retry counts, timeouts, and breaker thresholds per environment (more aggressive in dev, conservative in prod) without code changes — and validate them at startup ([Ch03 §10](../03-HostingAndDI/10-Validation.md)).

---

## Telemetry integration

The package emits resilience **metrics and traces** automatically (via the `Meter`/`ActivitySource` primitives — [Ch02 §08](../02-BCL/08-Diagnostics.md)): retry counts, circuit-breaker state changes, execution durations, timeouts. With OpenTelemetry configured ([Ch12](../12-Observability/README.md)), these flow to your dashboards — so you can see retry spikes (a struggling dependency) and circuit-breaker trips (an outage) without manual instrumentation. This built-in observability is a major reason to use the DI-integrated package over hand-rolled Polly.

---

## Common gotchas

### Hand-building Polly when the package fits

For HTTP, use `AddStandardResilienceHandler`; for named reusable pipelines, use `AddResiliencePipeline`. Hand-constructing pipelines loses DI, config-binding, and telemetry.

### Per-call pipeline construction

Resolve named pipelines from `ResiliencePipelineProvider` (built once, shared); building per call wastes work and resets circuit-breaker state (which must persist across calls — [05-CircuitBreaker.md](05-CircuitBreaker.md)).

### Ignoring the emitted telemetry

The package emits resilience metrics/traces for free — wire up OpenTelemetry to actually see them ([Ch12](../12-Observability/README.md)), or you're blind to retry storms / open circuits.

### Not validating config-bound options

Resilience parameters from config (retry counts, timeouts) should be validated at startup ([Ch03 §10](../03-HostingAndDI/10-Validation.md)) so a bad value fails fast, not at runtime.

---

## Summary

- **`Microsoft.Extensions.Resilience`** is Polly v8 integrated with the .NET host — **DI-registered named pipelines** (`AddResiliencePipeline` + `ResiliencePipelineProvider`), **config-bound** settings, and **built-in telemetry** — the recommended way to apply resilience.
- It powers the HTTP **`AddStandardResilienceHandler`** (the one-liner for outbound HTTP resilience — [Ch09 §04](../09-NetworkingAndHttp/04-Resilience.md)).
- **Bind resilience parameters to configuration** (per-environment tuning, startup-validated) and rely on its **automatic metrics/traces** (retry counts, circuit state) flowing to OpenTelemetry ([Ch12](../12-Observability/README.md)).
- Build/register pipelines **once** (shared circuit state); prefer this package over hand-rolled Polly for DI/config/observability.

→ Next: [04-Retry.md](04-Retry.md)
