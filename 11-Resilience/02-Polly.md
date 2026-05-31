# Polly

## The resilience library

Polly is the de-facto .NET resilience library — it provides composable **resilience strategies** (retry, circuit breaker, timeout, fallback, rate limiter, hedging) that you wrap around any operation. Modern Polly (v8+) is integrated into **`Microsoft.Extensions.Resilience`** and is what powers `HttpClient`'s standard resilience handler ([Ch09 §04](../09-NetworkingAndHttp/04-Resilience.md)).

```csharp
// Build a resilience pipeline (Polly v8 API)
var pipeline = new ResiliencePipelineBuilder()
    .AddRetry(new RetryStrategyOptions {
        MaxRetryAttempts = 3, BackoffType = DelayBackoffType.Exponential, UseJitter = true
    })
    .AddTimeout(TimeSpan.FromSeconds(10))
    .Build();

// Execute any operation through it
var result = await pipeline.ExecuteAsync(async ct => await CallServiceAsync(ct), cancellationToken);
```

A pipeline executes your delegate, applying each strategy in order. Polly works on **any** operation (HTTP, DB, message handling, arbitrary async code) — not just HTTP.

---

## The strategies

| Strategy | Handles | Behavior |
|---|---|---|
| **Retry** | transient failures | re-execute on failure (with backoff/jitter — [04](04-Retry.md)) |
| **Circuit breaker** | sustained failures | trip open to fail fast, let the dependency recover ([05](05-CircuitBreaker.md)) |
| **Timeout** | slow operations | cancel after a deadline ([06](06-Timeout.md)) |
| **Fallback** | unrecoverable failures | return a default/degraded result |
| **Rate limiter** | overload | bound concurrency/rate (bulkhead) |
| **Hedging** | tail latency | race additional attempts ([07](07-Hedging.md)) |

Each is a building block; the power is **composing** them into a pipeline (next).

---

## Composing strategies (order matters)

Strategies wrap each other like middleware — the **order you add them defines the nesting**, and that changes behavior significantly:

```csharp
var pipeline = new ResiliencePipelineBuilder()
    .AddTimeout(TimeSpan.FromSeconds(30))      // OUTER: total timeout for the whole operation (incl. retries)
    .AddRetry(new RetryStrategyOptions { MaxRetryAttempts = 3 })
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions { FailureRatio = 0.5 })
    .AddTimeout(TimeSpan.FromSeconds(5))        // INNER: per-attempt timeout
    .Build();
```

This reads outer→inner: a **total timeout** (30s) bounds everything, then **retry**, then the **circuit breaker** (so each retry checks the breaker), then a **per-attempt timeout** (5s) on each individual try. The standard order is: **total timeout → retry → circuit breaker → per-attempt timeout → the call**. Getting order wrong produces subtle bugs — e.g., circuit breaker *outside* retry vs *inside* retry behave very differently (does the breaker count each retry, or each logical operation?). Think about what each layer should "see."

---

## Predicates — what counts as a failure

Strategies act on outcomes you define. By default they handle exceptions, but you configure **which** exceptions/results trigger them — crucial so you retry only *transient* failures, not all errors:

```csharp
.AddRetry(new RetryStrategyOptions<HttpResponseMessage> {
    ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
        .Handle<HttpRequestException>()                              // network errors
        .Handle<TimeoutRejectedException>()                          // timeouts
        .HandleResult(r => r.StatusCode >= HttpStatusCode.InternalServerError),  // 5xx (transient)
        // NOT 4xx — retrying a 400/404 is pointless
    MaxRetryAttempts = 3
})
```

`ShouldHandle` (a predicate over exception **or** result) decides what's a failure. This is how you avoid retrying non-transient errors (a 404 won't fix itself; a validation error shouldn't retry). Configure predicates deliberately per dependency.

---

## Telemetry & callbacks

Polly strategies expose callbacks (and built-in telemetry) so you can observe and react:

```csharp
.AddRetry(new RetryStrategyOptions {
    MaxRetryAttempts = 3,
    OnRetry = args => {
        logger.LogWarning("Retry {Attempt} after {Delay}ms due to {Reason}",
            args.AttemptNumber, args.RetryDelay.TotalMilliseconds, args.Outcome.Exception?.Message);
        return default;
    }
})
```

