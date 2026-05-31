# Chapter 04 — ASP.NET Core — Coding Problems

Build a production-shaped API: Minimal API CRUD, validation, error handling, OpenAPI, filters, rate limiting, output caching, and health checks.

---

## Problem 1: A Minimal API CRUD resource

Build `/products` with GET (list), GET by id, POST, PUT, DELETE — using a route group, DI, and typed results.

<details><summary>Solution</summary>

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<IProductStore, InMemoryProductStore>();
var app = builder.Build();

var products = app.MapGroup("/products").WithTags("Products");

products.MapGet("/", (IProductStore store) => store.All());
products.MapGet("/{id:int}", (int id, IProductStore store) =>
    store.Get(id) is { } p ? Results.Ok(p) : Results.NotFound());
products.MapPost("/", (CreateProduct dto, IProductStore store) => {
    var p = store.Add(dto);
    return Results.Created($"/products/{p.Id}", p);
});
products.MapPut("/{id:int}", (int id, UpdateProduct dto, IProductStore store) =>
    store.Update(id, dto) ? Results.NoContent() : Results.NotFound());
products.MapDelete("/{id:int}", (int id, IProductStore store) =>
    store.Delete(id) ? Results.NoContent() : Results.NotFound());

app.Run();

public record CreateProduct(string Name, decimal Price);
public record UpdateProduct(string Name, decimal Price);
```

`MapGroup` shares the prefix/tags; parameters bind from route/body/DI; `Results.Created`/`NoContent`/`NotFound` set correct status codes. ([02-MinimalAPIs.md](02-MinimalAPIs.md).)

</details>

---

## Problem 2: Add validation that returns 400 ProblemDetails

Validate `CreateProduct` and return RFC 7807 errors for invalid input.

<details><summary>Solution</summary>

```csharp
public record CreateProduct(
    [property: Required, StringLength(100)] string Name,
    [property: Range(0.01, 1_000_000)] decimal Price);

// Reusable validation endpoint filter (pre-.NET 10; in .NET 10 use builder.Services.AddValidation())
public class ValidationFilter<T> : IEndpointFilter {
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext ctx, EndpointFilterDelegate next) {
        var model = ctx.Arguments.OfType<T>().FirstOrDefault();
        if (model is not null) {
            var results = new List<ValidationResult>();
            if (!Validator.TryValidateObject(model, new(model), results, true)) {
                var errors = results
                    .SelectMany(r => r.MemberNames.Select(m => (m, r.ErrorMessage)))
                    .GroupBy(x => x.m)
                    .ToDictionary(g => g.Key, g => g.Select(x => x.ErrorMessage ?? "").ToArray());
                return Results.ValidationProblem(errors);   // 400 ValidationProblemDetails
            }
        }
        return await next(ctx);
    }
}

products.MapPost("/", (CreateProduct dto, IProductStore store) => { ... })
    .AddEndpointFilter<ValidationFilter<CreateProduct>>();
```

Returns a standard `ValidationProblemDetails` (field → messages). In .NET 10, `builder.Services.AddValidation()` enables this automatically. ([08-Validation.md](08-Validation.md), [12-ProblemDetails.md](12-ProblemDetails.md).)

</details>

---

## Problem 3: Global error handling with ProblemDetails

Configure global exception handling that maps domain exceptions to status codes.

<details><summary>Solution</summary>

```csharp
builder.Services.AddProblemDetails();
builder.Services.AddExceptionHandler<DomainExceptionHandler>();

var app = builder.Build();
app.UseExceptionHandler();
app.UseStatusCodePages();

