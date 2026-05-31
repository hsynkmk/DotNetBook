# Chapter 11 — Resilience — Coding Problems

Build resilience pipelines (retry, circuit breaker, timeout), wire them to HttpClient, and apply application-level patterns.

---

## Problem 1: A retry pipeline with backoff + jitter

Build a Polly pipeline that retries transient failures with exponential backoff and jitter.

<details><summary>Solution</summary>

```csharp
var pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    .AddRetry(new RetryStrategyOptions<HttpResponseMessage> {
        MaxRetryAttempts = 3,
        BackoffType = DelayBackoffType.Exponential,    // 1s, 2s, 4s
        UseJitter = true,                               // randomize → no retry storm
        ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
            .Handle<HttpRequestException>()
            .Handle<TimeoutRejectedException>()
            .HandleResult(r => (int)r.StatusCode >= 500),   // 5xx transient; NOT 4xx
        OnRetry = args => { logger.LogWarning("Retry {N}", args.AttemptNumber); return default; }
    })
    .Build();

var response = await pipeline.ExecuteAsync(async ct => await http.GetAsync(url, ct), cancellationToken);
```

Exponential backoff + jitter; retries only transient failures (network/timeout/5xx, not 4xx); logs each retry. ([04-Retry.md](04-Retry.md), [02-Polly.md](02-Polly.md).)

</details>

---

## Problem 2: Add a circuit breaker

Compose a breaker so persistent failures stop the retries.

<details><summary>Solution</summary>

```csharp
var pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    .AddRetry(new RetryStrategyOptions<HttpResponseMessage> { MaxRetryAttempts = 3, UseJitter = true })
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage> {
        FailureRatio = 0.5,                          // trip if >50% fail...
        MinimumThroughput = 10,                      // ...over at least 10 calls
        SamplingDuration = TimeSpan.FromSeconds(30),
        BreakDuration = TimeSpan.FromSeconds(15),    // open for 15s, then test
        OnOpened = args => { logger.LogError("Circuit OPENED"); return default; },
        OnClosed = args => { logger.LogInformation("Circuit closed (recovered)"); return default; }
    })
    .Build();   // build ONCE — the breaker state is shared across calls
```

Retry handles blips; the breaker trips on sustained failure (fail fast, let the service recover). Build once so the breaker state persists. Log open/close — a critical signal. ([05-CircuitBreaker.md](05-CircuitBreaker.md).)

</details>

---

## Problem 3: Add per-attempt and total timeouts

Bound each attempt and the whole operation.

<details><summary>Solution</summary>

```csharp
var pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    .AddTimeout(TimeSpan.FromSeconds(30))            // OUTER: total budget (incl. all retries + backoff)
    .AddRetry(new RetryStrategyOptions<HttpResponseMessage> { MaxRetryAttempts = 3, UseJitter = true })
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage> { FailureRatio = 0.5 })
    .AddTimeout(TimeSpan.FromSeconds(5))             // INNER: per-attempt
    .Build();

// The operation MUST honor the token for the timeout to actually cancel the work:
await pipeline.ExecuteAsync(async ct => await http.GetAsync(url, ct), cancellationToken);  // forwards ct
```

Standard order: total timeout → retry → breaker → per-attempt timeout → call. The operation forwards the token so timeouts genuinely cancel the HTTP call (cooperative cancellation). ([06-Timeout.md](06-Timeout.md).)

</details>

---

## Problem 4: Wire resilience to HttpClient (the easy way)

Instead of hand-building, use the standard resilience handler.

<details><summary>Solution</summary>

```csharp
builder.Services.AddHttpClient<ApiClient>(c => c.BaseAddress = new Uri("https://api.example/"))
    .AddStandardResilienceHandler(o => {
        o.Retry.MaxRetryAttempts = 3;
        o.CircuitBreaker.FailureRatio = 0.5;
        o.AttemptTimeout.Timeout = TimeSpan.FromSeconds(5);
        o.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(30);
    });
```

