# Hedging

## Racing requests to cut tail latency

**Hedging** attacks **tail latency** — the problem where *most* requests are fast but a *few* are very slow (the p99/p999). Instead of waiting out a slow request, hedging sends **additional parallel attempts** after a short delay and uses whichever responds first. It trades extra load for lower worst-case latency. It's a newer, more specialized strategy in `Microsoft.Extensions.Resilience` ([03-ResiliencePipeline.md](03-ResiliencePipeline.md)).

```csharp
.AddHedging(new HedgingStrategyOptions<HttpResponseMessage> {
    MaxHedgedAttempts = 2,                         // up to 2 extra parallel attempts
    Delay = TimeSpan.FromMilliseconds(200),         // if no response in 200ms, fire another attempt
    ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
        .HandleResult(r => r.StatusCode >= HttpStatusCode.InternalServerError)
})
```

---

## How it works

```
T=0ms    → send attempt 1
T=200ms  → attempt 1 hasn't responded → send attempt 2 (in parallel)
T=400ms  → still nothing → send attempt 3
         → use whichever attempt responds FIRST; cancel the others
```

Hedging fires a backup request if the primary is taking too long (past the `Delay`), runs them concurrently, and takes the **first** success — canceling the slower ones. The insight: when latency is variable, a *retry of a slow request* is often faster than *waiting for the slow one to finish*. It directly targets the slow-tail, where a small fraction of requests dominate the worst-case experience.

Hedging vs retry: **retry** waits for failure, then tries again (sequential, for *failures*); **hedging** doesn't wait for failure — it fires extra attempts proactively while the first is still pending (parallel, for *slowness*).

---

## When hedging helps (and the cost)

Hedging is valuable when:
- **Tail latency matters** — p99/p999 is much worse than the median, and you need consistent low latency (user-facing reads, latency-SLA services).
- **The backend has spare capacity** and **idempotent** operations.
- **Multiple replicas** exist — a hedged request can hit a *different*, faster replica.

The cost: **extra load.** Every hedged attempt is a real request, so hedging multiplies traffic (potentially 2–3×). On a system that's already near capacity, hedging makes things **worse** (more load → slower → more hedging → overload). So hedging suits systems with **headroom** and a real tail-latency problem — not as a default. Tune `Delay` (fire hedges only for genuinely slow requests, not the median) and cap `MaxHedgedAttempts` to bound the load multiplier.

---

## Idempotency (again, critical)

Like retry ([04-Retry.md](04-Retry.md)), hedging runs the operation **more than once** — concurrently. So the operation **must be idempotent**, or you'll get duplicate side effects:

```
Hedged POST /charge → attempt 1 AND attempt 2 both reach the server → DOUBLE CHARGE
```

Hedging is safe only for **idempotent** operations (reads, GETs, idempotent-keyed writes). Never hedge non-idempotent writes without an idempotency key ([Ch07 §07](../07-Messaging/07-Patterns.md)) — and even then, concurrent duplicates stress the dedup logic. In practice, **hedge reads, not writes.**

---

## Hedging vs retry vs the others

| Strategy | Targets | Mechanism | When |
|---|---|---|---|
| **Retry** | transient *failures* | sequential re-attempt after failure | blips, transient errors |
| **Hedging** | tail *latency* (slowness) | parallel extra attempts proactively | p99 too high, spare capacity, idempotent reads |
| **Timeout** | hung operations | abandon after a deadline | always |
| **Circuit breaker** | sustained outages | fail fast | dependency down |

Hedging is the specialized tool for **latency**, not failures — use it specifically when consistent low latency matters more than minimizing load, on idempotent operations, with capacity to spare. For most apps, retry + circuit breaker + timeout cover the common needs; add hedging only for measured tail-latency problems.

---

## Common gotchas

### Hedging non-idempotent operations

Concurrent hedged attempts of a write **duplicate** side effects (double charge/order). Hedge **reads** (or idempotent-keyed writes) only.

### Hedging a near-capacity system

Hedging multiplies load; on a system already near its limit it causes overload (more load → slower → more hedges → collapse). Use it only where there's headroom.

### `Delay` too short → hedging everything

A tiny delay fires hedges for *median* requests too, multiplying nearly all traffic. Set the delay above the typical (median/p50) latency so only genuinely slow requests get hedged.

### Using hedging as a default resilience strategy

It's specialized for tail latency, not general failure handling. Use retry/breaker/timeout for failures; add hedging only for a measured latency problem.

### Forgetting to cancel losing attempts

The strategy cancels slower attempts once one succeeds — but the operation must honor cancellation ([06-Timeout.md](06-Timeout.md)) so the canceled attempts actually stop (and don't keep loading the backend).

---

## Summary

- **Hedging** cuts **tail latency** by firing **additional parallel attempts** after a short delay and using the first to respond (canceling the rest) — targeting slow requests proactively, unlike retry (which waits for failure).
- It trades **extra load** for lower worst-case latency — suitable only for systems with **spare capacity**, a real **tail-latency** problem, and **multiple replicas**; on a near-capacity system it causes overload.
- Hedged attempts run **concurrently**, so the operation **must be idempotent** — **hedge reads, not writes** (or idempotent-keyed writes only).
- Tune the **`Delay`** above median latency (hedge only genuinely slow requests) and cap `MaxHedgedAttempts`; it's a **specialized** tool — use retry/breaker/timeout for failures, add hedging only for measured latency needs.

→ Next: [08-Patterns.md](08-Patterns.md)
