# 04 — ASP.NET Core

## ⚡ 30-second answer

ASP.NET Core is the **Generic Host specialized for HTTP** ([03](03-Hosting-DI-Config.md)):
`WebApplication.CreateBuilder` gives you DI, configuration and logging, plus **Kestrel** and a
**middleware pipeline**. Two phases — register **services** on `builder.Services`, then compose
the **pipeline (`Use*`) and endpoints (`Map*`)** on `app`. Every request flows through an
**ordered chain of middleware** to an endpoint, and **that order is the single most common
source of bugs**. Three endpoint styles share routing, binding, validation and DI: **Minimal
APIs** (the default for new services — low ceremony, Native AOT-friendly), **MVC** (controllers
+ a rich filter pipeline), and **Razor Pages** (server-rendered forms). Each request gets its
own **DI scope**. Errors are handled once, at the boundary, as **ProblemDetails**.

---

## Core mechanics

### The two-phase shape

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDbContext<AppDb>(…);          // phase 1: register services
builder.Services.AddProblemDetails();

var app = builder.Build();
app.UseExceptionHandler();                        // phase 2: compose the pipeline
app.UseStaticFiles();
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
app.MapGet("/products/{id:int}", GetProduct);     // …and the endpoints
app.Run();                                        // start Kestrel
```

**Kestrel** is the cross-platform HTTP server (sockets + `System.IO.Pipelines`, HTTP/1.1, /2,
/3, TLS). It can be edge-facing, but usually sits behind a reverse proxy (nginx, YARP, a cloud
load balancer) in production. Your code is identical either way.

### Canonical middleware order

This ordering is worth memorizing verbatim — interviewers ask for it directly:

```text
ExceptionHandler   ← outermost, so it catches everything below
HSTS / HttpsRedirection
StaticFiles        ← early: public assets skip routing + auth
Routing            ← matches the endpoint, attaches its metadata
CORS
Authentication     ← who are you?      (needs endpoint metadata → after routing)
Authorization      ← are you allowed?  (needs the identity → after authentication)
RateLimiter / OutputCache
Endpoints          ← terminal
```

Each middleware runs code **before** `await next(context)` (inbound) and **after** it
(outbound) — a nested decorator chain. Not calling `next` **short-circuits** the pipeline.

```csharp
app.Use(async (ctx, next) =>
{
    var sw = Stopwatch.StartNew();      // inbound
    await next(ctx);                    // the rest of the pipeline
    log.LogInformation("{Path} took {Ms}ms", ctx.Request.Path, sw.ElapsedMilliseconds);
});
```

`Use` calls next; **`Run`** is terminal; **`Map`/`MapWhen`** branch the pipeline on a path or
predicate.

### Minimal APIs

Handlers are lambdas; parameters bind from route, query, body, headers and **DI**.

```csharp
var group = app.MapGroup("/products").WithTags("Products").RequireAuthorization();

group.MapGet("/{id:int}", async Task<Results<Ok<ProductDto>, NotFound>> (
        int id, AppDb db, CancellationToken ct) =>
    await db.Products.FindAsync([id], ct) is { } p
        ? TypedResults.Ok(p.ToDto())
        : TypedResults.NotFound())
     .WithName("GetProduct");