public class DomainExceptionHandler(ILogger<DomainExceptionHandler> log) : IExceptionHandler {
    public async ValueTask<bool> TryHandleAsync(HttpContext ctx, Exception ex, CancellationToken ct) {
        log.LogError(ex, "Unhandled exception");                  // log details server-side
        var (status, title) = ex switch {
            NotFoundException  => (StatusCodes.Status404NotFound, "Not found"),
            ConflictException  => (StatusCodes.Status409Conflict, "Conflict"),
            _                  => (StatusCodes.Status500InternalServerError, "Server error"),
        };
        await Results.Problem(statusCode: status, title: title).ExecuteAsync(ctx);  // generic body to client
        return true;
    }
}
```

Handlers throw freely; the handler maps types to status codes, logs internally, and returns a generic ProblemDetails (no internals leaked). ([12-ProblemDetails.md](12-ProblemDetails.md).)

</details>

---

## Problem 4: Custom middleware with correct lifetimes

Write request-timing middleware as a class, using a scoped service correctly.

<details><summary>Solution</summary>

```csharp
public class TimingMiddleware(RequestDelegate next, ILogger<TimingMiddleware> logger) {
    // ctor deps are singleton-scoped (RequestDelegate, ILogger are fine)
    public async Task InvokeAsync(HttpContext context, IRequestAuditor auditor) {
        // SCOPED service injected here (per-request), NOT in the constructor
        var sw = Stopwatch.StartNew();
        await next(context);
        sw.Stop();
        context.Response.Headers["X-Response-Time"] = $"{sw.ElapsedMilliseconds}ms";
        await auditor.RecordAsync(context.Request.Path, sw.Elapsed);
    }
}

app.UseMiddleware<TimingMiddleware>();
builder.Services.AddScoped<IRequestAuditor, RequestAuditor>();
```

Scoped `IRequestAuditor` is injected into `InvokeAsync` (resolved per request), not the constructor — injecting it in the ctor would be a captive dependency. ([05-Middleware.md](05-Middleware.md).)

</details>

---

## Problem 5: OpenAPI with accurate response types

Document the GET-by-id endpoint so OpenAPI shows 200 (Product) and 404.

<details><summary>Solution</summary>

```csharp
builder.Services.AddOpenApi();
var app = builder.Build();
if (app.Environment.IsDevelopment()) app.MapOpenApi();   // /openapi/v1.json (dev only)

app.MapGet("/products/{id:int}",
    Results<Ok<Product>, NotFound> (int id, IProductStore store) =>
        store.Get(id) is { } p ? TypedResults.Ok(p) : TypedResults.NotFound())
    .WithName("GetProduct")
    .WithSummary("Get a product by id")
    .WithTags("Products");
```

`Results<Ok<Product>, NotFound>` + `TypedResults` lets OpenAPI infer both response shapes automatically; `MapOpenApi` is dev-only. ([10-OpenAPI.md](10-OpenAPI.md).)

</details>

---

## Problem 6: Rate limit per client

Apply a per-user (fallback per-IP) token-bucket limit returning 429 with `Retry-After`.

<details><summary>Solution</summary>

```csharp
builder.Services.AddRateLimiter(options => {
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(ctx => {
        var key = ctx.User.Identity?.Name
                  ?? ctx.Connection.RemoteIpAddress?.ToString() ?? "anon";
        return RateLimitPartition.GetTokenBucketLimiter(key, _ => new() {
            TokenLimit = 100, TokensPerPeriod = 10,
            ReplenishmentPeriod = TimeSpan.FromSeconds(1), QueueLimit = 0
        });
    });
    options.OnRejected = async (ctx, ct) => {
        ctx.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
        if (ctx.Lease.TryGetMetadata(MetadataName.RetryAfter, out var ra))
            ctx.HttpContext.Response.Headers.RetryAfter = ((int)ra.TotalSeconds).ToString();
        await ctx.HttpContext.Response.WriteAsync("Rate limit exceeded", ct);
    };
});

var app = builder.Build();
app.UseRateLimiter();
```

Per-client partitioning (one bucket per user/IP), token bucket (sustained 10/sec, burst to 100), and `Retry-After` so clients back off. Note: per-instance — use Redis/gateway for global quotas. ([14-RateLimiting.md](14-RateLimiting.md).)

</details>

---

## Problem 7: Output cache with tag invalidation

Cache the product list, vary by query, and evict on writes.

<details><summary>Solution</summary>

```csharp
builder.Services.AddOutputCache();
var app = builder.Build();
app.UseOutputCache();

app.MapGet("/products", (string? category, IProductStore store) => store.All(category))
    .CacheOutput(p => p.SetVaryByQuery("category").Tag("products").Expire(TimeSpan.FromMinutes(5)));

