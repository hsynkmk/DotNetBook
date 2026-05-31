# Health Checks (Observability Context)

## Telling the platform if you're healthy

Health checks expose endpoints reporting whether the app (and its dependencies) are functioning, so load balancers, Kubernetes, and monitoring can route around or restart unhealthy instances. They're part of observability — a coarse, always-on signal of instance health that complements the detailed pillars (logs/metrics/traces).

> Health checks are covered in depth in **[Chapter 04 §16](../04-AspNetCore/16-HealthChecks.md)** (liveness vs readiness, statuses, the deployment story). This file recaps them in the **observability** context and how they relate to the pillars.

```csharp
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"])
    .AddDbContextCheck<AppDbContext>("db", tags: ["ready"])
    .AddCheck<RedisHealthCheck>("redis", tags: ["ready"]);

app.MapHealthChecks("/health/live",  new() { Predicate = c => c.Tags.Contains("live") });
app.MapHealthChecks("/health/ready", new() { Predicate = c => c.Tags.Contains("ready") });
```

---

## Liveness vs readiness (the critical distinction, recap)

The one rule to remember ([Ch04 §16](../04-AspNetCore/16-HealthChecks.md)):

| Probe | "?" | On fail | Checks |
|---|---|---|---|
| **Liveness** | is the process alive (not deadlocked)? | **restart** the instance | **only the app** — *not* dependencies |
| **Readiness** | can it serve traffic now? | **stop routing** (don't restart) | the app **plus** dependencies (DB, cache) |

**Liveness must check only the app** — if it checks the database, a DB blip fails liveness → the orchestrator **restarts every instance** (a restart storm that worsens the outage). **Readiness checks dependencies** — a DB outage fails readiness → traffic is routed away (but instances aren't killed) until it recovers. Conflating them is the classic, damaging mistake.

---

## Health checks as an observability signal

Health checks fit the observability picture as a **coarse instance-level signal**, distinct from the fine-grained pillars:

```
Health checks → "is this instance up / ready?"        (binary, per-instance, for the orchestrator)
Metrics       → "how is the fleet performing?"         (aggregate trends, dashboards, alerts)
Traces        → "where did this request slow/fail?"    (per-request, cross-service)
Logs          → "what exactly happened?"               (detailed events)
```

- Health checks drive **infrastructure decisions** (route/restart) and feed **uptime monitoring**.
- They're not a substitute for metrics/traces/logs (which explain *why* and show *trends*) — a passing health check doesn't mean the app is performing well, just that it's up/ready.
- Health-check results can be **scraped as metrics** and dashboards built on them (fleet readiness over time), tying them into the observability backend.

Use health checks for orchestration and uptime; use the pillars to understand behavior and diagnose problems.

---

## Statuses and custom checks (recap)

```csharp
public class RedisHealthCheck(IConnectionMultiplexer redis) : IHealthCheck {
    public async Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext ctx, CancellationToken ct) {
        try { await redis.GetDatabase().PingAsync(); return HealthCheckResult.Healthy(); }
        catch (Exception ex) { return HealthCheckResult.Unhealthy("Redis unreachable", ex); }
    }
}
```

Three statuses: **Healthy**, **Degraded** (impaired but serving — returns 200, signals a problem), **Unhealthy** (503). Use ready-made `AspNetCore.HealthChecks.*` packages (DB/Redis/RabbitMQ/URLs) or custom `IHealthCheck`s. **Keep checks fast and lightweight** (they're polled frequently — a slow check delays probe responses and adds load). Full details: [Ch04 §16](../04-AspNetCore/16-HealthChecks.md).

---

## Health checks + graceful shutdown (zero-downtime)

The observability/operations payoff: wiring **readiness** to **graceful shutdown** ([Ch08 §08](../08-BackgroundProcessing/08-ReliabilityAndScale.md)) enables zero-downtime deploys:

```
On SIGTERM (deploy/scale-down):
  fail readiness → orchestrator stops routing NEW requests to this instance
  → in-flight requests drain (within the shutdown timeout)
  → instance exits cleanly → no dropped requests
```

Combined with a **startup probe** (give a slow-starting app time before liveness kicks in, avoiding crash loops) and rolling updates, health checks are the contract that makes deploys, autoscaling, and self-healing work — the operational backbone that keeps the *system* available as individual instances churn ([Ch11 §08](../11-Resilience/08-Patterns.md)).

---

## Securing health endpoints

Detailed health output can reveal infrastructure (which DBs/caches/services exist):
- **Public**: a bare 200/503 (liveness/readiness) is fine for the orchestrator.
- **Detailed** (per-check status, a custom `ResponseWriter`): expose only **internally** or behind auth — don't leak your dependency map publicly ([Ch04 §16](../04-AspNetCore/16-HealthChecks.md)).

---

## Common gotchas

### Liveness checking dependencies → restart storm

The cardinal mistake: a DB/cache blip fails liveness → the orchestrator restarts all instances, amplifying the outage. **Liveness checks only the app**; readiness checks dependencies.

### Treating health checks as full observability

A passing health check means "up/ready," not "performing well." Use metrics/traces/logs to understand behavior; health checks are a coarse binary signal.

### Slow/heavy checks

Frequently-polled checks that run expensive queries slow probes and add load. Keep them light (a ping/`SELECT 1`) with timeouts.

### No readiness drain on shutdown

Without readiness failing on SIGTERM, the orchestrator keeps routing to a terminating instance → dropped requests. Wire readiness to graceful shutdown.

### Exposing detailed health publicly

Per-check detail leaks infrastructure. Public = bare status; detailed = internal/authenticated.

---

## Summary

- **Health checks** expose instance health for orchestrators (route around / restart) — a coarse, always-on observability signal distinct from the detailed pillars.
- **Liveness** (is the process alive? → restart) checks **only the app**; **readiness** (can it serve? → stop routing) checks **dependencies** — conflating them causes **restart storms** (the critical rule from [Ch04 §16](../04-AspNetCore/16-HealthChecks.md)).
- They drive **infrastructure decisions and uptime monitoring**, not behavior understanding — use metrics/traces/logs for *why*; a passing check ≠ good performance. Results can be scraped as fleet-readiness metrics.
- Wire **readiness to graceful shutdown** (+ a **startup probe**) for **zero-downtime deploys** and crash-loop avoidance; keep checks fast; secure detailed endpoints. Full coverage: [Chapter 04 §16](../04-AspNetCore/16-HealthChecks.md).

→ Next: [Questions.md](Questions.md)
