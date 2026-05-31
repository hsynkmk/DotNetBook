# Chapter 11 — Resilience

> Retry, timeout, circuit breaker, fallback, hedging. Surviving the unreliable world: networks fail, services slow down, things break.

**Prerequisites**: Chapter 03 (Hosting & DI), CSharpBook Chapter 08 (Concurrency).

**Time to read**: ~4-6 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-WhyResilience.md](01-WhyResilience.md) | What can fail; the difference between resilience and reliability. |
| [02-Polly.md](02-Polly.md) | Polly's policies: retry, circuit breaker, timeout, fallback, bulkhead. |
| [03-ResiliencePipeline.md](03-ResiliencePipeline.md) | `Microsoft.Extensions.Resilience` (.NET 8+) — combining strategies. |
| [04-Retry.md](04-Retry.md) | Backoff strategies, jitter, idempotency, retry-and-poison. |
| [05-CircuitBreaker.md](05-CircuitBreaker.md) | State machine, open/half-open/closed, sliding window. |
| [06-Timeout.md](06-Timeout.md) | Pessimistic vs cooperative; CancellationToken propagation. |
| [07-Hedging.md](07-Hedging.md) | Hedged requests (Microsoft.Extensions.Resilience) — racing N copies. |
| [08-Patterns.md](08-Patterns.md) | Saga, compensation, outbox — application-level resilience. |
| [Questions.md](Questions.md) | Drilling. |
| [Coding.md](Coding.md) | Wrap HttpClient with retry+circuit+timeout. |

→ Begin: [01-WhyResilience.md](01-WhyResilience.md)