app.MapPost("/products", async (CreateProduct dto, IProductStore store, IOutputCacheStore cache, CancellationToken ct) => {
    var p = store.Add(dto);
    await cache.EvictByTagAsync("products", ct);          // invalidate cached lists on write
    return Results.Created($"/products/{p.Id}", p);
});
```

Varies by `category` (separate cache entries), tags entries `products`, and evicts them on create — fresh data with cache benefits. Don't cache personalized responses without per-user keys. ([15-OutputCaching.md](15-OutputCaching.md).)

</details>

---

## Problem 8: Liveness and readiness health checks

Expose separate liveness (app-only) and readiness (with DB) endpoints.

<details><summary>Solution</summary>

```csharp
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"])
    .AddDbContextCheck<AppDbContext>("db", tags: ["ready"])
    .AddCheck<RedisHealthCheck>("redis", tags: ["ready"]);

var app = builder.Build();

// Liveness: ONLY app-self (a DB blip must NOT restart the pod)
app.MapHealthChecks("/health/live", new() { Predicate = c => c.Tags.Contains("live") });
// Readiness: app + dependencies (stop routing if a dependency is down)
app.MapHealthChecks("/health/ready", new() { Predicate = c => c.Tags.Contains("ready") });
```

Liveness checks only the app (failing it restarts the container — must not depend on the DB); readiness checks dependencies (failing it stops traffic without restarting). Tags separate them. ([16-HealthChecks.md](16-HealthChecks.md).)

</details>

---

## Problem 9: Bind to a DTO to prevent over-posting

A client shouldn't be able to set `Id`, `IsAdmin`, or `CreatedAt`. Fix this endpoint.

```csharp
app.MapPost("/users", (User user, IUserStore store) => store.Create(user));   // BUG: binds the entity
```

<details><summary>Solution</summary>

```csharp
// DTO with ONLY the fields a client may supply
public record CreateUserDto(string Email, string DisplayName);

app.MapPost("/users", (CreateUserDto dto, IUserStore store) => {
    var user = new User {
        Email = dto.Email,
        DisplayName = dto.DisplayName,
        Id = 0,                                  // server-assigned
        IsAdmin = false,                         // never client-settable
        CreatedAt = DateTimeOffset.UtcNow,        // server-set
    };
    var created = store.Create(user);
    return Results.Created($"/users/{created.Id}", created);
});
```

Binding the entity directly let a client POST `{"isAdmin": true}`. A DTO exposes only allowed fields; the server controls the rest — preventing mass-assignment. ([07-ModelBinding.md](07-ModelBinding.md), [08-Validation.md](08-Validation.md).)

</details>

---

## Problem 10: Compose the full pipeline in correct order

Arrange a production pipeline: exception handling, HTTPS, static files, routing, CORS, auth, rate limiting, output caching, endpoints.

<details><summary>Solution</summary>

```csharp
var app = builder.Build();

// Order is critical:
app.UseExceptionHandler();        // 1. outermost — catch everything below
app.UseHsts();                     // 2. (prod) strict transport security
app.UseHttpsRedirection();         // 3. force https
app.UseStaticFiles();              // 4. serve public assets early (short-circuits)
app.UseRouting();                  // 5. match endpoint (before auth — auth needs metadata)
app.UseCors();                     // 6. cross-origin
app.UseRateLimiter();              // 7. throttle
app.UseAuthentication();           // 8. who are you?
app.UseAuthorization();            // 9. are you allowed?
app.UseOutputCache();              // 10. cache responses
app.MapControllers();              // 11. endpoints (terminal)

app.Run();
```

The canonical order: exception handler outermost, static files before routing, **routing before auth** (auth reads endpoint metadata), authentication before authorization, endpoints last. Wrong order is the most common ASP.NET Core bug. ([05-Middleware.md](05-Middleware.md).)

</details>

---

You can now build a production-shaped HTTP API: CRUD with Minimal APIs, DTO binding (no over-posting), validation and global error handling as ProblemDetails, accurate OpenAPI, custom middleware with correct lifetimes, per-client rate limiting, output caching with invalidation, and liveness/readiness health checks in the right pipeline order.

→ Back to [Chapter 04 README](README.md) · Next chapter: [Chapter 05 — Entity Framework Core](../05-EFCore/README.md)