```

- **`TypedResults`** over `Results`: concrete types make handlers unit-testable and let OpenAPI
  infer the response shape. `Results<T1,T2>` unions declare every possible outcome.
- **`MapGroup`** shares a prefix, auth, filters and tags across endpoints.
- **Endpoint filters** (`IEndpointFilter`) wrap handlers for validation/logging.
- **Native AOT**: the **Request Delegate Generator** builds the binding glue at compile time, so
  no runtime reflection — this is why Minimal APIs are AOT-ready and MVC isn't.

### MVC

```csharp
[ApiController]                 // ← always, on API controllers
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet("{id:int}")]
    public async Task<ActionResult<ProductDto>> Get(int id, CancellationToken ct) => …;
}
```

**`[ApiController]`** buys you: automatic **400 ValidationProblemDetails** on invalid
`ModelState` *before the action runs*, binding-source inference, ProblemDetails error bodies, and
required attribute routing.

### Model binding

Source inference: **route** if the name matches a route parameter → **query** for remaining
simple types → **body** (JSON, one per request) for complex types → **DI** for registered
services. Special types (`HttpContext`, `ClaimsPrincipal`, **`CancellationToken`**) are injected
automatically — always forward that token to async calls. Override with `[FromRoute]`,
`[FromQuery]`, `[FromBody]`, `[FromHeader]`, `[FromServices]`; bundle many parameters with
`[AsParameters]`.

Simple types convert via **`IParsable<T>`/`TryParse`** — including your own types. Minimal APIs
can take full control with a static **`BindAsync`**.

### Validation

Three layers, and interviewers like hearing all three: **request validation** (format/UX →
400), **domain invariants** (business rules), **database constraints** (the backstop).

```csharp
public record CreateProduct([Required, MaxLength(100)] string Name,
                            [Range(0.01, 10_000)] decimal Price);
```

MVC `[ApiController]` runs DataAnnotations automatically. Minimal APIs need an endpoint filter,
`.AddValidation()` (.NET 10), or **FluentValidation** — which is what you reach for when rules
get complex (async DB checks, `When` conditionals, `RuleForEach`).

### Errors as ProblemDetails

**RFC 7807/9457**: `type`, `title`, `status`, `detail`, `instance`, served as
`application/problem+json`. `ValidationProblemDetails` adds an `errors` map.

```csharp
builder.Services.AddProblemDetails();
builder.Services.AddExceptionHandler<NotFoundExceptionHandler>();   // IExceptionHandler, .NET 8+
app.UseExceptionHandler();      // unhandled exceptions → problem+json
app.UseStatusCodePages();       // bare 404/403 → problem+json
```

Handle errors **once at the global boundary**, not per handler. Map exception types to status
codes centrally with `IExceptionHandler`. Dev gets detail; **production gets a generic body plus
a `traceId`** — never a stack trace.

### Rate limiting and output caching

```csharp
builder.Services.AddRateLimiter(o => o.AddTokenBucketLimiter("api", …));
builder.Services.AddOutputCache(o => o.AddPolicy("short", b => b.Expire(TimeSpan.FromSeconds(30))));

