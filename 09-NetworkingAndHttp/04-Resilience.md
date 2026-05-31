# HTTP Resilience

## Surviving transient failures

Network calls fail transiently — a momentary blip, a brief overload, a slow response, a service restarting. **Resilience** is the set of patterns that handle these gracefully: retry the request, stop hammering a failing service (circuit breaker), bound how long you wait (timeout), and fall back when all else fails. .NET 8+ provides a **standard resilience handler** (built on **Polly**) that adds these to an `HttpClient` in one line.

```csharp
builder.Services.AddHttpClient<ApiClient>(c => c.BaseAddress = new Uri("https://api.example.com"))
    .AddStandardResilienceHandler();   // retry + circuit breaker + timeout + rate limiter, sensible defaults
```

> This is the HTTP-specific application; the broader resilience patterns (and `Microsoft.Extensions.Resilience`/Polly in general) are **[Chapter 11](../11-Resilience/README.md)**. This file focuses on resilient `HttpClient`.

---

## The standard resilience handler (.NET 8+)

`AddStandardResilienceHandler` adds a pre-configured pipeline of resilience strategies with production-sensible defaults — the recommended starting point for outbound HTTP:

```
Outbound request →
  [ Rate limiter ]      — bound concurrent outbound requests
  [ Total timeout ]     — overall deadline for the whole operation (incl. retries)
  [ Retry ]             — retry transient failures with exponential backoff + jitter
  [ Circuit breaker ]   — stop calling a failing endpoint for a while
  [ Attempt timeout ]   — per-try timeout
  → the actual HTTP call
```

It's a `DelegatingHandler` ([03-DelegatingHandlers.md](03-DelegatingHandlers.md)) under the hood, composing Polly strategies. Customize the defaults:

```csharp
.AddStandardResilienceHandler(o => {
    o.Retry.MaxRetryAttempts = 3;
    o.Retry.Delay = TimeSpan.FromMilliseconds(500);
    o.CircuitBreaker.FailureRatio = 0.5;            // open if >50% of calls fail
    o.AttemptTimeout.Timeout = TimeSpan.FromSeconds(5);
    o.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(30);
});
```

---

## Retry — with backoff and jitter

Retrying transient failures is the most basic resilience strategy, but naive retries make things worse:

```csharp
// The standard handler retries with EXPONENTIAL BACKOFF + JITTER by default:
//   attempt 1 fails → wait ~1s → attempt 2 fails → wait ~2s → attempt 3 ...
//   jitter randomizes delays so many clients don't retry in lockstep (avoiding a "retry storm")
```

