# Filters

## Cross-cutting logic around endpoints

Filters run code **before and after** an endpoint executes, for cross-cutting concerns — authorization, validation, logging, caching, exception handling, response shaping. MVC has a rich, typed **filter pipeline**; Minimal APIs have lighter **endpoint filters**. Both let you factor out repetitive logic that would otherwise clutter every handler.

---

## MVC filter pipeline

MVC runs several **filter types** in a defined order around an action:

```
Authorization filters   → is the caller allowed? (runs first; can short-circuit)
   ↓
Resource filters        → wrap model binding + action (caching, short-circuit early)
   ↓
[ model binding ]
   ↓
Action filters          → before/after the action method (most common)
   ↓
[ action executes ]
   ↓
Exception filters       → handle exceptions from the action
   ↓
Result filters          → before/after the result executes (response shaping)
```

```csharp
// An action filter (most common)
public class AuditFilter(ILogger<AuditFilter> log) : IActionFilter {
    public void OnActionExecuting(ActionExecutingContext ctx) =>
        log.LogInformation("→ {Action}", ctx.ActionDescriptor.DisplayName);
    public void OnActionExecuted(ActionExecutedContext ctx) =>
        log.LogInformation("← {Status}", ctx.HttpContext.Response.StatusCode);
}

// Async variant (preferred for I/O)
public class TimingFilter : IAsyncActionFilter {
    public async Task OnActionExecutionAsync(ActionExecutingContext ctx, ActionExecutionDelegate next) {
        var sw = Stopwatch.StartNew();
        var executed = await next();        // runs the action (and inner filters)
        sw.Stop();
        executed.HttpContext.Response.Headers["X-Time"] = $"{sw.ElapsedMilliseconds}ms";
    }
}
```

Each filter type has a sync (`IActionFilter`) and async (`IAsyncActionFilter`) form — use async for any I/O. The `next()` delegate continues the pipeline; not calling it short-circuits.

---

## Applying MVC filters & DI

Filters apply at three scopes, and can be DI-resolved:

```csharp
// Global (all actions)
builder.Services.AddControllers(o => o.Filters.Add<AuditFilter>());

// Controller or action level — but attributes can't easily take DI deps, so use:
[ServiceFilter(typeof(AuditFilter))]      // resolves AuditFilter from DI (register it!)
[TypeFilter(typeof(AuditFilter))]         // instantiates with DI for ctor args (no registration needed)
public class OrdersController : ControllerBase {
    [Authorize]                            // built-in authorization filter
    [HttpPost]
    public IActionResult Create(...) { }
}
builder.Services.AddScoped<AuditFilter>(); // for ServiceFilter
```

- **`[ServiceFilter]`** — resolves the filter from DI (register it; respects its lifetime).
- **`[TypeFilter]`** — instantiates the filter, injecting constructor args from DI (no registration needed).
- Plain attribute filters (`[Authorize]`, `[ValidateAntiForgeryToken]`) need no DI.

Order within a scope follows the `Order` property; across scopes: global → controller → action (for "executing"), reversed for "executed".

---

## Endpoint filters (Minimal APIs)

Minimal APIs use a single, simpler filter concept — `IEndpointFilter` — that wraps the handler:

```csharp
app.MapPost("/products", (CreateProduct dto, IProductSvc svc) => svc.Create(dto))
    .AddEndpointFilter(async (ctx, next) => {
        var sw = Stopwatch.StartNew();
        var result = await next(ctx);                 // run the handler (and inner filters)
        sw.Stop();
        ctx.HttpContext.Response.Headers["X-Time"] = $"{sw.ElapsedMilliseconds}ms";
        return result;
    });

// Reusable class filter
public class LoggingFilter(ILogger<LoggingFilter> log) : IEndpointFilter {
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext ctx, EndpointFilterDelegate next) {
        log.LogInformation("→ {Path}", ctx.HttpContext.Request.Path);
        var result = await next(ctx);
        return result;
    }
}
app.MapPost("/x", ...).AddEndpointFilter<LoggingFilter>();

// Apply to a whole group
app.MapGroup("/api").AddEndpointFilter<LoggingFilter>();
```

Endpoint filters run around the handler (before/after), can inspect bound arguments (`ctx.Arguments`/`GetArgument<T>`), short-circuit (return a result without calling `next`), and stack. They apply per-endpoint or per-group. (Validation via endpoint filters: [08-Validation.md](08-Validation.md).)

