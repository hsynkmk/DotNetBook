# 11 — Resilience

## ⚡ 30-second answer

**Resilience** is keeping the system working acceptably **despite** failures — distinct from
**reliability** (not failing in the first place). In distributed systems failure is the normal
case, and the danger to name is **cascading failure**: one slow dependency ties up a caller's
threads and connections, which exhausts *its* callers, and the whole system collapses. The
strategies map to failure modes: **retry** (transient blips, with **exponential backoff + jitter**,
and **only for idempotent operations**), **circuit breaker** (sustained failure — trip open and
fail fast so a dead dependency can't consume your resources), **timeout** (bound every outbound
call — an unbounded wait anywhere is a latent cascade), **bulkhead** (isolate pools), **fallback**
(degrade gracefully), **hedging** (tail latency). **Polly v8** via
`Microsoft.Extensions.Resilience` composes them, and **order matters**. But real resilience is
architectural: **idempotency**, the **outbox**, sagas, health checks and graceful shutdown.

---

## Core mechanics

### Match the strategy to the failure mode

| Failure mode | Strategy |
|---|---|
| Transient blip (network, transient deadlock, 5xx, 429) | **Retry** with backoff + jitter |
| Sustained outage | **Circuit breaker** — fail fast |
| Slow / hung dependency | **Timeout** (per-attempt **and** total) |
| Overload / noisy neighbour | **Bulkhead** — isolate resource pools |
| Non-critical dependency down | **Fallback** — degrade gracefully |
| Tail latency (p99 ≫ p50) | **Hedging** |

### Cascading failure — the thing you're defending against

A dependency gets slow (not down — *slow*). Your calls to it don't return, so request threads and
connection-pool slots pile up waiting. Your service stops responding to *everything*, including
requests that never touch that dependency. Your callers then do the same. One slow component takes
down the system.

The cure is to turn slow failures into **fast** ones: **timeouts** bound the wait, the **circuit
breaker** stops calling entirely once it's clearly broken, **bulkheads** confine the damage to one
pool, and a **fallback** keeps the rest of the feature working.

### Polly v8 pipelines

```csharp
builder.Services.AddResiliencePipeline("db", b => b
    .AddTimeout(TimeSpan.FromSeconds(30))                     // total
    .AddRetry(new RetryStrategyOptions
    {
        MaxRetryAttempts = 3,
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true,                                     // ← essential with many clients
        ShouldHandle = new PredicateBuilder().Handle<TimeoutException>()
    })
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions
    {
        FailureRatio = 0.5, MinimumThroughput = 10,
        SamplingDuration = TimeSpan.FromSeconds(30),
        BreakDuration = TimeSpan.FromSeconds(15)
    })
    .AddTimeout(TimeSpan.FromSeconds(5)));                    // per-attempt
```

**The standard order, outermost to innermost**: total timeout → retry → circuit breaker →
per-attempt timeout → the call. Changing it changes behavior subtly — see the traps.

**`ShouldHandle` predicates matter**: retry 5xx, timeouts and 429; **never retry 4xx**, which
won't succeed on a second try.

**Register pipelines once in DI.** Circuit-breaker state lives in the pipeline, so a pipeline
constructed per call **never trips** — every call gets a fresh, closed circuit.

For outbound HTTP, prefer **`AddStandardResilienceHandler`** ([09](09-Http-gRPC-SignalR.md)); use
custom pipelines for non-HTTP operations.

### Retry

Exponential backoff gives a struggling service room to recover. **Jitter** prevents synchronized
retry storms — without it, every client that failed during an outage retries at the same instant
and knocks the service over again just as it comes back.

**The hazard**: retrying a non-idempotent operation duplicates it. If the first attempt actually
succeeded but the response was lost, retrying charges the card twice. Retry only idempotent
operations, or use an **idempotency key** the server deduplicates on — that's the correct way to
retry a payment.

Honor **`Retry-After`** on a 429; the server is telling you exactly how long to wait.

### Circuit breaker

A **Closed → Open → Half-open** state machine. Closed = calls pass through, failures counted.
Once the failure ratio over the sampling window crosses the threshold, it trips **Open** — calls
fail **instantly** without touching the dependency. After the break duration it goes
**Half-open** and lets a trial call through: success closes it, failure re-opens it.

The value is turning slow, resource-consuming failures into instant ones, freeing your threads
and connections so one dead dependency can't exhaust you.

