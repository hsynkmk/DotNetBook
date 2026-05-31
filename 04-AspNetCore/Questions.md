# Chapter 04 — ASP.NET Core — Q & A

---

### Q1. What is `WebApplication.CreateBuilder`?

A Generic Host specialized for HTTP. It gives DI, configuration, and logging (Chapter 03) plus Kestrel and the middleware pipeline. Two phases: register services on `builder.Services`, then compose the pipeline (`Use*`) and endpoints (`Map*`) on `app`; `app.Run()` starts the server.

---

### Q2. What is Kestrel?

The default cross-platform, high-performance HTTP server (built on sockets + `System.IO.Pipelines`), handling HTTP/1.1, HTTP/2, HTTP/3, and TLS. Often runs behind a reverse proxy (nginx/YARP/cloud LB) in production but can be edge-facing; your app code is the same either way.

---

### Q3. Minimal APIs vs MVC vs Razor Pages?

**Minimal APIs** — endpoints as lambdas, low ceremony, AOT-friendly (the default for new APIs). **MVC** — controller classes with a rich filter pipeline, for large apps and server-rendered views. **Razor Pages** — page-per-URL server-rendered UI. They share routing, binding, validation, and DI.

---

### Q4. How does Minimal API parameter binding infer sources?

Route if the name matches a route parameter; query for simple types not in the route; body (JSON) for complex types (one per request); DI for registered services; special types (`HttpContext`, `CancellationToken`, `ClaimsPrincipal`) auto-injected. Override with `[FromRoute]`/`[FromQuery]`/`[FromBody]`/etc.

---

### Q5. `Results` vs `TypedResults`?

`Results.Ok(x)` returns `IResult`; `TypedResults.Ok(x)` returns the concrete `Ok<T>` — better for unit testing handlers and for OpenAPI to infer response types. Use `Results<Ok<T>, NotFound>` unions to declare multiple possible results.

---

### Q6. Why do Minimal APIs have first-class Native AOT support but MVC doesn't?

Minimal APIs use the **Request Delegate Generator** (a source generator) to create binding/handler glue at compile time, avoiding runtime reflection. MVC's pipeline is reflection-based. So for AOT/cloud-native APIs, Minimal APIs + STJ source-gen are recommended.

---

### Q7. What does `[ApiController]` add?

Automatic 400 ProblemDetails on invalid `ModelState` (before the action runs), binding-source inference, ProblemDetails for error status codes, and required attribute routing. Always use it on API controllers — it removes boilerplate.

---

### Q8. How does Razor Pages routing work?

Convention-based on the file path under `Pages/` plus the `@page` directive (`Pages/Products/Edit.cshtml` → `/Products/Edit`; `@page "{id:int}"` adds parameters). The folder structure is the URL structure — no attribute routing.

---

### Q9. What is the Post-Redirect-Get pattern and why use it?

After a successful POST, **redirect** (`RedirectToPage`) instead of rendering, so a browser refresh doesn't re-submit the form (avoiding duplicate operations) and gives clean URLs. A fundamental web-form practice Razor Pages encourages.

---

### Q10. How does a middleware work, and why does order matter?

Each middleware gets `HttpContext` + a `next` delegate; it runs code before `await next()` (inbound) and after (outbound), or short-circuits by not calling next. It's a nested decorator chain. Order determines correctness — e.g., routing before auth (auth needs endpoint metadata), exception handler outermost, static files early.

---

### Q11. The middleware lifetime rule for class-based middleware?

Convention-based middleware is effectively a **singleton** — constructor dependencies are resolved once (singleton-scoped). To use **scoped** services, inject them into `InvokeAsync`'s parameters (resolved per request), not the constructor. Or use `IMiddleware` (DI-activated per request).

---

### Q12. `Use` vs `Run` vs `Map`?

`Use` adds middleware that (usually) calls `next`; `Run` is terminal (never calls next); `Map`/`MapWhen` branch the pipeline on a path/predicate. The endpoint middleware is effectively a `Run`.

---

### Q13. Are route constraints validation?

No. `{id:int}` filters **matching** (a non-int yields 404, not 400) — it's for routing, not business rules. Validate real constraints (range, existence) in the handler/model.

---

### Q14. Why generate URLs instead of hardcoding them?

Hardcoded URLs break when routes change. Use `LinkGenerator`/named routes (Minimal), `Url.Action`/`CreatedAtAction` (MVC), or `asp-page`/`asp-route-*` (Razor) so links survive route refactors.

---

### Q15. Why bind to DTOs instead of domain entities?

Binding/validating an entity lets clients set fields they shouldn't (over-posting / mass assignment, e.g., `IsAdmin`), and couples the API to the persistence model. Bind to a DTO with only allowed fields, then map to the entity.

---

### Q16. How does validation differ between MVC and Minimal APIs?

