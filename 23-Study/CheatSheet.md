# Cheat Sheet — Morning-Of Scan

> Every table in one place, the traps that cost offers, and the phrases that make an answer sound
> senior. 15 minutes, the morning of.

---

## 1. The tables

### DI lifetimes — and the captive dependency

| Lifetime | Instances | Thread-safe? | Use for |
|---|---|---|---|
| **Singleton** | one per app | **you must ensure it** | stateless services, caches |
| **Scoped** | one per scope/request | one request at a time | `DbContext`, per-request state |
| **Transient** | one per resolution | n/a | lightweight, stateless |

| Depends on ↓ | Singleton | Scoped | Transient |
|---|---|---|---|
| **Singleton** | ✅ | ❌ **captive** | ⚠️ held for app lifetime |
| **Scoped** | ✅ | ✅ | ⚠️ held for the scope |
| **Transient** | ✅ | ✅ | ✅ |

### Options accessors

| | Lifetime | Reloads | Use in |
|---|---|---|---|
| `IOptions<T>` | singleton | ❌ | the default |
| `IOptionsSnapshot<T>` | scoped | ✅ per request | web request code |
| `IOptionsMonitor<T>` | singleton | ✅ live | **singletons, background services** |

### Middleware order (memorize verbatim)

```text
ExceptionHandler → HSTS/HttpsRedirection → StaticFiles → Routing → CORS
  → Authentication → Authorization → RateLimiter/OutputCache → Endpoints
```

Routing before auth (metadata). Authentication before authorization (identity). Exception handler
outermost (catches everything).

### Minimal APIs vs MVC vs Razor Pages

| | Minimal APIs | MVC | Razor Pages |
|---|---|---|---|
| Unit | route-handler lambda | controller + actions | `.cshtml` + `PageModel` |
| Cross-cutting | endpoint filters | **rich filter pipeline** | page filters |
| Native AOT | **yes** (RDG) | no | no |
| Content negotiation | **JSON only** | yes | n/a |
| For | new APIs, microservices | large apps, views | server forms |

### EF Core loading & tracking

| Strategy | What | When |
|---|---|---|
| **Eager** `Include` | loads navigations with the query | you need the graph |
| **Lazy** proxies | on first access | **N+1 risk — avoid** |
| **Explicit** `Entry().Load` | on demand | selective |
| **Projection** `Select` | only needed columns | **read APIs — best** |

| | Tracked | `AsNoTracking()` |
|---|---|---|
| Snapshot / identity resolution | yes | no |
| `SaveChanges` sees changes | yes | **no** |

| Disconnected save | Tracks as | Writes |
|---|---|---|
| `Attach` | Unchanged | nothing until marked |
| `Update` | Modified (**all**) | **every column** |
| **load-then-mutate** | Unchanged → Modified | only real changes ✅ |

### Cache tiers

| | `IMemoryCache` | `IDistributedCache` | **`HybridCache`** |
|---|---|---|---|
| Shared across instances | ❌ | ✅ | ✅ |
| Cost per hit | none | serialize + network | L1: none |
| **Stampede protection** | ❌ | ❌ | **✅** |
| Tag invalidation | ❌ | ❌ | ✅ |

### Brokers

| | `Channel<T>` | RabbitMQ | Service Bus | Kafka |
|---|---|---|---|---|
| Durable | ❌ | ✅ | ✅ | ✅ |
| After consumption | gone | gone | gone | **retained, replayable** |
| Ordering | FIFO | per queue | **sessions** | **partitions** |
| For | in-app | task queues, routing | enterprise Azure | streams, replay |

**Send** = command → one consumer. **Publish** = event → all subscribers.
**All brokers are at-least-once → consumers must be idempotent.**

### Resilience strategies (and their order)

```text
total timeout → retry → circuit breaker → per-attempt timeout → the call
```

| Strategy | For | Caveat |
|---|---|---|
| **Retry** + backoff **+ jitter** | transient faults | **idempotent only** |
| **Circuit breaker** | sustained outage | build the pipeline **once** or it never trips |
| **Timeout** | slow dependencies | **propagate the token** or it's cosmetic |
| **Bulkhead** | overload | isolate pools per dependency |
| **Fallback** | degrade gracefully | degrade *and alert* |
| **Hedging** | tail latency | reads only; needs spare capacity |

### Auth: cookies vs bearer tokens

| | Cookie | JWT bearer |
|---|---|---|
| Sent | automatically | explicit header |
| Client can read | ❌ encrypted | ✅ **payload is readable** |
| Revocable | ✅ | ❌ until expiry |
| **CSRF risk** | **✅ needs antiforgery** | ❌ |
| For | server-rendered apps | APIs, SPAs, mobile |

