# Application-Level Resilience Patterns

## Beyond Polly — resilience in the architecture

Polly strategies (retry, breaker, timeout — [02](02-Polly.md)) handle resilience at the **call** level. But true resilience is also an **architectural** concern: how do you keep data consistent and operations recoverable across services when failures are inevitable? This file covers the application-level patterns — **idempotency**, the **outbox**, **sagas/compensation**, and graceful degradation — that complement the call-level strategies. (These are introduced for messaging in [Ch07 §07](../07-Messaging/07-Patterns.md) and background work in [Ch08 §08](../08-BackgroundProcessing/08-ReliabilityAndScale.md); here they're framed as resilience.)

---

## Idempotency — the foundation

Because retries, redeliveries, and hedging make operations run **more than once** ([04-Retry.md](04-Retry.md), [Ch07 §07](../07-Messaging/07-Patterns.md)), **idempotency** is the bedrock of resilient design: running an operation twice must have the same effect as once.

```csharp
// Idempotency key — dedupe at the boundary so a retried request is processed once
public async Task<Result> ProcessAsync(string idempotencyKey, Command cmd) {
    if (await _store.WasProcessedAsync(idempotencyKey)) return await _store.GetResultAsync(idempotencyKey);
    var result = await DoWorkAsync(cmd);
    await _store.RecordAsync(idempotencyKey, result);   // atomically with the work
    return result;
}
```

Without idempotency, every resilience mechanism that re-runs work (retry, hedging, at-least-once delivery) becomes a **correctness hazard** (double charges, duplicate orders). Make operations idempotent (dedup keys, natural idempotency like "set to X", or conditional writes), and resilience becomes safe. This is the single most important application-level resilience practice.

---

## The outbox — reliable side effects

A common resilience failure: an operation updates the database **and** must trigger a side effect (publish an event, send a message), but the two aren't atomic — a crash between them loses the side effect or creates a phantom one. The **outbox pattern** ([Ch07 §07](../07-Messaging/07-Patterns.md), [Ch05 §08](../05-EFCore/08-Transactions.md)) makes them atomic:

```
In ONE DB transaction: write the business change AND insert the event into an outbox table → commit
A background relay reliably publishes outbox rows (retrying until success) → marks them sent
```

This guarantees the DB change and its side effect happen together (or neither) — no lost events, no distributed transaction. It's a **resilience** pattern because it makes cross-system effects survive crashes. Pair with idempotent consumers (the events are delivered at-least-once).

---

## Sagas & compensation — recoverable workflows

A multi-step workflow across services (order → payment → inventory → shipping) can't be a single transaction. A **saga** ([Ch07 §07](../07-Messaging/07-Patterns.md)) makes it resilient: each step is a local transaction + a message, and on failure, **compensating actions** undo prior steps:

```
ChargePayment ✓ → ReserveInventory ✗ (out of stock)
  → COMPENSATE: RefundPayment, CancelOrder   (undo what succeeded)
```

Sagas give **eventual consistency** with recoverability — instead of a doomed distributed transaction, the system reaches a consistent state via compensation when a step fails. The resilience angle: design the **failure paths** (compensations), not just the happy path, so a partial failure doesn't leave inconsistent state. (Avoid distributed transactions — [Ch05 §08](../05-EFCore/08-Transactions.md).)

---

## Graceful degradation — partial functionality over total failure

Resilient systems **degrade** rather than collapse when a dependency fails. Decide, per dependency, what happens when it's unavailable:

```csharp
// A non-critical dependency (recommendations) is down → degrade, don't fail the whole page
try {
    recommendations = await recoService.GetAsync(userId, ct);
} catch (BrokenCircuitException) {
    recommendations = [];   // empty/cached — the page still renders without recommendations
}
```

- **Critical dependencies** (the payment processor for a checkout) — can't degrade; fail the operation clearly.
- **Non-critical dependencies** (recommendations, related items, analytics) — degrade: serve cached/default/partial results so the core function still works.

Combine with **fallbacks** ([05-CircuitBreaker.md](05-CircuitBreaker.md)) and **caching** ([Ch06](../06-DataAndCaching/README.md)) — e.g., serve stale cached data when the source is down. Graceful degradation is a deliberate design decision per feature: what's the acceptable reduced experience?

---

## Health checks & graceful shutdown (recap)

Two infrastructure-level resilience mechanisms (covered elsewhere):
- **Health checks** ([Ch04 §16](../04-AspNetCore/16-HealthChecks.md)) — let orchestrators route around unhealthy instances (readiness) and restart dead ones (liveness), so the *system* stays available even as individual instances fail.
- **Graceful shutdown** ([Ch08 §08](../08-BackgroundProcessing/08-ReliabilityAndScale.md)) — drain in-flight work on SIGTERM (deploys/scale-downs) so a routine restart doesn't drop requests/work.

These make the system resilient to the *normal* churn of deploys, scaling, and instance failures — not just dependency faults.

---

## Putting it together: layered resilience

A resilient system layers these:

```
Call level:      timeout (bound waits) + retry (transient) + circuit breaker (outages) + fallback (degrade)
Operation level: idempotency (safe to re-run) + outbox (reliable side effects)
Workflow level:  sagas + compensation (recoverable multi-step consistency)
System level:    health checks + graceful shutdown + redundancy/replicas
```

No single layer suffices: Polly handles call-level faults, but without idempotency, retries corrupt data; without the outbox, side effects are lost; without sagas, multi-step workflows leave inconsistent state; without health checks/graceful shutdown, instance churn drops work. Resilience is the **combination**, applied deliberately.

---

## Common gotchas

### Call-level resilience without idempotency

Retry/hedging re-run operations — without idempotency, they corrupt data (double charges). Idempotency is the foundation that makes retries safe.

### Dual-write without the outbox

Updating the DB then publishing an event (or vice versa) loses/duplicates the side effect on a crash. Use the outbox for atomic DB-change + side-effect.

### Sagas without compensation

Designing only the happy path leaves inconsistent state on partial failure. Define compensating actions for each step.

### Failing the whole operation for a non-critical dependency

Letting a recommendations outage fail the entire checkout is poor resilience. Degrade gracefully — serve the core function without the optional part.

### Relying only on Polly

Call-level strategies don't make multi-step workflows or side effects resilient. Add idempotency, outbox, sagas, and system-level mechanisms.

---

## Summary

- True resilience is **architectural**, not just call-level (Polly): it requires application-level patterns.
- **Idempotency** is the foundation — since retry/hedging/at-least-once re-run operations, making them safe to repeat (dedup keys, natural idempotency, conditional writes) is what makes resilience *correct*.
- The **outbox** makes DB-change + side-effect atomic (no lost/phantom events, no distributed transaction); **sagas + compensation** give recoverable, eventually-consistent multi-step workflows (design the failure paths).
- **Graceful degradation** keeps core functionality working when non-critical dependencies fail (fallbacks, cached data) — a per-dependency decision; **health checks + graceful shutdown** make the system resilient to instance churn.
- Resilience is the **combination** of call-level (timeout/retry/breaker/fallback), operation-level (idempotency/outbox), workflow-level (sagas), and system-level (health/shutdown) mechanisms, applied deliberately.

→ Next: [Questions.md](Questions.md)
