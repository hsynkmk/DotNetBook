# Circuit Breaker

## Fail fast when a dependency is down

When a downstream service is genuinely down or overloaded, **retrying every request just makes it worse** — you pile load on a struggling service, and every caller waits through timeouts. A **circuit breaker** detects sustained failure and "trips open" — short-circuiting subsequent calls (failing them **immediately**, without even trying) for a cooldown period — then cautiously tests whether the service has recovered. It's the antidote to cascading failure ([01-WhyResilience.md](01-WhyResilience.md)).

```csharp
.AddCircuitBreaker(new CircuitBreakerStrategyOptions {
    FailureRatio = 0.5,                          // trip if >50% of calls fail
    MinimumThroughput = 10,                      // ...over at least 10 calls (avoid tripping on 1 failure)
    SamplingDuration = TimeSpan.FromSeconds(30), // ...within this rolling window
    BreakDuration = TimeSpan.FromSeconds(15)     // stay open (fail fast) for 15s, then test recovery
})
```

---

## The state machine

A circuit breaker is a three-state machine, named after an electrical breaker:

```
   ┌──────────┐  failure ratio exceeds threshold   ┌────────┐
   │  CLOSED   │ ─────────────────────────────────▶ │  OPEN   │
   │ (normal:  │                                     │ (fail   │
   │  calls    │ ◀───────── trial call succeeds ──── │  fast,  │
   │  flow,    │                                     │  no     │
   │  track    │     ┌───────────┐  after BreakDuration  call) │
   │  failures)│ ◀── │ HALF-OPEN  │ ◀────────────────────┘
   └──────────┘ fail │ (one trial │
                 ──▶ │  call)      │ ── succeed ──▶ CLOSED
                     └───────────┘ ── fail ──────▶ OPEN
```

- **Closed** (normal) — calls flow through; the breaker tracks the failure rate.
- **Open** (tripped) — the failure threshold was exceeded; calls **fail immediately** (throwing `BrokenCircuitException`) **without calling the dependency** — for the `BreakDuration`. This protects the failing service (no load) and the caller (no waiting).
- **Half-open** (testing) — after the break duration, the breaker allows **one trial call**: if it succeeds → back to **Closed** (recovered); if it fails → back to **Open** (still down, wait again).

This automatic open→test→recover cycle lets the system self-heal without manual intervention.

---

## Why it prevents cascading failure

The circuit breaker breaks the cascade described in [01-WhyResilience.md](01-WhyResilience.md):

```
Without a breaker: B is down → every call to B waits through its timeout (e.g., 5s) → A's threads pile up
                   waiting on doomed calls → A's pool exhausts → A goes down too (cascade).

With a breaker:    B fails repeatedly → breaker OPENS → calls to B fail INSTANTLY (no 5s wait) →
                   A's threads aren't tied up → A stays responsive (fails fast on B-dependent work,
                   serves everything else) → no cascade. When B recovers, the breaker closes.
```

The key insight: an open breaker turns slow, resource-consuming failures into **instant** failures, freeing the caller's threads/connections so one dead dependency can't exhaust the caller. This is the single most important pattern for containing failure in microservices.

---

## Tuning the threshold

The breaker shouldn't trip on a single failure (transient blips happen) nor be so lenient it never trips. The parameters define "sustained failure":

```csharp
FailureRatio = 0.5,                          // trip when failures exceed 50% ...
MinimumThroughput = 20,                      // ... but only after at least 20 calls (statistical significance)
SamplingDuration = TimeSpan.FromSeconds(30), // ... within a rolling 30s window
BreakDuration = TimeSpan.FromSeconds(30)     // stay open 30s before testing
```

- **`FailureRatio`** — the failure proportion that trips it (0.5 = half).
- **`MinimumThroughput`** — minimum calls in the window before the ratio is evaluated (so 1 failure out of 1 call doesn't trip it).
- **`SamplingDuration`** — the rolling window over which the ratio is measured.
- **`BreakDuration`** — how long to stay open before a trial call.

Tune these per dependency: a critical, usually-reliable service might trip at a lower ratio; a flaky one might need a higher threshold to avoid constant tripping. Too sensitive → unnecessary outages; too lenient → no protection.

---

## Handling the open circuit (fallback)

When the circuit is open, calls fail immediately with `BrokenCircuitException`. You should handle this gracefully — often with a **fallback** (degrade rather than error):

```csharp
try {
    return await pipeline.ExecuteAsync(t => CallServiceAsync(t), ct);
} catch (BrokenCircuitException) {
    return await GetCachedFallbackAsync();   // serve stale/cached data while the circuit is open
}
// or compose a Fallback strategy into the pipeline directly
```

An open circuit is a chance to **degrade gracefully** — serve cached data, a default, or a partial response — rather than propagating an error. (Not everything can fall back: a payment can't, but a recommendations widget can.) Decide per dependency. Combine with retry ([04-Retry.md](04-Retry.md)): retry transient blips, but once the breaker opens, stop retrying and fall back.

---

## Per-dependency, shared state

Critically, the circuit breaker's **state is shared across all calls** through the pipeline — that's the whole point (one call's failures inform the next call's behavior). So:
- **Build the pipeline once** (DI-registered — [02-Polly.md](02-Polly.md)) and reuse it; a per-call pipeline resets the breaker state and renders it useless.
- Use a **separate breaker per dependency** — service B's failures shouldn't open the circuit for service C. Each downstream dependency gets its own named pipeline/breaker.

---

## Common gotchas

### Building the pipeline per call → no shared state

The breaker tracks failures **across** calls; a new pipeline per call resets that state, so it never trips. Build/register the pipeline once and reuse it.

### One breaker for multiple dependencies

A shared breaker means B's outage opens the circuit for unrelated C. Use a **separate breaker per dependency**.

### Threshold too sensitive or too lenient

Tripping on one transient failure causes unnecessary outages; never tripping provides no protection. Tune `FailureRatio`/`MinimumThroughput`/`SamplingDuration` per dependency.

### Retry without a breaker (or vice versa)

Retry alone hammers a sustained outage; a breaker alone doesn't smooth transient blips. Use **both**: retry for blips, breaker for outages.

### Not handling `BrokenCircuitException`

An open circuit throws — handle it (fallback to cache/default) so an open breaker degrades gracefully instead of surfacing raw errors.

### No telemetry on circuit state

A circuit opening is a major signal (a dependency is down). Wire `OnCircuitOpened`/telemetry to alerting ([Ch12](../12-Observability/README.md)) — don't let outages be silent.

---

## Summary

- A **circuit breaker** detects sustained failure and **trips open** to fail calls **instantly** (no call to the dependency) for a cooldown, then **half-opens** to test recovery — a self-healing **Closed → Open → Half-open** state machine.
- It **prevents cascading failure**: turning slow, thread-consuming failures into instant ones frees the caller's resources so one dead dependency can't exhaust it — the key microservices protection.
- **Tune** `FailureRatio` / `MinimumThroughput` / `SamplingDuration` / `BreakDuration` per dependency (don't trip on one blip, don't be too lenient); use a **separate breaker per dependency**.
- **State is shared across calls** — build/register the pipeline **once** (per-call pipelines never trip); handle the open circuit with a **fallback** (degrade gracefully).
- **Pair with retry** (retry blips, breaker for outages) and **alert on circuit-open** events.

→ Next: [06-Timeout.md](06-Timeout.md)
