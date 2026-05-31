# Retry

## Re-attempting transient failures

Retry is the most basic resilience strategy: when an operation fails **transiently** (a network blip, a transient DB deadlock, a momentary 503), try again — the second attempt often succeeds. Done well, retry turns intermittent failures invisible; done badly, it amplifies outages and corrupts data. The art is in **what** you retry, **how** you back off, and **whether the operation is safe to repeat**.

```csharp
.AddRetry(new RetryStrategyOptions {
    MaxRetryAttempts = 3,
    BackoffType = DelayBackoffType.Exponential,   // 1s, 2s, 4s...
    UseJitter = true,                              // randomize delays (anti-thundering-herd)
    ShouldHandle = new PredicateBuilder().Handle<HttpRequestException>().Handle<TimeoutRejectedException>()
})
```

---

## Backoff strategies

Retrying immediately and repeatedly hammers a struggling service. **Backoff** spaces out retries:

| Strategy | Delays | Use |
|---|---|---|
| **Constant** | 1s, 1s, 1s | rarely — doesn't relieve a struggling service |
| **Linear** | 1s, 2s, 3s | mild backoff |
| **Exponential** | 1s, 2s, 4s, 8s | **the default** — quickly backs off, giving the service room |

**Exponential backoff** is the standard: each retry waits exponentially longer, so a transient blip recovers fast (first retry is quick) while a struggling service gets increasing breathing room. Cap the maximum delay so you don't wait minutes.

---

## Jitter — preventing retry storms

The critical addition: **jitter** (randomized delays). Without it, many clients that failed at the same moment retry in **lockstep** — synchronized waves of retries that hammer the recovering service exactly when it's most fragile (a "thundering herd" / "retry storm"):

```
Without jitter: 1000 clients fail at T → all retry at T+1s → all retry at T+2s → synchronized spikes crush the service
With jitter:    each client's delay is randomized (e.g., 0.5–1.5s) → retries spread out → smooth load
```

`UseJitter = true` randomizes each delay around the backoff value, spreading retries over time. **Always use jitter** with backoff in any system with many clients — it's the difference between retries helping and retries causing a self-inflicted DDoS.

---

## The idempotency problem (the dangerous one)

The biggest retry hazard: **retrying a non-idempotent operation can cause duplicates.** If the first attempt actually **succeeded** but the response was lost (network dropped the reply, or a timeout fired after the server processed it), the retry executes the operation **again**:

```
Client → POST /charge ($100) → server CHARGES the card → response lost in transit
Client times out → RETRIES → server CHARGES AGAIN → customer charged $200 (!)
```

So:
- **Idempotent operations** (GET, PUT-by-id, "set status to X", `DELETE`) are **safe to retry** — repeating them has the same effect.
- **Non-idempotent operations** (POST that creates, "increment", "charge") are **dangerous to retry** — they can duplicate.

For non-idempotent operations, either **don't retry**, or make them idempotent with an **idempotency key**: the client sends a unique key; the server dedupes retries (records the key, ignores repeats) so a retried charge is processed once ([Ch07 §07](../07-Messaging/07-Patterns.md)). This is the correct way to retry a payment. Retrying without idempotency is a classic production bug — double charges, duplicate orders.

---

## What to retry (predicates)

Retry only **transient** failures — retrying a permanent error wastes time and can mask bugs:

```csharp
ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
    .Handle<HttpRequestException>()                                      // network errors — transient
    .Handle<TimeoutRejectedException>()                                  // timeouts — transient
    .HandleResult(r => r.StatusCode >= HttpStatusCode.InternalServerError) // 5xx — usually transient
    .HandleResult(r => r.StatusCode == HttpStatusCode.TooManyRequests)   // 429 — retry (honor Retry-After!)
    // NOT 4xx (400/401/403/404) — these won't fix themselves; retrying is pointless/harmful
```

- **Retry**: network errors, timeouts, 5xx, 429 (rate-limited — and honor the `Retry-After` header).
- **Don't retry**: 4xx client errors (bad request, unauthorized, not found) — retrying a 400 just fails again; retrying a 401/403 won't grant access.

---

## Retry + circuit breaker

Retry handles **transient** failures but **worsens sustained** ones (it adds load to a service that's actually down). The fix is to combine retry with a **circuit breaker** ([05-CircuitBreaker.md](05-CircuitBreaker.md)): retry a few times for transient blips, but if failures persist (the breaker trips), **stop retrying** and fail fast until the service recovers. This combination — retry for blips, breaker for outages — is the standard pairing (and what the HTTP standard handler does):

```csharp
.AddRetry(...)            // transient blips
.AddCircuitBreaker(...)   // sustained outage → stop hammering
```

---

## Bounding total retry time

Retries + backoff can take a long time (3 retries with exponential backoff might span 7+ seconds). Bound the **total** time with an outer timeout so an operation doesn't hang far longer than the caller expects:

```csharp
.AddTimeout(TimeSpan.FromSeconds(30))   // OUTER: total budget for all attempts ([06-Timeout.md])
.AddRetry(...)
```

Size the total timeout to accommodate the retry policy (or accept fewer retries). An unbounded retry chain can tie up resources well beyond a sane request deadline.

---

## Common gotchas

### Retrying non-idempotent operations

Retrying a POST/charge/increment can **duplicate** it if the first attempt succeeded but the response was lost. Retry only idempotent operations, or use an **idempotency key** the server dedupes on.

### No jitter → retry storm

Synchronized retries (no jitter) hammer a recovering service in waves. Always combine backoff with **jitter**.

### Retrying permanent failures

Retrying 4xx (bad request, not found, unauthorized) is pointless and wastes time. Configure predicates to retry only transient failures (network, timeout, 5xx, 429).

### Retry without a circuit breaker

Retrying a sustained outage piles load on a down service. Pair retry with a circuit breaker.

### Ignoring `Retry-After` on 429

A 429 (rate-limited) often includes `Retry-After`; ignoring it and retrying immediately worsens the throttling. Honor it.

### Unbounded total retry time

Retries + backoff can far exceed the expected deadline. Bound with an outer total timeout.

---

## Summary

- **Retry** re-attempts **transient** failures (network blips, transient deadlocks, 5xx, 429); use **exponential backoff** to give a struggling service room, plus **jitter** to prevent synchronized **retry storms** (essential with many clients).
- The dangerous hazard: **retrying non-idempotent operations duplicates them** (if the first attempt succeeded but the response was lost). Retry only idempotent ops, or use an **idempotency key** (server dedupes) — the correct way to retry a payment.
- Retry only **transient** failures (predicates: network/timeout/5xx/429, **not** 4xx); honor `Retry-After` on 429.
- **Pair retry with a circuit breaker** (retry for blips, breaker for outages) and bound **total retry time** with an outer timeout.

→ Next: [05-CircuitBreaker.md](05-CircuitBreaker.md)