**Validate on every JWT: signature, `exp`, `iss`, `aud`.**
**401** = not authenticated. **403** = authenticated, forbidden.

### Health probes

| Probe | Question | Failure means | Checks |
|---|---|---|---|
| **Liveness** | process alive? | **restart** | the app **only** |
| **Readiness** | can I serve? | **stop routing** | app + dependencies |
| **Startup** | finished booting? | keep waiting | slow init |

### Publish modes

| Mode | Startup | Reflection | For |
|---|---|---|---|
| Framework-dependent | baseline | full | controlled envs |
| **ReadyToRun** | **fast** | **full** | faster startup, no compat cost |
| Trimmed | baseline | ⚠️ static only | size |
| **Native AOT** | **instant** | ❌ restricted | CLIs, serverless |

### Observability pillars

| | Logs | Metrics | Traces |
|---|---|---|---|
| Role | **explain** | **detect / alert** | **localize** |
| Cardinality | unlimited | **must be low** | high is fine |

### Test database fidelity

| Approach | Verdict |
|---|---|
| Mock `DbContext` | ❌ LINQ-to-Objects, not LINQ-to-Entities |
| **InMemory** | ❌ not relational — no constraints, no translation |
| **SQLite in-memory** | ✅ sensible default (keep the connection open) |
| **Testcontainers** | ✅ real fidelity |

---

## 2. The top traps

**DI & configuration**
1. **Captive dependency** — scoped `DbContext` in a singleton or hosted service.
2. **`IOptionsSnapshot` in a singleton** — same bug, different hat. Use `IOptionsMonitor`.
3. **No `ValidateOnStart()`** — bad config surfaces in production, not at boot.
4. Env vars use **`__`**, not `:`. And they're **read once** — they never reload.
5. Missing config key returns **null silently**.

**ASP.NET Core**
6. **Middleware in the wrong order** — the #1 bug. Authz before authn; routing after auth.
7. **Scoped service in a middleware constructor** — inject into `InvokeAsync`.
8. **Binding to entities** → over-posting (`isAdmin: true`).
9. **Route constraints aren't validation** — a mismatch is **404**, not 400.
10. **Wrong output-cache vary-by** — everyone gets the first caller's response.
11. **Liveness that checks the database** → restart storm across the fleet.

**EF Core**
12. **N+1** — a navigation touched in a loop.
13. **Cartesian explosion** — two collection `Include`s. Use `AsSplitQuery`.
14. **`DbContext` isn't thread-safe** — "a second operation was started on this context".
15. **`Update()` overwrites every column.**
16. **No concurrency token** → silent lost updates.
17. **`Database.Migrate()` on startup across replicas** → they race.
18. **A rename becomes drop + add** → data loss. Review migrations.
19. **Interpolating into `FromSqlRaw`** → injection.
20. **`ExecuteUpdate`/`Delete` bypass interceptors** — soft delete and auditing don't run.

**HTTP & resilience**
21. **`new HttpClient()` per request** → socket exhaustion. **Static singleton** → stale DNS.
22. **`HttpClient` doesn't throw on 4xx/5xx.**
23. **Retrying a non-idempotent POST** → duplicates.
24. **Retry without jitter** → synchronized thundering herd.
25. **Pipeline built per call** → the circuit breaker never trips.
26. **Timeout without propagating the token** → the work keeps running. The timeout *causes* the
    exhaustion it was meant to prevent.

**Async & runtime**
27. **Sync-over-async** (`.Result`, `.Wait()`) → thread-pool starvation and deadlocks.
28. **`async void`** → uncatchable exceptions.
29. **Not propagating `CancellationToken`.**
30. **Boxing in a hot loop** → invisible Gen 0 pressure.

**Security**
31. **Secrets in a JWT** — the payload is base64, not encrypted.
32. **Not validating `aud`/`iss`** — a token for another service is accepted.
33. **Data Protection key ring not persisted** → random logouts across instances.
34. **Endpoints are public by default** — set a fallback policy.
35. **Fast hash for passwords**; **`==` for token comparison** (timing leak).

**Background & messaging**
36. **No scope per work item** in a hosted service.
37. **A scheduled `BackgroundService` runs in every replica.**
38. **Dual write** (save then publish) → use the **outbox**.
39. **Non-idempotent consumers** — delivery is at-least-once.
40. **Poison-message loop** — dead-letter after N attempts, and **monitor the DLQ**.

**Observability & caching**
41. **String interpolation in logs** → no queryable fields.
42. **High-cardinality metric dimensions** → time-series explosion.
43. **Unbounded `IMemoryCache`** → OOM.
44. **Cache key missing tenant/user** → cross-tenant data leak.
45. **Trace context lost across a broker** — propagation is manual over messaging.