One line gives the full pipeline (total timeout → retry → breaker → per-attempt timeout) pre-tuned for HTTP, with built-in telemetry — prefer this over hand-building for `HttpClient`. ([03-ResiliencePipeline.md](03-ResiliencePipeline.md), [Ch09 §04](../09-NetworkingAndHttp/04-Resilience.md).)

</details>

---

## Problem 5: Named pipeline in DI for a non-HTTP operation

Apply resilience to a database call via a registered named pipeline.

<details><summary>Solution</summary>

```csharp
builder.Services.AddResiliencePipeline("db", b => b
    .AddRetry(new RetryStrategyOptions {
        MaxRetryAttempts = 3, BackoffType = DelayBackoffType.Exponential, UseJitter = true,
        ShouldHandle = new PredicateBuilder().Handle<DbException>(ex => ex.IsTransient)  // transient DB faults only
    })
    .AddTimeout(TimeSpan.FromSeconds(10)));

public class ReportService(ResiliencePipelineProvider<string> pipelines, AppDbContext db) {
    public Task<Report> GetAsync(int id, CancellationToken ct) =>
        pipelines.GetPipeline("db").ExecuteAsync(t => BuildReportAsync(id, t), ct);
}
```

Polly isn't just for HTTP — apply a named, DI-registered pipeline to DB calls (retry only *transient* DB faults). Resolve from the provider (built once, shared state). ([02-Polly.md](02-Polly.md), [03-ResiliencePipeline.md](03-ResiliencePipeline.md).)

</details>

---

## Problem 6: Fallback / graceful degradation

Serve cached data when a non-critical dependency's circuit is open.

<details><summary>Solution</summary>

```csharp
public async Task<IReadOnlyList<Product>> GetRecommendationsAsync(int userId, CancellationToken ct) {
    try {
        return await _pipeline.ExecuteAsync(t => _recoService.GetAsync(userId, t), ct);
    }
    catch (BrokenCircuitException) {              // circuit open — service is down
        logger.LogWarning("Recommendations unavailable; serving cached/empty");
        return await _cache.GetAsync($"reco:{userId}") ?? [];   // degrade: cached or empty
    }
    catch (TimeoutRejectedException) {
        return [];                                 // degrade rather than fail the whole page
    }
}
```

Recommendations are non-critical — degrade (cached/empty) when the dependency is down rather than failing the page. A payment, by contrast, would fail clearly. Decide per dependency. ([05-CircuitBreaker.md](05-CircuitBreaker.md), [08-Patterns.md](08-Patterns.md).)

</details>

---

## Problem 7: Make a retried operation idempotent

Retrying a charge can double-charge. Make it safe with an idempotency key.

<details><summary>Solution</summary>

```csharp
public async Task<PaymentResult> ChargeAsync(string idempotencyKey, decimal amount, CancellationToken ct) {
    // Dedup: if we've already processed this key, return the prior result (don't charge again)
    if (await _store.TryGetResultAsync(idempotencyKey, out var existing)) return existing;

    var result = await _pipeline.ExecuteAsync(async t =>
        await _gateway.ChargeAsync(amount, idempotencyKey, t), ct);   // gateway ALSO dedupes on the key

    await _store.SaveResultAsync(idempotencyKey, result);
    return result;
}
```

The idempotency key makes the charge safe to retry: a retried request (after a lost response) is deduped — by your store and by the gateway — so the customer is charged once. This is what makes retrying a non-idempotent operation safe. ([04-Retry.md](04-Retry.md), [08-Patterns.md](08-Patterns.md), [Ch07 §07](../07-Messaging/07-Patterns.md).)

</details>

---

## Problem 8: Hedging for tail latency (idempotent reads)

Cut p99 latency on a read by hedging.

<details><summary>Solution</summary>

```csharp
var pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    .AddHedging(new HedgingStrategyOptions<HttpResponseMessage> {
        MaxHedgedAttempts = 2,                              // up to 2 extra parallel attempts
        Delay = TimeSpan.FromMilliseconds(200),             // fire a hedge only if no response in 200ms (above median)
        ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
            .HandleResult(r => (int)r.StatusCode >= 500)
    })
    .Build();

// Only for IDEMPOTENT reads — and only with backend capacity to spare
var response = await pipeline.ExecuteAsync(async ct => await http.GetAsync("/products", ct), cancellationToken);
```

