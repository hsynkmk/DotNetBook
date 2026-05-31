# Health Checks

## Telling orchestrators if you're healthy

A health check is an endpoint that reports whether the app (and its dependencies) are functioning — so load balancers, Kubernetes, and monitoring can route traffic away from, or restart, an unhealthy instance. ASP.NET Core has built-in health checks (`Microsoft.Extensions.Diagnostics.HealthChecks`).

```csharp
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy())                  // the app is running
    .AddDbContextCheck<AppDbContext>("database")                          // can reach the DB
    .AddCheck<RedisHealthCheck>("redis");                                  // custom check

var app = builder.Build();
app.MapHealthChecks("/health");                                            // GET /health → 200/503
app.Run();
```

`GET /health` returns **200 Healthy** or **503 Unhealthy** (with a JSON/text body), which orchestrators poll.

---

## Liveness vs readiness — the crucial distinction

Kubernetes (and similar) use **two different probes**, and conflating them causes restart loops or dropped traffic:

| Probe | Question | If it fails | Should check |
|---|---|---|---|
| **Liveness** | "Is the process alive (not deadlocked)?" | **restart** the container | only the app itself — *not* dependencies |
| **Readiness** | "Can it serve traffic right now?" | **stop routing** traffic (don't restart) | the app **plus** dependencies (DB, cache, queue) |

```csharp
// Tag checks so you can expose them separately
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"])
    .AddDbContextCheck<AppDbContext>("db", tags: ["ready"])
    .AddCheck<QueueHealthCheck>("queue", tags: ["ready"]);

// Liveness: ONLY app-self checks (no dependencies)
app.MapHealthChecks("/health/live", new() { Predicate = c => c.Tags.Contains("live") });
// Readiness: app + dependencies
app.MapHealthChecks("/health/ready", new() { Predicate = c => c.Tags.Contains("ready") });
```

**Why it matters**: if **liveness** checks the database and the DB blips, Kubernetes **restarts every pod** (making things worse) instead of just pausing traffic. Liveness must check *only the app* (am I deadlocked?); **readiness** checks dependencies (can I serve a real request?). Getting this wrong is the classic health-check mistake — a brief DB outage becomes an app-wide restart storm.

---

## Health statuses

A check returns one of three statuses:

```csharp
public class RedisHealthCheck(IConnectionMultiplexer redis) : IHealthCheck {
    public async Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext ctx, CancellationToken ct) {
        try {
            await redis.GetDatabase().PingAsync();
            return HealthCheckResult.Healthy("Redis reachable");
        } catch (Exception ex) {
            return HealthCheckResult.Unhealthy("Redis unreachable", ex);
        }
    }
}
```

- **`Healthy`** — fully operational.
- **`Degraded`** — working but impaired (e.g., a cache is down so it's slower, but it can still serve). Returns **200** by default (still serve traffic) but signals a problem to monitoring.
- **`Unhealthy`** — not functioning. Returns **503**.

`Degraded` is useful: a non-critical dependency failing shouldn't take you out of rotation, but you want it visible. Map statuses to codes deliberately.

---

## Common checks

```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>()                       // EF Core DB connectivity
    .AddSqlServer(connectionString)                           // (AspNetCore.HealthChecks.* packages)
    .AddNpgSql(pgConnectionString)
    .AddRedis(redisConnectionString)
    .AddRabbitMQ()
    .AddUrlGroup(new Uri("https://upstream/health"), "upstream-api")   // downstream dependency
    .AddCheck<CustomBusinessCheck>("business-rule");
```

The community **`AspNetCore.HealthChecks.*`** packages provide ready-made checks for databases, caches, message brokers, cloud services, and HTTP endpoints. Custom checks (`IHealthCheck`) verify anything app-specific. Keep checks **fast and lightweight** — they're polled frequently; a slow check delays probe responses.

---

## Detailed responses & UI

```csharp
app.MapHealthChecks("/health", new HealthCheckOptions {
    ResponseWriter = async (ctx, report) => {
        ctx.Response.ContentType = "application/json";
        await ctx.Response.WriteAsJsonAsync(new {
            status = report.Status.ToString(),
            checks = report.Entries.Select(e => new { name = e.Key, status = e.Value.Status.ToString(), e.Value.Description })
        });
    }
});
```

By default the response is just the overall status; a custom `ResponseWriter` emits per-check detail (useful for dashboards). The **`AspNetCore.HealthChecks.UI`** package provides a polling dashboard across services. **Secure detailed health endpoints** — per-check detail can reveal infrastructure; expose detail only internally or behind auth, and a bare 200/503 publicly.

---

## Health checks in the deployment story

```yaml
# Kubernetes probes (conceptual)
livenessProbe:  { httpGet: { path: /health/live,  port: 8080 }, periodSeconds: 10 }
readinessProbe: { httpGet: { path: /health/ready, port: 8080 }, periodSeconds: 5 }
startupProbe:   { httpGet: { path: /health/live,  port: 8080 }, failureThreshold: 30 }
```

- **Startup probe** — gives a slow-starting app time to come up before liveness kicks in (avoids killing apps mid-startup).
- **Readiness gating** — combined with graceful shutdown ([Ch03 §01](../03-HostingAndDI/01-GenericHost.md)), readiness lets you drain traffic before a pod terminates: on SIGTERM, fail readiness → orchestrator stops routing → finish in-flight requests → exit. This enables **zero-downtime deploys**.

Health checks are the contract between your app and the orchestrator — they're how rolling updates, autoscaling, and self-healing work. (Deployment: [Ch19](../19-Deployment/README.md).)

---

## Common gotchas

### Liveness checking dependencies → restart storms

The cardinal mistake: a DB/cache blip fails liveness → orchestrator restarts all pods → outage amplified. **Liveness checks only the app**; readiness checks dependencies.

### Slow/heavy checks

Health checks are polled often; an expensive check (full query, large computation) slows probes and adds load. Keep them light (a ping/`SELECT 1`), with timeouts.

### Exposing detailed health publicly

Per-check detail leaks infrastructure (which DB/cache/services exist). Public endpoint → bare status; detailed endpoint → internal/authenticated.

### Not wiring readiness to graceful shutdown

Without readiness failing on shutdown, the orchestrator keeps routing to a terminating pod → dropped requests. Fail readiness on SIGTERM and drain.

### Treating Degraded as failure

Returning Unhealthy for a non-critical dependency takes you out of rotation unnecessarily. Use `Degraded` for impaired-but-serving states.

### No startup probe for slow starts

Liveness killing an app that's still warming up causes crash loops. Use a startup probe (or initial delay) for slow starters.

---

## Summary

- **Health checks** report app/dependency health so orchestrators route around or restart unhealthy instances (`AddHealthChecks` + `MapHealthChecks`, returning 200/503).
- **Liveness** (am I alive? → restart on fail) must check **only the app**; **readiness** (can I serve? → stop routing on fail) checks **dependencies** — conflating them causes restart storms. Tag checks and expose separate `/health/live` and `/health/ready`.
- Statuses: **Healthy** / **Degraded** (impaired but serving, 200) / **Unhealthy** (503). Use ready-made `AspNetCore.HealthChecks.*` packages or custom `IHealthCheck`s; keep them **fast**.
- **Secure** detailed health endpoints (they reveal infrastructure); expose bare status publicly.
- Wire **readiness to graceful shutdown** + a **startup probe** for zero-downtime deploys and crash-loop avoidance. The orchestration contract — see [Ch19](../19-Deployment/README.md).

→ Next: [Questions.md](Questions.md)