MVC `[ApiController]` runs DataAnnotations automatically and returns 400 ValidationProblemDetails. Minimal APIs historically didn't — add an endpoint filter, use `.AddValidation()` (.NET 10), or FluentValidation. Both should return RFC 7807 ValidationProblemDetails.

---

### Q17. When use FluentValidation over DataAnnotations?

For complex rules attributes can't express: async checks (e.g., "category must exist" via DB), conditional rules (`When`), collection rules (`RuleForEach`), and rich messages — in a separate, unit-testable, DI-injectable validator class.

---

### Q18. MVC filter types and order?

Authorization → resource → action → exception → result filters. Action filters (before/after the action) are most common; exception filters handle action exceptions; result filters shape the response. Apply globally, per-controller, or per-action; use `[ServiceFilter]`/`[TypeFilter]` for DI.

---

### Q19. Exception filter vs exception-handling middleware?

Exception filters catch only **MVC action** exceptions. **`UseExceptionHandler`** (middleware) catches everything (middleware, binding, endpoints) — use it as the global error boundary (→ ProblemDetails). Use exception filters only for MVC-specific exception→result mapping.

---

### Q20. Built-in OpenAPI (.NET 9+) vs Swashbuckle?

Built-in `AddOpenApi`/`MapOpenApi` generates the **document** (AOT-friendlier, no third-party dep) but **no UI** — add a separate UI (Scalar/Swashbuckle UI). Swashbuckle/NSwag are mature, bundle UI, and (NSwag) generate clients. Enrich the doc with `Produces<T>`/`TypedResults` so it matches reality.

---

### Q21. Where should `UseStaticFiles` go, and what's the security note?

Early in the pipeline (before routing/auth) so public assets skip that overhead and short-circuit. Security: the web root (`wwwroot`) is **public** (no secrets there), path traversal is blocked, and auth is skipped for static files by default (protect specific files via an authorized endpoint).

---

### Q22. What is Problem Details and how do you enable it?

RFC 7807/9457 — a standard machine-readable error body (`type`, `title`, `status`, `detail`, `instance`) as `application/problem+json`. Enable with `AddProblemDetails()` + `UseExceptionHandler()` (unhandled exceptions) + `UseStatusCodePages()` (bare status codes). `ValidationProblemDetails` adds an `errors` map.

---

### Q23. How should error detail differ between dev and production?

Dev: detailed (`UseDeveloperExceptionPage`, stack traces) for debugging. Production: generic ProblemDetails — never leak stack traces/exception types/internal messages (information disclosure). Expose a `traceId` to correlate the client's error with server logs.

---

### Q24. Do Minimal APIs do content negotiation?

No — they serialize to JSON via System.Text.Json (return other formats explicitly with `Results.Json`/`Text`/`Bytes`/`File`). MVC supports content negotiation via formatters; only JSON is registered by default (add `AddXmlSerializerFormatters()` for XML). Set `ReturnHttpNotAcceptable` for strict 406 behavior.

---

### Q25. The four rate-limiting algorithms?

**Fixed window** (N per fixed window; edge-burst), **sliding window** (smoother), **token bucket** (sustained rate + bursts up to bucket size), **concurrency** (N simultaneous in-flight, not a rate). Pick per endpoint cost; partition per client (user/API-key/IP) with `PartitionedRateLimiter`.

---

### Q26. Why is per-instance rate limiting a caveat?

The built-in limiter is in-memory, so with N instances a "100/min" limit becomes "100/min × N." For a strict global quota, use distributed (Redis-backed) limiting or a gateway. Per-instance limits are often fine for basic protection.

---

### Q27. Output caching vs `IMemoryCache`/response caching?

**Output caching** (.NET 7+) caches full **responses** server-side, server-controlled, with **invalidation** (tags) — the modern response cache. **`IMemoryCache`/`IDistributedCache`** cache arbitrary **data** you choose inside logic ([Ch06](../06-DataAndCaching/README.md)). Old **response caching** relied on client/proxy honoring headers.

---

### Q28. What's the cardinal output-caching bug?

Wrong/missing **vary-by**: if the response depends on query/route/header/user but you don't vary by it, everyone gets the first cached entry — serving wrong content (and leaking personalized data if cached without per-user keys). Vary by exactly what changes the response; tag + `EvictByTagAsync` on writes for freshness.

---

### Q29. Liveness vs readiness health checks — why must they differ?

**Liveness** ("is the process alive?" → restart on fail) must check **only the app** — checking dependencies means a DB blip restarts every pod (a restart storm). **Readiness** ("can I serve traffic?" → stop routing on fail) checks **dependencies** (DB, cache). Expose them separately via tags.

---

### Q30. How do health checks enable zero-downtime deploys?

Wire **readiness** to graceful shutdown: on SIGTERM, fail readiness → the orchestrator stops routing new requests → in-flight requests drain → the app exits. Combined with a startup probe (for slow starts) and rolling updates, this gives zero-downtime deploys.

---

→ Next: [Coding.md](Coding.md)