---

## Exception filters vs exception-handling middleware

MVC **exception filters** (`IExceptionFilter`) catch exceptions from actions:

```csharp
public class DomainExceptionFilter : IExceptionFilter {
    public void OnException(ExceptionContext ctx) {
        if (ctx.Exception is DomainException dex) {
            ctx.Result = new ObjectResult(new { error = dex.Message }) { StatusCode = 422 };
            ctx.ExceptionHandled = true;
        }
    }
}
```

But for **app-wide** error handling, prefer **exception-handling middleware** (`UseExceptionHandler` → ProblemDetails — [12-ProblemDetails.md](12-ProblemDetails.md)), which catches everything (middleware, filters, endpoints), not just MVC actions. Use exception filters only for MVC-specific exception-to-result mapping; use middleware as the global boundary (CSharpBook Ch17 §13).

---

## Filters vs middleware vs decorators

| Tool | Scope | Sees | Use |
|---|---|---|---|
| **Middleware** | whole pipeline (all requests) | `HttpContext` | global concerns (auth, HTTPS, logging all requests) — [05-Middleware.md](05-Middleware.md) |
| **MVC filters** | actions/controllers | model-bound args, action context, result | MVC cross-cutting (auth, validation, result shaping) |
| **Endpoint filters** | Minimal endpoints/groups | bound arguments, result | Minimal API cross-cutting |
| **DI decorators** | a specific service | the service's methods | service-level concerns (caching a repo) — [Ch03 §06](../03-HostingAndDI/06-Decorate.md) |

Choose by **what you need to see**: middleware sees raw HTTP (runs before model binding); filters/endpoint-filters see bound arguments and endpoint metadata (run after binding); decorators wrap a service. Don't put endpoint-specific logic in middleware (it runs for everything) or global logic in a filter (it only covers MVC/that endpoint).

---

## Common uses

- **Authorization** — `[Authorize]` (built-in authorization filter) / `.RequireAuthorization()` (endpoint).
- **Validation** — auto in MVC `[ApiController]`; endpoint filter / FluentValidation in Minimal APIs ([08-Validation.md](08-Validation.md)).
- **Logging / auditing / timing** — action/endpoint filters.
- **Caching** — prefer built-in **output caching** ([15-OutputCaching.md](15-OutputCaching.md)) over hand-rolled filters.
- **Response transformation / enveloping** — result filters / endpoint filters.
- **Anti-forgery** — `[ValidateAntiForgeryToken]` (MVC); automatic in Razor Pages.

---

## Common gotchas

### Filter with DI deps applied as a plain attribute

A plain `[MyFilter]` attribute can't receive constructor DI. Use `[ServiceFilter]` (registered) or `[TypeFilter]` (DI-constructed) for filters needing dependencies.

### Exception filter instead of middleware for global errors

Exception filters only catch **action** exceptions, missing middleware/binding errors. Use `UseExceptionHandler` for the global boundary.

### Filter ordering confusion

"Executing" runs global→controller→action; "executed" runs the reverse. If ordering matters, set the `Order` property explicitly.

### Endpoint filter not returning the result

An `IEndpointFilter` must return the result of `await next(ctx)` (or its own short-circuit result). Forgetting to return drops the response.

### Heavy/blocking work in filters

Filters run on the request thread; blocking starves the pool. Use async filters and `await`.

---

## Summary

- **Filters** run cross-cutting logic around endpoints. **MVC** has a typed pipeline (authorization → resource → action → exception → result filters); **Minimal APIs** have lighter **endpoint filters** (`IEndpointFilter`, applied per-endpoint/group).
- Apply MVC filters globally, per-controller, or per-action; use **`[ServiceFilter]`/`[TypeFilter]`** for filters with DI dependencies. Use **async** filter variants for I/O.
- Prefer **exception-handling middleware** (`UseExceptionHandler` → ProblemDetails) for global errors; use exception filters only for MVC-specific exception→result mapping.
- Choose by what you need to see: **middleware** (raw HTTP, all requests), **filters/endpoint filters** (bound args + endpoint metadata), **decorators** (a specific service).
- Prefer built-ins (`[Authorize]`, output caching, auto-validation) over hand-rolled filters; keep filters async and remember to return the result.

→ Next: [10-OpenAPI.md](10-OpenAPI.md)
