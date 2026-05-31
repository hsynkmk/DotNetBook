# Why Resilience

## The world is unreliable

Distributed systems fail constantly in small ways: a network packet drops, a service is briefly overloaded, a database has a transient deadlock, a dependency is mid-deploy, a connection times out. **Resilience** is the discipline of designing systems that **keep working acceptably despite these failures** — recovering from transient faults, isolating failing components, and degrading gracefully rather than collapsing.

```
Without resilience: a 200ms blip in service B → requests to A pile up waiting → A's threads exhaust →
                    A starts failing too → the failure cascades across the system.

With resilience:    B blips → A retries transient failures, the circuit breaker trips if B is truly down,
                    timeouts free A's threads, a fallback degrades gracefully → A stays up.
```

In a microservices/cloud world, **failure is the normal case, not the exception** — a service with 50 dependencies, each 99.9% available, is itself far less than 99.9% available unless it handles dependency failures. Resilience is what makes a system robust at scale.

---

## Resilience vs reliability

These related terms mean different things:

- **Reliability** — the system *doesn't fail* (high availability, correctness). Achieved through redundancy, testing, good code, robust infrastructure.
- **Resilience** — the system *recovers from / tolerates failure* gracefully. Achieved through retry, circuit breakers, timeouts, fallbacks, bulkheads.

You can't make dependencies perfectly reliable (networks and remote services *will* fail), so you build **resilience** to handle their failures. Reliability reduces how often things break; resilience controls what happens *when* they break. A robust system needs both — but resilience is specifically about graceful behavior under failure.

---

## What can fail (the failure taxonomy)

Resilience strategies target different failure modes:

| Failure | Example | Strategy |
|---|---|---|
| **Transient** | a momentary network blip, transient DB deadlock | **retry** (with backoff) |
| **Sustained** | a dependency is down/overloaded for minutes | **circuit breaker** (fail fast, let it recover) |
| **Slow** | a dependency responds but very slowly | **timeout** (don't wait forever) |
| **Overload** | too many concurrent calls to one resource | **bulkhead** (bound concurrency) |
| **Total** | a non-critical dependency is unavailable | **fallback** (degrade gracefully) |
| **Tail latency** | most calls fast, some very slow | **hedging** (race a second call) |

The art is matching the strategy to the failure: retry helps a transient blip but *worsens* a sustained outage (more load on a struggling service) — that's what the circuit breaker is for. The rest of this chapter covers each strategy ([02-Polly.md](02-Polly.md) onward).

---

## The cascading failure problem

The reason resilience matters so much: failures **cascade**. A single slow/failing dependency can take down a whole system if unhandled:

```
Service B slows down (e.g., 5s instead of 50ms)
  → Service A's calls to B block, holding A's threads/connections
  → A's thread pool / connection pool exhausts ([Ch01 §08])
  → A can't serve ANY requests (even those not needing B) → A appears down
  → Services calling A now block and exhaust → the failure spreads upstream
  → the whole system is down because of one slow dependency
```

This is **cascading failure**, and it's why a single weak link can collapse a distributed system. Resilience patterns break the cascade:
- **Timeouts** stop A's threads from blocking forever on slow B.
- **Circuit breakers** make A fail fast (not wait) once B is clearly down.
- **Bulkheads** isolate B-related calls so they can't exhaust *all* of A's resources.
- **Fallbacks** let A keep serving (degraded) when B is unavailable.

Together they contain a failure to its blast radius instead of letting it spread.

---

## Resilience is a system property, not a library

While this chapter focuses on tools (Polly, the resilience pipeline — [02](02-Polly.md)/[03](03-ResiliencePipeline.md)), resilience is ultimately a **design property**:
- **Bound everything** — every outbound call has a timeout; every queue/pool has a limit. Unbounded waits and resources are how cascades happen.
- **Isolate failures** — bulkheads, separate pools, circuit breakers per dependency, so one failure doesn't sink everything.
- **Degrade gracefully** — decide per dependency what happens when it's down (fallback, cached data, partial response, or fail — a payment can't fall back, a recommendations widget can).
- **Design for at-least-once / idempotency** — retries and redeliveries mean operations run more than once ([Ch07 §07](../07-Messaging/07-Patterns.md), [Ch08 §08](../08-BackgroundProcessing/08-ReliabilityAndScale.md)).
- **Health checks + graceful shutdown** — let orchestrators route around and recover failed instances ([Ch04 §16](../04-AspNetCore/16-HealthChecks.md)).

The libraries implement the patterns; you must apply them deliberately across the system.

---

## Common gotchas

### Retrying a sustained outage

Retry helps transient blips but **piles load** on a service that's actually down, worsening and prolonging the outage. Pair retry with a **circuit breaker** (stop retrying when it's clearly down).

### Unbounded waits → cascading failure

A call with no timeout blocks a thread indefinitely when a dependency hangs; enough of these exhaust the pool and cascade. **Always timeout** outbound calls.

### No failure isolation

Without bulkheads/separate pools, one failing dependency can exhaust shared resources and take down unrelated functionality. Isolate by dependency.

### Confusing resilience with reliability

Making code "more reliable" (better testing, redundancy) doesn't handle the inevitable dependency failures — you also need resilience (retry/breaker/timeout/fallback) for graceful behavior under failure.

### Treating failure as exceptional

In distributed systems, failures are routine, not rare. Design for them as the normal case (every call can fail/be slow), not as an edge case to bolt on later.

---

## Summary

- **Resilience** = keeping the system working acceptably **despite** failures (recover from transient faults, isolate failing components, degrade gracefully) — distinct from **reliability** (not failing in the first place). You need both, but resilience handles the inevitable dependency failures.
- In distributed/cloud systems, **failure is the normal case** — match strategies to failure modes: **retry** (transient), **circuit breaker** (sustained), **timeout** (slow), **bulkhead** (overload), **fallback** (degrade), **hedging** (tail latency).
- The core danger is **cascading failure**: one slow/down dependency exhausts a caller's threads/pools, which exhausts *its* callers, collapsing the system. Timeouts, circuit breakers, bulkheads, and fallbacks contain the blast radius.
- Resilience is a **system design property**, not just a library: bound everything, isolate failures, degrade gracefully, design for idempotency, and use health checks + graceful shutdown.

→ Next: [02-Polly.md](02-Polly.md)
