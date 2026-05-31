# Chapter 11 — Resilience — Q & A

---

### Q1. Resilience vs reliability?

**Reliability** = the system doesn't fail (availability, correctness — via redundancy, testing). **Resilience** = the system recovers from / tolerates failure gracefully (via retry, circuit breakers, timeouts, fallbacks). You can't make dependencies perfectly reliable, so you build resilience to handle their inevitable failures.

---

### Q2. What is cascading failure and how do you prevent it?

One slow/failing dependency ties up a caller's threads/connections (waiting on doomed calls), exhausting its pools, which exhausts *its* callers — the failure spreads system-wide. Prevent it with **timeouts** (don't wait forever), **circuit breakers** (fail fast when a dependency is down), **bulkheads** (isolate), and **fallbacks** (degrade).

---

### Q3. Match failure modes to strategies.

Transient blip → **retry** (with backoff). Sustained outage → **circuit breaker**. Slow response → **timeout**. Overload → **bulkhead/rate limiter**. Non-critical dependency down → **fallback**. Tail latency → **hedging**.

---

### Q4. What is Polly?

The de-facto .NET resilience library (v8+, integrated into `Microsoft.Extensions.Resilience`) providing composable strategies (retry, circuit breaker, timeout, fallback, rate limiter, hedging) wrapped around any operation via a `ResiliencePipeline`. It powers the HTTP standard resilience handler.

---

### Q5. Why does strategy order matter in a Polly pipeline?

Strategies nest like middleware. The standard order is total timeout → retry → circuit breaker → per-attempt timeout → call. A circuit breaker *inside* retry counts each retry; *outside* retry counts each logical operation — very different behavior. Reason about what each layer should see.

---

### Q6. What do predicates (`ShouldHandle`) do?

They define **what counts as a failure** for a strategy — so you retry only transient failures (network, timeout, 5xx, 429), not permanent ones (4xx). Without correct predicates, you retry things that won't recover (a 404) or treat success as failure.

---

### Q7. What does Microsoft.Extensions.Resilience add over raw Polly?

DI-integrated **named pipelines** (`AddResiliencePipeline`), **config-bound** settings (per-environment tuning), and **built-in telemetry** (metrics/traces, OpenTelemetry-ready) — plus the HTTP `AddStandardResilienceHandler`. The recommended way to apply resilience in modern apps.

---

### Q8. Three backoff strategies, and which to use?

Constant (same delay), linear (1s, 2s, 3s), and **exponential** (1s, 2s, 4s, 8s — the default). Exponential backs off quickly to give a struggling service room while recovering transient blips fast. Cap the max delay.

---

### Q9. What is jitter and why is it essential?

Randomized retry delays. Without it, many clients that failed simultaneously retry in **lockstep** — synchronized waves (a retry storm/thundering herd) that crush a recovering service. Jitter spreads retries over time. Always combine backoff with jitter in multi-client systems.

---

### Q10. What's the danger of retrying non-idempotent operations?

If the first attempt **succeeded** but the response was lost (network drop, timeout after processing), the retry runs the operation **again** → duplicates (double charge, duplicate order). Retry only idempotent operations, or use an **idempotency key** the server dedupes on.

---

### Q11. Which failures should you retry vs not?

**Retry**: transient — network errors, timeouts, 5xx, 429 (honor `Retry-After`). **Don't retry**: 4xx client errors (400/401/403/404) — they won't fix themselves; retrying is pointless or harmful.

---

### Q12. Why pair retry with a circuit breaker?

Retry handles transient blips but **worsens** sustained outages (it piles load on a down service). The circuit breaker trips after persistent failures and stops the retries (fail fast) until the service recovers. Retry for blips, breaker for outages — the standard pairing.

---

### Q13. Describe the circuit breaker state machine.

**Closed** (normal — calls flow, track failures) → trips to **Open** (fail fast, no call to the dependency, for a break duration) → **Half-open** (allow one trial call) → success → Closed, failure → Open. It self-heals: open, wait, test, recover.