---

## 3. Power phrases

Short, precise formulations that signal depth. Use them verbatim.

- **Captive dependency**: *"A singleton capturing a scoped service pins one instance for the app's
  lifetime — so per-request semantics break and a non-thread-safe object gets shared across
  threads. Scope validation catches it in Development."*
- **Middleware order**: *"Each middleware wraps the rest of the pipeline, so position determines
  what it can see. Routing has to come first so auth can read the endpoint's metadata, and
  authentication before authorization because there's nothing to authorize otherwise."*
- **`DbContext`**: *"It's a Unit of Work, and its `DbSet`s are repositories — which is why a
  generic `Repository<T>` over EF is usually redundant."*
- **N+1**: *"One query for the parents plus one per parent for the children. Fix with `Include`,
  or better, project to a DTO so the join happens in SQL."*
- **Socket exhaustion**: *"`new HttpClient()` per request leaves sockets in `TIME_WAIT` and
  exhausts ephemeral ports; a static singleton fixes that but pins stale DNS.
  `IHttpClientFactory` pools handlers and rotates them, which solves both."*
- **Thread-pool starvation**: *"Blocking on async occupies a pool thread, and the pool injects
  replacements slowly — so throughput collapses while CPU sits idle. That combination is how you
  tell it apart from a CPU bottleneck."*
- **Cascading failure**: *"A slow dependency ties up threads and connections in its callers until
  they stop serving anything. Timeouts and circuit breakers turn slow failures into fast ones,
  which is what contains the blast radius."*
- **Cooperative cancellation**: *"A timeout just cancels a token — if you don't propagate it to the
  real I/O, the caller times out while the work carries on holding its connection."*
- **Dual write / outbox**: *"You can't atomically commit the database and publish to a broker. The
  outbox writes the message in the same transaction and a relay publishes it afterwards, so it goes
  out if and only if the change committed."*
- **At-least-once**: *"Every broker delivers at-least-once, so every consumer has to be idempotent —
  dedupe on a message id or make the operation naturally idempotent."*
- **JWT**: *"It's signed, not encrypted — the payload is readable by anyone holding it. The
  signature gives integrity, not confidentiality."*
- **Data Protection**: *"In a multi-instance deployment you have to persist the key ring to shared
  storage — otherwise each instance mints its own keys and users get logged out at random."*
- **Metric cardinality**: *"Every unique dimension combination is a separate time series, so ids go
  on trace spans, not metric tags."*
- **Observability**: *"Metrics detect, traces localize, logs explain — correlated by TraceId."*
- **Trimming**: *"Static analysis can't see a reflective lookup by name, so the type gets trimmed
  and it fails at runtime in the published build only. `IL2xxx` warnings predict exactly that, and
  source generators are the real fix."*
- **Aggregates**: *"Keep them small, protect true invariants together, reference other aggregates
  by id, and update one per transaction."*
- **Measure first**: *"Intuition about bottlenecks is reliably wrong. Baseline, profile, change one
  thing, re-measure — and for a data app, check the database before anything else."*

---

## 4. Answer protocol

**For any question**
1. **Restate** it, and clarify an assumption if it's ambiguous.
2. Answer the **direct question first** — one or two sentences.
3. *Then* add depth: the mechanism, a trade-off, or a trap.
4. Don't bluff internals. "I'd reason about it as…" plus stated assumptions beats a confident
   wrong answer.

**For "X vs Y"**
> one-sentence difference → when you'd use each → **a trap** in the one they'd expect you to pick.

**For "how would you design…"**
> clarify scale and constraints → the simplest thing that works → what you'd change at 10× →
> name the failure mode you're guarding against.

**For a coding question**
> state the approach and Big-O **before** typing → write it → verify with an example →
> mention the edge cases you'd test.

**For "how would you debug this in production?"**
> metrics to confirm and scope it → trace to localize the slow span → logs to explain →
> `dotnet-counters` for triage → the right deep tool (`-trace` for CPU, `-gcdump` for leaks,
> `-dump` for hangs).

---

## 5. If you have five minutes left

Say these out loud:

1. Three DI lifetimes + what a captive dependency is.
2. The middleware order.
3. N+1 and its two fixes.
4. `IHttpClientFactory` — socket exhaustion **and** stale DNS.
5. Sync-over-async → thread-pool starvation.
6. At-least-once → idempotency → the outbox.
7. JWT: signed not encrypted; validate signature/exp/iss/aud.
8. Liveness checks the app; readiness checks dependencies.
9. Metrics detect, traces localize, logs explain.
10. Measure before you optimize; check the database first.

---

→ Full self-quiz: [RapidFire.md](RapidFire.md) · Topic map: [README.md](README.md)