app.MapGet("/report", …).RequireRateLimiting("api").CacheOutput("short").Tag("reports");
await cache.EvictByTagAsync("reports", ct);      // invalidate on write
```

Rate limiting protects you **from your clients**; resilience ([11](11-Resilience.md)) protects
you **from your dependencies**. Output caching caches whole **responses** server-side (with
invalidation); `IMemoryCache`/`IDistributedCache` cache **data** inside your logic
([06](06-DataAccess-Caching.md)).

### Health checks

```csharp
app.MapHealthChecks("/health/live",  new() { Predicate = _ => false });          // app only
app.MapHealthChecks("/health/ready", new() { Predicate = c => c.Tags.Contains("ready") });
```

**Liveness** = "is the process alive?" → failure means **restart**, so it must check *only the
app*. **Readiness** = "can I serve?" → failure means **stop routing**, so it checks dependencies.
Wire readiness to graceful shutdown for zero-downtime deploys
([18–19](18-19-Aspire-Deployment.md)).

---

## Comparison tables

| | Minimal APIs | MVC | Razor Pages |
|---|---|---|---|
| Unit of code | route-handler lambda | controller + actions | `.cshtml` + `PageModel` |
| Routing | `Map*` + `MapGroup` | attribute routes | folder structure + `@page` |
| Cross-cutting | endpoint filters | **rich filter pipeline** | page filters |
| Native AOT | **yes** (RDG source-gen) | no (reflection) | no |
| Content negotiation | no — **JSON only** | yes (formatters) | n/a (HTML) |
| Best for | new APIs, microservices, AOT | large apps, server-rendered views | form-driven server UI |

| Middleware | Filters / endpoint filters | Decorators |
|---|---|---|
| sees raw HTTP, **every** request | sees bound arguments + endpoint metadata | sees one service's calls |
| cross-cutting HTTP concerns | per-endpoint concerns | per-dependency concerns |
| auth, CORS, exceptions, logging | validation, auditing, result shaping | caching, retry around a client |

| Rate-limit algorithm | Behavior | Use for |
|---|---|---|
| **Fixed window** | N per fixed window | simple caps (edge-burst at boundaries) |
| **Sliding window** | rolling window | smoother than fixed |
| **Token bucket** | sustained rate + burst | APIs with bursty but bounded traffic |
| **Concurrency** | N simultaneous in-flight | expensive endpoints (not a *rate*) |

| Health probe | Question | Failure means | Checks |
|---|---|---|---|
| **Liveness** | is the process alive? | **restart me** | the app only |
| **Readiness** | can I serve traffic? | **stop routing to me** | app + dependencies |
| **Startup** | have I finished booting? | keep waiting | slow init |

| Status | Meaning | HTTP |
|---|---|---|
| `Healthy` | all good | 200 |
| `Degraded` | impaired but serving | 200 |
| `Unhealthy` | can't serve | 503 |

---

## 🪤 Traps & gotchas

- **Middleware in the wrong order** — the #1 ASP.NET Core bug. `UseAuthorization()` before
  `UseAuthentication()` means there's no identity to authorize. `UseRouting()` after auth means
  auth can't see the endpoint's `[Authorize]` metadata. `UseExceptionHandler()` anywhere but
  first means it can't catch what runs above it.
- **Scoped service in a class-based middleware constructor** — convention-based middleware is
  built **once** (effectively a singleton), so a `DbContext` injected into its constructor is
  captured for the app's lifetime.

  ```csharp
  public class MyMiddleware(RequestDelegate next, AppDb db)   // 🐛 captive DbContext
  {
      public async Task InvokeAsync(HttpContext ctx) { … }
  }
  // ✅ inject per-request services into InvokeAsync instead:
  public async Task InvokeAsync(HttpContext ctx, AppDb db) { … }
  ```

- **Writing to the response after `next()`** — once the response has started, headers are already
  on the wire and mutating them throws. Check `ctx.Response.HasStarted`.
- **Route constraints are not validation.** `{id:int}` decides whether the route *matches* — a
  non-integer gives **404, not 400**. Range and existence checks belong in the handler.
- **Binding to domain entities** — over-posting / mass assignment: a client sends
  `"isAdmin": true` and the binder happily sets it. Bind to a DTO with only the allowed fields.
- **Assuming Minimal APIs validate DataAnnotations** — historically they don't. `[Required]` on
  a Minimal API record is decoration unless you add a filter, `.AddValidation()` (.NET 10), or
  FluentValidation.
- **Two complex parameters from the body** — only one may bind from the body per request.
- **Forgetting `CancellationToken`** — the parameter is injected for free, but if you don't
  forward it to EF/HTTP calls, a client that disconnects still costs you the full query.
- **Leaking exception detail in production** — stack traces, exception types and inner messages
  are information disclosure. Generic ProblemDetails + a `traceId` for correlation.
- **Catching exceptions in every handler** instead of once at the boundary — repetitive, and
  inevitably inconsistent. `UseExceptionHandler` + `IExceptionHandler`.
- **Exception filters don't catch everything** — they see only **MVC action** exceptions, not
  middleware or model-binding failures. Middleware is the real global boundary.
- **Wrong or missing output-cache vary-by** — the cardinal caching bug. If the response depends
  on the user, the query string or a header and you don't vary by it, **everyone gets the first
  caller's response** — including their personalized data.
- **Caching authenticated responses** without a per-user key is a data leak, not a perf win.
- **Built-in rate limiting is per-instance** — in-memory, so "100/min" across 5 pods is really
  500/min. Use Redis-backed or gateway limiting for a true global quota.
- **429 without a `Retry-After` header** gives clients nothing to back off against.
- **Liveness that checks the database** — a 30-second DB blip fails liveness on every pod and the
  orchestrator restarts the entire fleet. Restart storm. Liveness checks the app, full stop.
- **Health endpoints exposed publicly with detail** — they enumerate your infrastructure. Bare
  status public, detail behind auth.
- **`UseStaticFiles` placed after auth** — every image and stylesheet now pays for routing and
  authorization. Put it early. And remember `wwwroot` is **public**: no secrets there, and
  static files skip auth by default.
- **Hardcoded URLs** break silently on route refactors. `LinkGenerator`, named routes,
  `CreatedAtAction`, `asp-page`.
- **Expecting content negotiation from Minimal APIs** — they're JSON-only; return other formats
  explicitly via `Results.Text`/`Bytes`/`File`.
- **Rendering after a successful POST** instead of redirecting — a browser refresh re-submits the
  form. **Post-Redirect-Get**.
- **Shipping the OpenAPI UI to production** un-gated — free documentation for an attacker.

---

## ❓ Likely questions

**Q: What does `WebApplication.CreateBuilder` give you?**
A: The Generic Host specialized for HTTP — DI, configuration and logging, plus Kestrel and the
middleware pipeline. You register services on the builder, compose pipeline and endpoints on the
app, then `app.Run()`.

**Q: Why does middleware order matter? Give the canonical order.**
A: Each middleware wraps the rest of the pipeline, so position determines what it can see and
change. Exception handler outermost → HSTS/HTTPS → static files → routing → CORS →
authentication → authorization → rate limiting/caching → endpoints. Routing must precede auth so
auth can read the endpoint's metadata; authentication must precede authorization.

**Q: How does a middleware actually work?**
A: It receives `HttpContext` and a `next` delegate, runs inbound code, awaits `next`, then runs
outbound code — a nested decorator chain. Not calling `next` short-circuits.

**Q: What's the lifetime rule for class-based middleware?**
A: Convention-based middleware is constructed once, so constructor dependencies are effectively
singletons. Inject **scoped** services (like `DbContext`) into `InvokeAsync`'s parameters, which
are resolved per request — or use `IMiddleware` for per-request DI activation.

**Q: Minimal APIs vs MVC vs Razor Pages — how do you choose?**
A: Minimal APIs for new APIs, microservices and anything AOT (lowest ceremony, source-generated
binding). MVC for large apps, server-rendered views, or where the rich filter pipeline earns its
keep. Razor Pages for form-driven server UI. They share routing, binding, validation and DI and
can coexist in one app.

**Q: Why do Minimal APIs support Native AOT when MVC doesn't?**
A: The Request Delegate Generator emits the binding and handler glue at compile time, so there's
no runtime reflection. MVC's pipeline is reflection-based, which AOT trimming can't preserve.

**Q: `Results` vs `TypedResults`?**
A: `Results.Ok(x)` returns `IResult`; `TypedResults.Ok(x)` returns the concrete `Ok<T>` — so
handlers are unit-testable without executing HTTP, and OpenAPI can infer the response type.
`Results<Ok<T>, NotFound>` declares the full set of outcomes.

**Q: What does `[ApiController]` add?**
A: Automatic 400 ValidationProblemDetails on invalid `ModelState` before the action runs,
binding-source inference, ProblemDetails for error status codes, and required attribute routing.

**Q: How does Minimal API parameter binding infer its source?**
A: Route if the name matches a route parameter, query for other simple types, body (JSON) for a
complex type, DI for registered services, and special types like `HttpContext` and
`CancellationToken` automatically. `[From*]` overrides it.

**Q: Are route constraints validation?**
A: No. They decide whether a route matches — a failed constraint is a 404, not a 400. Real
validation (ranges, existence, business rules) happens in the handler.

**Q: Why bind to DTOs rather than entities?**
A: Over-posting. Binding an entity lets a client set any bindable property — `IsAdmin`,
`Balance` — and couples your public contract to the persistence model. A DTO exposes only what
you meant to accept.

**Q: Middleware vs filter vs decorator — when do you use which?**
A: By what you need to see. Middleware sees raw HTTP on every request (auth, CORS, errors).
Filters/endpoint filters see bound arguments and endpoint metadata (validation, auditing).
Decorators wrap a single service (caching, retry).

**Q: Exception filter or exception-handling middleware?**
A: Middleware. `UseExceptionHandler` is the global boundary — it catches everything including
middleware and binding failures. Exception filters only see MVC action exceptions; use them for
MVC-specific exception-to-result mapping.

**Q: What is ProblemDetails and how do you enable it?**
A: RFC 7807/9457 — a standard machine-readable error body served as `application/problem+json`.
`AddProblemDetails()` plus `UseExceptionHandler()` for unhandled exceptions and
`UseStatusCodePages()` for bare status codes. `ValidationProblemDetails` adds an `errors` map.

**Q: How should errors differ between dev and production?**
A: Dev shows detail for debugging. Production returns a generic ProblemDetails with a `traceId`
and no stack trace, exception type or internal message — those are information disclosure.

**Q: Name the four rate-limiting algorithms.**
A: Fixed window (N per window, bursts at the boundary), sliding window (smoother), token bucket
(sustained rate plus a burst allowance), and concurrency (N in flight — a limit on
simultaneity, not rate). Partition per user/API-key/IP, or one noisy client starves everyone.

**Q: What's the catch with built-in rate limiting?**
A: It's in-memory and therefore per-instance — N replicas multiply the effective limit by N. A
strict global quota needs Redis-backed limiting or a gateway.

**Q: Output caching vs `IMemoryCache`?**
A: Output caching stores whole responses server-side, with tag-based invalidation — the modern
replacement for response caching (which merely asked clients and proxies to cooperate).
`IMemoryCache`/`IDistributedCache` cache arbitrary data inside your logic.

**Q: Liveness vs readiness — why must they be different?**
A: Liveness failure restarts the pod, so it must check only the app; if it checks the database, a
brief DB outage restarts your whole fleet. Readiness failure just stops traffic being routed, so
it's the one that checks dependencies.

**Q: How do health checks give you zero-downtime deploys?**
A: On SIGTERM, fail readiness first — the orchestrator stops sending new requests, in-flight
requests drain, then the process exits. Add a startup probe for slow boots and roll instances
one at a time.

---

## 🎓 Senior Extra

- **`IExceptionHandler` (.NET 8+)** is the clean seam for mapping exception types to status
  codes: registered handlers are tried in order, and the first to return `true` wins. Far better
  than a `switch` inside a lambda in `UseExceptionHandler`.
- **Endpoint metadata is the mechanism** behind most of ASP.NET Core's "magic". `UseRouting`
  selects the endpoint and attaches its metadata collection; every downstream middleware
  (`[Authorize]`, CORS, rate limiting, output caching) simply reads that collection. Understanding
  this explains the whole ordering rule in one sentence.
- **Kestrel limits matter under attack**: `MaxRequestBodySize`, `MaxConcurrentConnections`, and
  the request-headers/body timeouts are your first line against slowloris and oversized uploads.
- **`ConfigureHttpJsonOptions` (Minimal) vs `AddJsonOptions` (MVC)** configure *different*
  serializers — setting one and expecting the other to change is a classic afternoon lost.
- **System.Text.Json source generators** (`JsonSerializerContext`) remove reflection from
  serialization — required for AOT, and a measurable throughput win even without it.
- **`MapStaticAssets` (.NET 9+)** does build-time fingerprinting and pre-compression, so you get
  `immutable` caching for free and never serve a stale asset.
- **The filter pipeline runs inside the endpoint middleware**, which is why filters can't catch
  what happens above them — and why `UseExceptionHandler` outranks any exception filter.
- **Graceful shutdown** is `IHostApplicationLifetime.ApplicationStopping` plus
  `ShutdownTimeout`; readiness-fail-then-drain is the pattern that makes rolling deploys
  invisible to callers ([08](08-BackgroundProcessing.md)).
- **API versioning** (`Asp.Versioning`) is far cheaper to add on day one than on the day you need
  it; URL-segment versioning (`/v1/…`) is the clearest for public APIs, and each version gets its
  own OpenAPI document.
- **HTTP/3 and response compression** are one-liners in Kestrel but interact: never compress
  responses containing secrets alongside attacker-controlled input (BREACH).
- **Output caching + authentication** deserve a rule you can state: cache anonymous responses
  aggressively, personalized responses only with the user in the cache key, and never cache
  `Set-Cookie`.

→ Deeper: [`../04-AspNetCore/`](../04-AspNetCore/README.md)
