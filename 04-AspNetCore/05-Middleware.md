# Middleware

## The request pipeline

Middleware are the components that process every HTTP request in sequence — the **request pipeline**. Each middleware can inspect/modify the request, short-circuit (return early), or call the next middleware, then act on the response on the way back out. It's a **decorator chain** ([Ch03 §06](../03-HostingAndDI/06-Decorate.md)) over the request: nested, with each layer wrapping the rest.

```
Request  →  [Exception handler] → [HTTPS] → [Static files] → [Routing] → [Auth] → [Endpoint]
                  │                                                                    │
                  └──────────────── Response flows back out (in reverse) ─────────────┘
```

```csharp
var app = builder.Build();

app.UseExceptionHandler("/error");   // 1. catch downstream exceptions
app.UseHttpsRedirection();            // 2. http → https
app.UseStaticFiles();                 // 3. serve wwwroot files (short-circuits if matched)
app.UseRouting();                     // 4. match the endpoint
app.UseAuthentication();              // 5. who are you?
app.UseAuthorization();               // 6. are you allowed?
app.MapControllers();                 // 7. terminal: run the endpoint

app.Run();
```

---

## How a middleware works

Each middleware receives the `HttpContext` and a `next` delegate. It can do work **before** calling `next` (inbound), **after** (outbound), or **instead** of `next` (short-circuit):

```csharp
app.Use(async (context, next) => {
    // BEFORE: runs on the way in
    var sw = Stopwatch.StartNew();

    await next(context);   // call the rest of the pipeline (the inner middleware + endpoint)

    // AFTER: runs on the way out (response is being formed)
    sw.Stop();
    context.Response.Headers["X-Response-Time"] = $"{sw.ElapsedMilliseconds}ms";
});
```

Because each middleware *awaits* the next, the pipeline is nested: the first middleware's "after" code runs **last**. Calling `next` continues; **not** calling it short-circuits (the response is whatever this middleware set).

```csharp
// Short-circuiting middleware (e.g., a maintenance gate)
app.Use(async (context, next) => {
    if (MaintenanceMode) {
        context.Response.StatusCode = 503;
        await context.Response.WriteAsync("Down for maintenance");
        return;                  // do NOT call next → pipeline stops here
    }
    await next(context);
});
```

---

## Order is everything

Middleware runs in the **order you add it**, and order determines correctness:

```csharp
// ✓ correct order
app.UseRouting();          // must run before auth (auth needs the matched endpoint's metadata)
app.UseAuthentication();   // must run before authorization
app.UseAuthorization();    // must run before the endpoint
app.MapControllers();

// ✗ wrong: authorization before authentication → no identity to authorize
// ✗ wrong: static files after routing → unnecessary work
```

Canonical ordering (typical web API):
```
ExceptionHandler / DeveloperExceptionPage   (outermost — catches everything below)
  → HSTS / HttpsRedirection
  → StaticFiles
  → Routing
  → CORS
  → Authentication
  → Authorization
  → RateLimiting / OutputCaching
  → Endpoints (terminal)
```

Getting this wrong is the most common ASP.NET Core bug — e.g., exception handling not outermost (so it misses errors), or auth after the endpoint (so it never protects it).

---

## `Use`, `Run`, `Map`

Three ways to add to the pipeline:

```csharp
// Use — middleware that (usually) calls next
app.Use(async (ctx, next) => { /* ... */ await next(ctx); });

// Run — TERMINAL middleware (never calls next; ends the pipeline)
app.Run(async ctx => await ctx.Response.WriteAsync("end of line"));

// Map — branch the pipeline on a path prefix
app.Map("/admin", admin => {
    admin.UseAuthorization();
    admin.Run(async ctx => await ctx.Response.WriteAsync("admin area"));
});

// MapWhen — branch on an arbitrary predicate
app.MapWhen(ctx => ctx.Request.Query.ContainsKey("debug"), branch => { ... });
```

`Use` is the common case; `Run` is terminal (the endpoint middleware is effectively a `Run`); `Map`/`MapWhen` create **branches** that handle matching requests separately.

---

## Class-based middleware

Inline lambdas are fine for small logic; for reusable, testable middleware, write a class:

```csharp
public class RequestLoggingMiddleware(RequestDelegate next, ILogger<RequestLoggingMiddleware> logger) {
    public async Task InvokeAsync(HttpContext context) {
        logger.LogInformation("→ {Method} {Path}", context.Request.Method, context.Request.Path);
        await next(context);                       // call the next middleware
        logger.LogInformation("← {StatusCode}", context.Response.StatusCode);
    }
}

// Register
app.UseMiddleware<RequestLoggingMiddleware>();
```

The convention: a constructor taking `RequestDelegate next` (+ any singleton dependencies) and an `InvokeAsync(HttpContext)` method. **Constructor dependencies are resolved once** (the middleware is effectively a singleton) — to use **scoped** services, inject them into `InvokeAsync` as parameters (the framework resolves them per request):

```csharp
public async Task InvokeAsync(HttpContext context, IScopedService scoped) {  // scoped resolved per request
    ...
}
```

This is the **key middleware lifetime rule**: constructor = singleton-scoped deps; `InvokeAsync` parameters = per-request (scoped) deps. Injecting a scoped service into the constructor is a captive-dependency bug ([Ch03 §03](../03-HostingAndDI/03-Lifetimes.md)).

---

## `IMiddleware` (factory-based) alternative

```csharp
public class FactoryMiddleware(IScopedDependency dep) : IMiddleware {   // can take scoped deps in ctor
    public async Task InvokeAsync(HttpContext context, RequestDelegate next) { ... await next(context); }
}
builder.Services.AddScoped<FactoryMiddleware>();   // register it
app.UseMiddleware<FactoryMiddleware>();
```

`IMiddleware` is **activated per request from DI** (so it *can* take scoped dependencies in its constructor) — useful when your middleware genuinely needs scoped state at construction. The convention-based form is more common; use `IMiddleware` when you want strong typing and DI-activation.

---

## Built-in middleware (the common ones)

| Middleware | Purpose | Section |
|---|---|---|
| `UseExceptionHandler` / `UseDeveloperExceptionPage` | catch & format errors | [12](12-ProblemDetails.md) |
| `UseHttpsRedirection` / `UseHsts` | force HTTPS | — |
| `UseStaticFiles` | serve `wwwroot` | [11](11-StaticFiles.md) |
| `UseRouting` / endpoints | match & run endpoints | [06](06-Routing.md) |
| `UseCors` | cross-origin requests | — |
| `UseAuthentication` / `UseAuthorization` | identity & access | [Ch09](../10-Identity/README.md) |
| `UseRateLimiter` | throttle | [14](14-RateLimiting.md) |
| `UseOutputCache` | cache responses | [15](15-OutputCaching.md) |
| `UseResponseCompression` | gzip/brotli | — |

Most apps compose a handful of these in the canonical order plus a little custom middleware.

---

## Common gotchas

### Wrong order

Auth before authentication, exception handler not outermost, static files after routing — the #1 ASP.NET Core bug. Follow the canonical order.

### Scoped service in a middleware constructor

Convention-based middleware is singleton-scoped; injecting a scoped service into its constructor is a captive dependency. Inject scoped deps into `InvokeAsync` parameters (or use `IMiddleware` + DI activation).

### Forgetting to call `next`

If your middleware doesn't call `next` (and isn't meant to short-circuit), the pipeline stops and the request hangs/returns empty. Call `await next(context)` unless deliberately short-circuiting.

### Writing to the response after it started

Once the response body has begun sending, you can't change status/headers (`Response.HasStarted` is true). Set headers/status **before** writing the body or calling `next` for downstream-affecting changes.

### Heavy work blocking the pipeline

Middleware runs on the request thread; blocking calls starve the thread pool. Keep middleware async and fast.

---

## Summary

- **Middleware** form the ordered **request pipeline** — a nested decorator chain; each component runs **before** and **after** `await next(context)`, or **short-circuits** by not calling `next`.
- **Order is critical** (exception handler outermost → HTTPS → static files → routing → auth → endpoints); wrong order is the most common bug.
- **`Use`** (calls next), **`Run`** (terminal), **`Map`/`MapWhen`** (branch the pipeline).
- **Class-based middleware** (`RequestDelegate next` ctor + `InvokeAsync`): constructor deps are **singleton-scoped**, inject **scoped** deps into `InvokeAsync` parameters (or use `IMiddleware` for DI activation).
- Compose built-in middleware (HTTPS, static files, auth, CORS, rate limiting, caching) in the canonical order plus thin custom middleware; keep it async and don't write the response twice.

→ Next: [06-Routing.md](06-Routing.md)
