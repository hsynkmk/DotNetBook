# Timeout

## Don't wait forever

A timeout bounds how long an operation may take before it's abandoned. It's the most fundamental resilience strategy because **unbounded waits are how cascading failures start** ([01-WhyResilience.md](01-WhyResilience.md)): a call to a hung dependency that never returns ties up a thread/connection forever, and enough of those exhaust the caller's resources. Every outbound call should have a timeout.

```csharp
.AddTimeout(TimeSpan.FromSeconds(10))   // abandon the operation after 10s → TimeoutRejectedException
```

---

## Per-attempt vs total timeout

In a resilience pipeline with retries, there are two distinct timeouts (both matter):

```csharp
.AddTimeout(TimeSpan.FromSeconds(30))    // TOTAL: the whole operation, INCLUDING all retries + backoff
.AddRetry(new RetryStrategyOptions { MaxRetryAttempts = 3 })
.AddTimeout(TimeSpan.FromSeconds(5))     // PER-ATTEMPT: each individual try
```

- **Per-attempt timeout** (inner) — bounds a single try, so one hung request doesn't block forever before a retry.
- **Total timeout** (outer) — bounds the entire operation including all retries and their backoff delays, so retry + backoff doesn't blow past the caller's deadline.

Both are needed: per-attempt prevents one slow call from stalling; total prevents the retry chain from taking minutes. Size the total to accommodate the retry policy (else retries get cut off) or accept fewer retries.

---

## Cooperative vs forceful cancellation

The deep issue: how is a timed-out operation actually **stopped**? .NET cancellation is **cooperative** — a `CancellationToken` *signals* cancellation, but the operation must **observe and honor** it to actually stop ([CSharpBook Ch08 §05]):

```csharp
// ✓ — the operation honors the token, so a timeout actually cancels the work
async Task<Data> FetchAsync(CancellationToken ct) {
    var response = await http.GetAsync(url, ct);   // forwards ct → the HTTP call cancels on timeout
    return await response.Content.ReadFromJsonAsync<Data>(ct);
}

// ✗ — the operation ignores the token, so the timeout signal does nothing; the work keeps running
async Task<Data> BadFetchAsync(CancellationToken ct) {
    return await SomeBlockingCallThatIgnoresToken();   // timeout fires, but this keeps going
}
```

Polly's timeout strategy cancels the token at the deadline; if your operation **forwards and honors** the token (passes it to `HttpClient`, EF Core, etc.), the work genuinely stops. If the operation **ignores** the token (blocking I/O, non-cancellable calls), the timeout strategy reports a timeout to the caller but the underlying work **keeps running** in the background (a "fire-and-forget leak"). So: **propagate the `CancellationToken` everywhere** so timeouts are effective, not just cosmetic.

---

## CancellationToken propagation (the linchpin)

A timeout is only as good as the token propagation behind it. The token must flow through the entire call chain to the actual I/O:

```csharp
// The token flows: pipeline timeout → ExecuteAsync → your method → HttpClient/EF/etc.
await pipeline.ExecuteAsync(async ct => {     // Polly provides a token that cancels at the timeout
    var data = await repo.GetAsync(id, ct);    // forward ct
    return await transform.RunAsync(data, ct); // forward ct again
}, callerToken);                               // linked with the caller's token too
```

`HttpClient`, EF Core, and most BCL async APIs accept and honor a `CancellationToken` — forward it. Combine the timeout's token with the caller's token (Polly does this) so a client disconnect *or* a timeout cancels the work. Code that drops the token (calls async methods without passing it) breaks timeout/cancellation throughout — a common, subtle bug.

---

## Timeouts at every layer

Timeouts should exist at multiple levels, each bounding its scope:

| Layer | Timeout |
|---|---|
| HTTP request | per-request via the resilience pipeline / `CancellationToken` (+ linked CTS) — [Ch09 §01](../09-NetworkingAndHttp/01-HttpClient.md) |
| Resilience pipeline | per-attempt + total ([above]) |
| Database query | `CommandTimeout` / pass `CancellationToken` ([Ch05](../05-EFCore/README.md)) |
| Whole request (server) | request timeout middleware / cancellation on disconnect |
| Background work | bound each unit; honor the stopping token ([Ch08](../08-BackgroundProcessing/README.md)) |

The principle: **bound everything**. An unbounded wait anywhere in the chain is a latent cascade trigger. Set sensible deadlines at each layer and propagate cancellation through them.

---

## Common gotchas

### No timeout → cascading failure

An unbounded call to a hung dependency ties up a thread/connection indefinitely; enough of these exhaust the pool and cascade. Always set a timeout on outbound calls.

### Operation ignores the CancellationToken

If the work doesn't observe the token, the timeout reports a failure but the work **keeps running** (background leak). Propagate and honor the token through to the actual I/O.

### Dropping the token mid-chain

Calling an async method without forwarding the token breaks cancellation/timeout downstream. Thread the token through every async call.

### Total timeout too short for the retry policy

If the total timeout < (attempts × (per-attempt timeout + backoff)), retries get cut off. Size the total to fit the retry policy, or reduce retries.

### Only a per-attempt timeout (or only a total)

Per-attempt alone lets the retry chain run long; total alone lets one hung attempt stall until the whole budget elapses. Use both.

### Relying on `HttpClient.Timeout` for granularity

It's whole-request and per-client. Use the resilience pipeline / per-request `CancellationToken` for granular, composable timeouts ([Ch09 §01](../09-NetworkingAndHttp/01-HttpClient.md)).

---

## Summary

- A **timeout** bounds operation duration; **unbounded waits cause cascading failure** (a hung dependency ties up the caller's threads/connections), so **every outbound call needs a timeout**.
- Use both a **per-attempt** timeout (bounds one try) and a **total** timeout (bounds the whole operation including retries + backoff); size the total to fit the retry policy.
- .NET cancellation is **cooperative** — a timeout cancels a `CancellationToken`, but the operation must **observe and honor** it to actually stop. **Propagate the token** through the entire chain to the real I/O (`HttpClient`/EF/etc.), or the timeout is cosmetic and the work leaks.
- **Bound everything** at every layer (HTTP, DB, request, background), and forward cancellation through them — an unbounded wait anywhere is a latent cascade trigger.

→ Next: [07-Hedging.md](07-Hedging.md)