`OnRetry`/`OnCircuitOpened`/`OnTimeout` etc. let you log, emit metrics, or trigger side effects when resilience events occur — important for observability ([Ch12](../12-Observability/README.md)) (you want to *know* when retries spike or a circuit opens, as it signals a struggling dependency). Polly v8 also emits telemetry that integrates with metrics/tracing automatically.

---

## Polly + DI (named pipelines)

Register pipelines in DI and resolve them by name:

```csharp
builder.Services.AddResiliencePipeline("db-resilience", builder => builder
    .AddRetry(new RetryStrategyOptions { MaxRetryAttempts = 3, BackoffType = DelayBackoffType.Exponential })
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions { FailureRatio = 0.5 }));

public class DataService(ResiliencePipelineProvider<string> pipelines) {
    public async Task<Data> GetAsync(CancellationToken ct) {
        var pipeline = pipelines.GetPipeline("db-resilience");
        return await pipeline.ExecuteAsync(async token => await _db.QueryAsync(token), ct);
    }
}
```

`AddResiliencePipeline` + `ResiliencePipelineProvider` let you define reusable, named pipelines and apply them across your app — the same pattern the HTTP standard resilience handler uses for outbound calls.

---

## Polly for HTTP (recap) and beyond

For `HttpClient`, you usually don't build a pipeline by hand — use **`AddStandardResilienceHandler`** ([Ch09 §04](../09-NetworkingAndHttp/04-Resilience.md)), a pre-tuned Polly pipeline (retry + breaker + timeouts + rate limiter). But Polly applies to **any** operation:

```csharp
// Resilient DB call, message processing, external SDK call, etc. — not just HTTP
await pipeline.ExecuteAsync(async ct => await someExternalSdk.CallAsync(ct), cancellationToken);
```

Use the standard HTTP handler for HTTP; build custom Polly pipelines for non-HTTP operations (databases, message brokers, third-party SDKs) or when you need specific strategy ordering/predicates.

---

## Common gotchas

### Wrong strategy order

Circuit breaker inside vs outside retry, total vs per-attempt timeout placement — order changes behavior. Use the standard order (total timeout → retry → breaker → per-attempt timeout → call) and reason about what each layer sees.

### Retrying non-transient failures

Default predicates may retry things that won't recover (4xx, validation errors). Configure `ShouldHandle` to retry **only transient** failures (network, timeout, 5xx).

### Retry without a circuit breaker

Retrying a sustained outage piles load on a down service. Combine retry with a circuit breaker (and backoff/jitter — [04](04-Retry.md)).

### No telemetry on resilience events

Silent retries/circuit-opens hide a struggling dependency. Wire `OnRetry`/`OnCircuitOpened` to logging/metrics — spiking retries or an open circuit is an alert.

### Hand-rolling instead of using Polly

Custom retry/breaker code mishandles backoff, idempotency, state transitions, and concurrency. Use Polly (or the HTTP standard handler).

### Building a pipeline per call

Constructing a `ResiliencePipeline` per operation wastes work (and circuit-breaker state must persist across calls!). Build/register pipelines once (DI) and reuse — the circuit breaker's state is *shared* across executions, which is the point.

---

## Summary

- **Polly** (v8+, integrated into `Microsoft.Extensions.Resilience`) provides composable **resilience strategies** — retry, circuit breaker, timeout, fallback, rate limiter, hedging — wrapped around **any** operation via a `ResiliencePipeline`.
- **Compose** strategies into a pipeline; **order matters** (standard: total timeout → retry → circuit breaker → per-attempt timeout → call) — order changes behavior subtly.
- Configure **predicates** (`ShouldHandle`) so strategies act only on the right outcomes (retry transient failures/5xx, not 4xx); wire **callbacks/telemetry** (`OnRetry`/`OnCircuitOpened`) for observability.
- Register **named pipelines in DI** (`AddResiliencePipeline` + `ResiliencePipelineProvider`) and reuse them — circuit-breaker state is shared across executions, so build pipelines **once**, not per call.
- For HTTP, prefer the **standard resilience handler** ([Ch09 §04](../09-NetworkingAndHttp/04-Resilience.md)); use custom Polly pipelines for non-HTTP operations or specific needs.

→ Next: [03-ResiliencePipeline.md](03-ResiliencePipeline.md)