Tune per dependency: `FailureRatio`, `MinimumThroughput` (don't trip on 2 out of 2 requests),
`SamplingDuration`, `BreakDuration`. **A separate breaker per dependency** — otherwise one bad
downstream trips calls to a healthy one. Pair it with a **fallback** so an open circuit degrades
rather than errors, and **alert on circuit-open events**.

### Timeout

.NET cancellation is **cooperative**. A timeout cancels a `CancellationToken`, but the operation
must **observe** it to actually stop. **If you don't propagate the token all the way down to the
real I/O, the timeout is cosmetic** — your caller gets a `TimeoutException` while the work keeps
running and keeps holding its connection.

Use both a **per-attempt** timeout and a **total** timeout, and size the total to fit the retry
policy — a 5s total around 3 retries with backoff will always fire during the second attempt.

### Hedging

Fire additional parallel attempts after a short delay and take the first response, cancelling the
rest. Unlike retry (which waits for failure), hedging targets **slow** requests proactively.

It trades **extra load** for lower tail latency, so it needs spare capacity, a genuine tail-latency
problem, and multiple replicas. On a near-capacity system it causes the overload it was meant to
hide. Attempts run concurrently, so the operation **must be idempotent** — **hedge reads, not
writes**. Set the delay **above median latency** so you only hedge genuinely slow requests.

### Architectural resilience

Call-level strategies aren't enough:

- **Idempotency is the foundation.** Retry, hedging and at-least-once delivery all re-run
  operations — making them safe to repeat is what makes resilience *correct* rather than merely
  persistent ([07](07-Messaging.md)).
- **The outbox** makes a database change plus a published event atomic, with no distributed
  transaction.
- **Sagas with compensating actions** give recoverable multi-step workflows — and the failure
  paths are the design, not an afterthought.
- **Graceful degradation** is a per-dependency decision: which features must work when the
  recommendation service is down?
- **Health checks + graceful shutdown** make the system resilient to instance churn
  ([04](04-AspNetCore.md), [08](08-BackgroundProcessing.md)).

---

## 🪤 Traps & gotchas

- **Building the pipeline per call** — circuit-breaker state lives in the pipeline instance, so a
  fresh one is always closed and **the breaker never trips**. Register once in DI.
- **Retrying non-idempotent operations** — the request timed out but the server processed it.
  Retrying creates a second order or a second charge.
- **Retry without jitter** — synchronized retry storms. Every client backs off by the same
  exponential amount and hits the recovering service simultaneously.
- **Retrying 4xx** — a 400 or 404 will not succeed on the second attempt; you've just tripled load
  and latency for nothing. (429 is the exception — retry it, honoring `Retry-After`.)
- **Nested retries** — a resilience handler retrying 3× inside application code retrying 3× is 9
  requests, and neither layer can see the other. Retry at exactly one level.
- **Retry with no total timeout** — three retries with exponential backoff can take a minute, long
  after the caller gave up.
- **Circuit breaker inside the retry** instead of outside — the retries are counted as separate
  failures and it trips much sooner than you intended. (Standard order: retry outside, breaker
  inside.)
- **One shared circuit breaker for all dependencies** — service B being down stops your calls to
  healthy service C.
- **`MinimumThroughput` too low** — the breaker trips on 1 failure out of 2 requests during a quiet
  period, when nothing is actually wrong.
- **`BreakDuration` too long** — the dependency recovered in 5 seconds and you kept failing calls
  for 5 minutes.
- **A timeout without propagating the `CancellationToken`** — the cardinal resilience bug. The
  caller gets a `TimeoutException`, the work carries on in the background, and you leak a thread
  and a connection per timed-out request. This is how a timeout *causes* the exhaustion it was
  supposed to prevent.
- **Any unbounded wait** — an `HttpClient` with the default timeout, a database command with no
  timeout, a lock with no timeout. Every one is a latent cascade trigger.
- **Hedging writes** — parallel attempts of a non-idempotent operation means doing it twice, on
  purpose.
- **Hedging on a saturated system** — you've added load precisely when you had none to spare, and
  turned a latency problem into an outage.
- **Hedge delay below median latency** — you're now doubling traffic on every request rather than
  the slow tail.
- **A fallback that hides a real outage** — returning cached or empty data silently means nobody
  notices the dependency has been down for a day. Degrade *and* alert.
- **Catching and swallowing the `BrokenCircuitException`** without a deliberate degraded response.
- **Resilience without idempotency** — you've made the system persistent at doing the wrong thing
  twice.
- **No telemetry on retries or circuit state** — you can't tell a healthy system from one that's
  retrying every call and just barely holding on ([12](12-Observability.md)).

---

## ❓ Likely questions

**Q: Resilience vs reliability?**
A: Reliability is not failing; resilience is continuing to work acceptably when something does
fail. In a distributed system dependency failure is inevitable, so you need resilience regardless
of how reliable each component is.

**Q: What is cascading failure and how do you prevent it?**
A: A slow dependency causes callers to accumulate blocked threads and connections until they stop
serving anything; their callers then do the same and the system collapses. Prevent it by turning
slow failures into fast ones — timeouts on every call, a circuit breaker to stop calling a broken
dependency, bulkheads to isolate pools, and fallbacks to degrade.

**Q: How does a circuit breaker work?**
A: Closed, it passes calls and counts failures. When the failure ratio over the sampling window
crosses a threshold it trips **Open** and fails calls instantly without contacting the
dependency. After the break duration it goes **Half-open** and lets a trial call through — success
closes it, failure re-opens it.

**Q: Why does a circuit breaker help rather than just failing faster?**
A: Two reasons. It frees *your* resources — you're no longer holding threads and connections
waiting on something that won't answer. And it takes load off the struggling dependency, giving it
room to recover instead of being hammered while it's down.

**Q: What's the correct order of resilience strategies?**
A: Outermost to innermost: total timeout, retry, circuit breaker, per-attempt timeout, then the
call. The total timeout bounds everything including backoff; retry sits outside the breaker so the
breaker sees one logical call's outcome rather than counting each retry.

**Q: When is it safe to retry?**
A: When the operation is idempotent — GET, PUT, DELETE, or a write with an idempotency key the
server deduplicates on. A plain POST that times out may already have succeeded, so retrying
duplicates it.

**Q: Why does retry need jitter?**
A: Without it every client that failed during an outage computes the same backoff and retries at
the same moment — a synchronized thundering herd that knocks the service over again as it
recovers. Jitter randomizes the delay and spreads the load.

**Q: What's the trap with timeouts in .NET?**
A: Cancellation is cooperative. A timeout cancels a token, but if you don't pass that token all
the way down to the actual I/O call, nothing stops — the caller sees a timeout while the work keeps
running and holding its connection. So a timeout that isn't propagated actively makes exhaustion
worse.

**Q: Why do you need both a per-attempt and a total timeout?**
A: The per-attempt timeout bounds one try so a hung call doesn't block a retry from happening. The
total bounds the whole operation including all retries and backoff, so the caller has a real upper
bound on latency.

**Q: What is hedging and when would you use it?**
A: Firing extra parallel attempts after a short delay and taking the first response. It targets
tail latency proactively rather than waiting for a failure. Use it only when you have spare
capacity, multiple replicas, a measured p99 problem, and an idempotent (read) operation.

**Q: What is a bulkhead?**
A: Isolating resource pools so one dependency can't consume all of them — separate connection
pools or bounded concurrency per downstream. Named after ship compartments: one flooded section
doesn't sink the vessel.

**Q: Is Polly enough to make a system resilient?**
A: No — it handles the call level. Real resilience also needs idempotent operations (because retry
and at-least-once delivery re-run things), the outbox for atomic state-change-plus-publish, sagas
with compensation for multi-step workflows, and health checks with graceful shutdown for instance
churn.

**Q: How do you decide what to degrade?**
A: Per dependency, as a product decision. Classify each as critical (the feature can't work
without it) or non-critical (serve stale or empty data and carry on). The recommendations panel can
disappear; the checkout can't.

---

## 🎓 Senior Extra

- **The most valuable resilience change in most systems is a timeout that was missing**, not a
  clever policy. Audit every outbound call — HTTP, database, cache, broker, lock — for an
  explicit bound.
- **Retry budgets** cap total retries as a fraction of traffic (say 10%), which prevents retries
  from becoming a self-inflicted DDoS during a partial outage. More robust than per-call retry
  counts at scale.
- **Circuit-breaker metrics are the leading indicator.** Circuit-open events, retry counts, and
  the ratio of retried to first-attempt-successful calls tell you the system is degrading while
  latency still looks fine.
- **Load shedding is the strategy people forget**: when you're saturated, rejecting some requests
  fast (429) preserves service for the rest. Rate limiting at the edge is the same idea applied
  proactively ([04](04-AspNetCore.md)).
- **`Microsoft.Extensions.Resilience` binds pipeline options to configuration**, so you can tune
  break duration and retry counts per environment and validate them at startup — far better than
  constants compiled into the code ([03](03-Hosting-DI-Config.md)).
- **The idempotency key belongs to the client, not the server.** The client generates it once for
  a logical operation and reuses it across every retry — which is what makes retry-after-timeout
  correct rather than hopeful.
- **Compensation is not rollback.** A saga's compensating action is a new business fact (a refund,
  a cancellation) with its own audit trail, not an undo — and it can fail too, which is why
  compensations need to be idempotent and retryable.
- **Timeouts should shrink as you go deeper.** If the edge has a 3-second budget, a service three
  hops in should not have a 30-second timeout — that's what gRPC deadline propagation formalizes
  ([09](09-Http-gRPC-SignalR.md)).
- **Test the failure paths.** Chaos experiments, or at minimum an integration test with a
  Testcontainer you stop mid-test, are the only way to know your fallback actually works
  ([17](17-Testing.md)).
- **A half-open circuit is a single point of contention** under high concurrency — the trial call
  matters, so make sure your breaker only admits one and doesn't stampede the recovering
  dependency with the full load the instant it closes.

→ Deeper: [`../11-Resilience/`](../11-Resilience/README.md)