Hedging fires a backup request if the first is slow (past 200ms — above median, so only genuinely slow requests hedge), using whichever responds first. Safe only because it's an idempotent GET; the cost is extra load. ([07-Hedging.md](07-Hedging.md).)

</details>

---

## Problem 9: Spot the resilience mistakes

Find the problems.

```csharp
// (1) new pipeline per call
var pipeline = new ResiliencePipelineBuilder().AddCircuitBreaker(new()).Build();
await pipeline.ExecuteAsync(...);

// (2) retry a POST with no idempotency
.AddRetry(new RetryStrategyOptions { MaxRetryAttempts = 5 })  // on a payment POST

// (3) no timeout
await httpClient.GetAsync(url);   // could hang forever

// (4) retry everything
ShouldHandle = _ => true   // retries 4xx too
```

<details><summary>Solution</summary>

1. **New pipeline per call** — the circuit breaker's state is shared across calls; rebuilding per call resets it so it **never trips**. Build once (DI-registered) and reuse.
2. **Retrying a non-idempotent POST** (payment) — a retry after a lost response **double-charges**. Use an idempotency key (Problem 7) or don't retry.
3. **No timeout** — an unbounded call to a hung dependency ties up a thread forever → cascading failure. Add per-attempt + total timeouts.
4. **Retrying everything** (`ShouldHandle = _ => true`) — retries permanent 4xx errors (pointless) and worsens load. Retry only transient failures (network/timeout/5xx/429), not 4xx.

Fixed: build the pipeline once, idempotency-key the charge, add timeouts, and scope `ShouldHandle` to transient failures. ([02](02-Polly.md)/[04](04-Retry.md)/[05](05-CircuitBreaker.md)/[06](06-Timeout.md).)

</details>

---

## Problem 10: A complete resilient client (everything together)

Compose retry + breaker + timeouts + fallback for a critical dependency call.

<details><summary>Solution</summary>

```csharp
// Pipeline (built once via DI)
builder.Services.AddResiliencePipeline<string, Product?>("product-svc", b => b
    .AddTimeout(TimeSpan.FromSeconds(20))                                  // total
    .AddRetry(new RetryStrategyOptions<Product?> {
        MaxRetryAttempts = 3, BackoffType = DelayBackoffType.Exponential, UseJitter = true })
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions<Product?> { FailureRatio = 0.5, MinimumThroughput = 10 })
    .AddTimeout(TimeSpan.FromSeconds(5)));                                  // per-attempt

public class ProductService(ResiliencePipelineProvider<string> pipelines, IMemoryCache cache, IProductApi api) {
    public async Task<Product?> GetAsync(int id, CancellationToken ct) {
        var pipeline = pipelines.GetPipeline<Product?>("product-svc");
        try {
            var product = await pipeline.ExecuteAsync(t => api.GetAsync(id, t), ct);
            if (product is not null) cache.Set($"product:{id}", product, TimeSpan.FromMinutes(10));  // refresh cache
            return product;
        }
        catch (Exception ex) when (ex is BrokenCircuitException or TimeoutRejectedException) {
            logger.LogWarning(ex, "Product service unavailable; serving cached");
            return cache.Get<Product>($"product:{id}");   // FALLBACK: stale cache (or null) — degrade
        }
    }
}
```

Layered: total + per-attempt timeouts, retry with backoff/jitter, circuit breaker (built once, shared state), and a **fallback** to stale cache when the circuit is open/times out — graceful degradation. The full call-level resilience stack. ([02–06], [08-Patterns.md](08-Patterns.md).)

</details>

---

You can now build layered resilience: retry (backoff + jitter, transient-only), circuit breakers (fail fast, recover), per-attempt + total timeouts (cooperative cancellation), hedging (idempotent reads), fallbacks (graceful degradation), and the application-level idempotency that makes it all correct — wired via DI-registered pipelines or the HTTP standard handler.

→ Back to [Chapter 11 README](README.md) · Next chapter: [Chapter 12 — Observability](../12-Observability/README.md)
