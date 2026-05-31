# Rate Limiting

## Protecting your API from overload

Rate limiting caps how many requests a client (or the whole app) can make in a time window — protecting against abuse, accidental floods, and resource exhaustion, and ensuring fair use. ASP.NET Core has **built-in rate limiting** middleware (`Microsoft.AspNetCore.RateLimiting`, .NET 7+) with several algorithms.

```csharp
builder.Services.AddRateLimiter(options => {
    options.AddFixedWindowLimiter("fixed", o => {
        o.PermitLimit = 100;                       // 100 requests
        o.Window = TimeSpan.FromMinutes(1);         // per minute
        o.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        o.QueueLimit = 10;                          // queue up to 10 when over limit
    });
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});

var app = builder.Build();
app.UseRateLimiter();                                // add the middleware
app.MapGet("/api/data", () => ...).RequireRateLimiting("fixed");
app.Run();
```

When the limit is exceeded, requests are queued (up to `QueueLimit`) or rejected with **429 Too Many Requests** ([12-ProblemDetails.md](12-ProblemDetails.md)).

---

## The four algorithms

| Algorithm | Behavior | Best for |
|---|---|---|
| **Fixed window** | N requests per fixed time window (resets at boundary) | simple caps; can burst at window edges |
| **Sliding window** | N per rolling window (smooths the boundary burst) | fairer than fixed window |
| **Token bucket** | tokens refill at a rate; each request spends one; bursts allowed up to bucket size | bursty traffic with a sustained rate |
| **Concurrency** | N requests *in flight* at once (not per time) | protecting limited resources (DB connections, CPU) |

```csharp
options.AddSlidingWindowLimiter("sliding", o => {
    o.PermitLimit = 100; o.Window = TimeSpan.FromMinutes(1); o.SegmentsPerWindow = 6;
});
options.AddTokenBucketLimiter("token", o => {
    o.TokenLimit = 100;                              // bucket size (max burst)
    o.TokensPerPeriod = 10; o.ReplenishmentPeriod = TimeSpan.FromSeconds(1);   // 10/sec sustained
});
options.AddConcurrencyLimiter("concurrency", o => { o.PermitLimit = 20; o.QueueLimit = 50; });
```

- **Fixed window** is simplest but allows a double burst at the boundary (e.g., 100 at 0:59 + 100 at 1:00).
- **Sliding window** mitigates that by tracking sub-segments.
- **Token bucket** is ideal for "X per second sustained, allow short bursts."
- **Concurrency** limits *simultaneous* requests, not a rate — protects a scarce resource directly.

---

## Per-client partitioning

A single global limit isn't enough — you usually want a limit **per client** (per user, API key, or IP). Partition the limiter by a key extracted from the request:

```csharp
builder.Services.AddRateLimiter(options => {
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(ctx => {
        // partition by authenticated user, else by IP
        var key = ctx.User.Identity?.Name
                  ?? ctx.Connection.RemoteIpAddress?.ToString()
                  ?? "anonymous";
        return RateLimitPartition.GetFixedWindowLimiter(key, _ => new() {
            PermitLimit = 100, Window = TimeSpan.FromMinutes(1)
        });
    });
});
```

`PartitionedRateLimiter` applies a separate limiter per partition key — so each user/IP/API-key gets its own quota. Choosing the partition key is the crux: per-user (authenticated APIs), per-API-key (public APIs), or per-IP (anonymous) — IP is weakest (shared NATs, spoofing) but sometimes the only option.

---

## Applying limits & policies

```csharp
// Named policies applied per endpoint/group
app.MapGet("/cheap", () => ...).RequireRateLimiting("token");
app.MapGet("/expensive", () => ...).RequireRateLimiting("concurrency");
app.MapGroup("/api").RequireRateLimiting("fixed");

// Exempt an endpoint from a global limiter
app.MapGet("/health", () => ...).DisableRateLimiting();
```

Apply different policies to different endpoints (a cheap read vs an expensive report), or a global limiter to everything with per-endpoint overrides. MVC uses `[EnableRateLimiting("policy")]`/`[DisableRateLimiting]` attributes.

---

## Telling clients what happened (429 + headers)

```csharp
options.OnRejected = async (context, ct) => {
    context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
    if (context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
        context.HttpContext.Response.Headers.RetryAfter = ((int)retryAfter.TotalSeconds).ToString();
    await context.HttpContext.Response.WriteAsync("Rate limit exceeded. Try again later.", ct);
};
```

A good rate limiter tells clients **when to retry** via the `Retry-After` header so well-behaved clients back off (and Polly's retry can honor it — [Ch11](../11-Resilience/README.md)). Returning a clear 429 (ideally ProblemDetails) lets clients distinguish throttling from other errors.

---

## Distributed rate limiting

The built-in limiter is **per-instance** (in-memory). Across a multi-instance deployment (load-balanced, Kubernetes), each instance counts independently — so a "100/min" limit becomes "100/min × N instances." For a true global limit you need **distributed** state (e.g., Redis-backed counters via a library, or an API gateway / reverse proxy that rate-limits centrally). Decide whether per-instance limits suffice (often fine for basic protection) or you need a distributed/gateway-level limiter for strict global quotas.

---

## Rate limiting vs other protections

| Concern | Tool |
|---|---|
| Too many requests | **Rate limiting** (this file) |
| Slow/unreliable downstreams | **Resilience** (retry, circuit breaker — [Ch11](../11-Resilience/README.md)) |
| Overload of a scarce resource | **Concurrency limiter** (here) or a `SemaphoreSlim` |
| Malicious traffic / DDoS | **Gateway/WAF/CDN** (upstream of the app) |

Rate limiting protects *your* app from *clients*; resilience protects your app from *its dependencies*. They're complementary. Heavy DDoS belongs upstream (CDN/WAF), not in app code.

---

## Common gotchas

### Global instead of per-client limits

A single shared limit lets one abuser starve everyone. Partition by user/API-key/IP with `PartitionedRateLimiter`.

### Per-instance limits assumed to be global

With multiple instances, in-memory limits multiply by instance count. Use distributed/gateway limiting for strict global quotas.

### Fixed-window boundary burst

Fixed window allows up to 2× the limit around the window boundary. Use sliding window or token bucket if that matters.

### No `Retry-After`

Without it, clients retry blindly and may worsen the overload. Emit `Retry-After` on 429.

### Forgetting `UseRateLimiter` or `RequireRateLimiting`

Registering policies isn't enough — add the middleware and apply policies to endpoints/groups, or set a global limiter.

### Rate-limiting health checks

Don't throttle `/health` (it'd cause false unhealthy signals). `DisableRateLimiting` on health/infra endpoints.

---

## Summary

- Built-in **rate limiting** (`AddRateLimiter` + `UseRateLimiter`, .NET 7+) caps request rates, returning **429** when exceeded — protecting against abuse and overload.
- Four algorithms: **fixed window** (simple, edge-burst), **sliding window** (smoother), **token bucket** (sustained rate + bursts), **concurrency** (simultaneous requests). Pick per endpoint cost.
- Use **`PartitionedRateLimiter`** to limit **per client** (user/API-key/IP) — a single global limit lets one client starve others.
- Emit **`Retry-After`** on 429 so clients back off; apply policies via `RequireRateLimiting`/`[EnableRateLimiting]`, exempt infra endpoints.
- Built-in limits are **per-instance** — use distributed/gateway limiting for global quotas across multiple instances. Rate limiting (protect from clients) complements **resilience** (protect from dependencies, [Ch11](../11-Resilience/README.md)).

→ Next: [15-OutputCaching.md](15-OutputCaching.md)