---

### Q14. How does a circuit breaker prevent cascading failure?

When open, it turns slow, resource-consuming failures into **instant** failures (no call, no waiting) — freeing the caller's threads/connections so a dead dependency can't exhaust the caller's pool. The caller fails fast on that dependency while staying responsive overall.

---

### Q15. How do you tune a circuit breaker?

`FailureRatio` (proportion that trips it), `MinimumThroughput` (min calls before evaluating — so 1 failure doesn't trip it), `SamplingDuration` (the rolling window), `BreakDuration` (how long to stay open). Tune per dependency: too sensitive → needless outages; too lenient → no protection.

---

### Q16. Why must a circuit breaker pipeline be built once, not per call?

The breaker's **state is shared across calls** (one call's failures inform the next) — that's the point. Building a new pipeline per call resets the state, so it never trips. Build/register it once (DI) and reuse; use a separate breaker per dependency.

---

### Q17. Why does every outbound call need a timeout?

An unbounded wait on a hung dependency ties up a thread/connection indefinitely; enough of these exhaust the pool and cascade. Timeouts bound the wait so the caller frees resources — the most fundamental anti-cascade measure.

---

### Q18. Per-attempt vs total timeout?

**Per-attempt** bounds a single try (so one hung request doesn't stall before a retry). **Total** bounds the whole operation including all retries + backoff (so the retry chain doesn't exceed the caller's deadline). Use both; size the total to fit the retry policy.

---

### Q19. Why is cooperative cancellation critical to timeouts?

.NET cancellation is cooperative — a timeout *signals* a `CancellationToken`, but the operation must **observe and honor** it to actually stop. If the operation ignores the token (blocking I/O), the timeout reports a failure but the work **keeps running** (a background leak). Propagate the token through the whole chain to the real I/O.

---

### Q20. What is hedging and what does it target?

Hedging fires **additional parallel attempts** after a short delay and uses the first to respond (canceling the rest) — targeting **tail latency** (slow requests), not failures. Unlike retry (sequential, after failure), hedging is proactive and parallel, for slowness.

---

### Q21. When is hedging appropriate, and what's the cost?

Appropriate when tail latency (p99) matters, the backend has **spare capacity**, there are **multiple replicas**, and operations are **idempotent**. The cost is **extra load** (2–3× traffic) — on a near-capacity system hedging causes overload. Hedge reads, not writes; set the delay above median latency.

---

### Q22. Why must hedged operations be idempotent?

Hedging runs the operation **concurrently** multiple times — a hedged non-idempotent write causes duplicate side effects (double charge). Hedge only idempotent operations (reads, or idempotent-keyed writes). In practice, hedge reads, not writes.

---

### Q23. Why is idempotency the foundation of resilience?

Retry, hedging, and at-least-once delivery all **re-run** operations. Without idempotency, every such mechanism becomes a correctness hazard (duplicates). Making operations safe to repeat (dedup keys, natural idempotency, conditional writes) is what makes resilience *correct*, not just available.

---

### Q24. How does the outbox pattern contribute to resilience?

It makes a DB change and a side effect (publish an event) **atomic** by writing the event to an outbox table in the same transaction, then relaying it reliably. So a crash between the two can't lose the event or create a phantom one — cross-system effects survive failures, without a distributed transaction.

---

### Q25. What is graceful degradation?

Keeping core functionality working when a dependency fails — per dependency, decide the reduced experience: critical ones (payment) fail clearly; non-critical ones (recommendations) degrade (cached/default/empty results, via fallbacks). Partial functionality beats total failure.

---

### Q26. Why isn't Polly alone sufficient for resilience?

Polly handles **call-level** faults, but without **idempotency** retries corrupt data, without the **outbox** side effects are lost, without **sagas** multi-step workflows leave inconsistent state, and without **health checks/graceful shutdown** instance churn drops work. Resilience is the *combination* of call-level, operation-level, workflow-level, and system-level mechanisms.

---

→ Next: [Coding.md](Coding.md)