- **Backoff** — increase the delay between retries (exponential) so you don't hammer a struggling service.
- **Jitter** — randomize delays so many clients retrying simultaneously don't synchronize into a thundering herd.
- **Retry only safe failures** — transient ones (5xx, timeouts, connection errors), not 4xx client errors (retrying a 400 won't help) — the standard handler distinguishes these.

**The idempotency caveat**: retrying a non-idempotent request (e.g., POST that creates a resource) can cause **duplicates** if the first attempt actually succeeded but the response was lost. Only retry idempotent operations safely, or use an **idempotency key** so the server dedupes retries ([Ch07 §07](../07-Messaging/07-Patterns.md)). The standard handler retries by default — be careful with non-idempotent POSTs.

---

## Circuit breaker — stop hammering a failing service

If a downstream service is down, retrying every request just piles on load and makes every caller wait through timeouts. A **circuit breaker** detects a high failure rate and "opens" — short-circuiting subsequent calls (failing fast) for a cooldown period, then "half-opens" to test recovery:

```
Closed   → calls flow normally; track failures
   ↓ (failure ratio exceeds threshold)
Open     → calls FAIL FAST immediately (no network call) for a break duration → protects the failing service + the caller
   ↓ (after the break)
Half-open → allow a trial call; success → Closed, failure → Open again
```

This gives a struggling service room to recover (no flood of retries) and makes the caller fail fast (no waiting through timeouts on every call). It's essential in microservices — without it, one slow dependency can cascade into system-wide latency/failure. The standard handler includes a circuit breaker tuned by failure ratio.

---

## Timeouts — don't wait forever

Two layers of timeout in the standard handler:

```csharp
o.AttemptTimeout.Timeout = TimeSpan.FromSeconds(5);        // each individual try
o.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(30);   // the whole operation, including all retries
```

- **Attempt (per-try) timeout** — bound a single request so a hung connection doesn't block forever.
- **Total (overall) timeout** — bound the entire operation including retries, so retry + backoff doesn't exceed a sane deadline.

Timeouts surface as `TimeoutRejectedException`/`TaskCanceledException`. They prevent slow dependencies from tying up your threads/resources indefinitely. Always bound outbound calls — an unbounded wait is a resource leak under failure.

---

## Fallback — graceful degradation

When retries and the circuit breaker can't get a result, a **fallback** provides a default rather than failing the whole operation:

```csharp
// Fallback (configure via a custom Polly pipeline if the standard handler's defaults don't fit)
// e.g., return cached data, a default value, or a degraded response when the dependency is unavailable
var product = await cache.GetAsync(id) ?? await TryFetchWithResilienceAsync(id) ?? Product.Unavailable;
```

Fallbacks enable graceful degradation — serve stale cache, a default, or a partial response instead of an error when a non-critical dependency is down. (Not every call should fall back — a payment can't "fall back," but a recommendations widget can.) Decide per dependency whether degradation is acceptable.

---

## Custom resilience pipelines

For control beyond the standard handler, compose a Polly `ResiliencePipeline` explicitly:

```csharp
builder.Services.AddResiliencePipeline("custom", builder => {
    builder
        .AddRetry(new RetryStrategyOptions { MaxRetryAttempts = 3, BackoffType = DelayBackoffType.Exponential, UseJitter = true })
        .AddCircuitBreaker(new CircuitBreakerStrategyOptions { FailureRatio = 0.5, BreakDuration = TimeSpan.FromSeconds(30) })
        .AddTimeout(TimeSpan.FromSeconds(10));
});
```

The standard handler covers most needs; use a custom pipeline ([Ch11](../11-Resilience/README.md)) when you need specific strategies/ordering, fallbacks, or hedging (racing duplicate requests). Order matters (e.g., timeout inside retry vs outside).

---

## Common gotchas

### Retrying non-idempotent requests

Retrying a POST that creates a resource can duplicate it if the first attempt succeeded but the response was lost. Retry only idempotent operations, or use an idempotency key the server dedupes on.

### No circuit breaker → cascading failure

Retrying against a down service floods it and makes every caller wait through timeouts, cascading latency system-wide. Use a circuit breaker to fail fast and let the service recover.

### No timeout → resource exhaustion

An unbounded outbound call ties up a thread/connection indefinitely when a dependency hangs. Always set attempt + total timeouts.

### Retry without backoff/jitter

Immediate, synchronized retries (no backoff/jitter) create a retry storm that worsens overload. Use exponential backoff + jitter (the standard handler defaults).

### Hand-rolling resilience as a `DelegatingHandler`

Naive custom retry/breaker handlers mishandle idempotency, response disposal, and timing. Use `AddStandardResilienceHandler` / Polly pipelines.

### Total timeout shorter than retries need

If the total timeout is less than (attempts × (attempt timeout + backoff)), retries get cut off. Size the total timeout to accommodate the retry policy, or accept fewer retries.

---

## Summary

- **HTTP resilience** handles transient failures: **`AddStandardResilienceHandler`** (.NET 8+) adds retry + circuit breaker + timeouts + rate limiting (Polly-based) to an `HttpClient` in one line with sensible defaults.
- **Retry** transient failures with **exponential backoff + jitter** (the handler defaults) — but only **idempotent** operations, or use an idempotency key (retrying non-idempotent POSTs duplicates).
- **Circuit breaker** opens on high failure rate to **fail fast** and let a struggling service recover — essential to prevent cascading failures in microservices.
- **Timeouts** (per-attempt + total) bound waits so slow dependencies don't exhaust resources; **fallbacks** enable graceful degradation where acceptable.
- The standard handler suits most cases; build custom Polly pipelines ([Ch11](../11-Resilience/README.md)) for specific strategies/ordering. Don't hand-roll resilience as a delegating handler.

→ Next: [05-Authentication.md](05-Authentication.md)
