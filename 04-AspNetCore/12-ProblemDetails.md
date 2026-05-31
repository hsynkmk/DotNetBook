# Problem Details & Error Handling

## A standard shape for HTTP errors

When an API returns an error, clients need a **consistent, machine-readable** body — not ad-hoc JSON that differs per endpoint. **Problem Details** (RFC 7807 / RFC 9457) is that standard: a JSON object with `type`, `title`, `status`, `detail`, and `instance` fields, served as `application/problem+json`. ASP.NET Core produces it for you.

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.4",
  "title": "Not Found",
  "status": 404,
  "detail": "Product 42 does not exist.",
  "instance": "/api/products/42",
  "traceId": "00-abc123..."
}
```

```csharp
builder.Services.AddProblemDetails();    // enable ProblemDetails generation
var app = builder.Build();
app.UseExceptionHandler();                // unhandled exceptions → ProblemDetails
app.UseStatusCodePages();                 // bare status codes (404 etc.) → ProblemDetails
```

---

## Returning Problem Details

```csharp
// Minimal APIs
return Results.Problem(detail: "Product does not exist.", statusCode: 404, title: "Not Found");
return Results.NotFound();                 // also produces a problem body with ProblemDetails enabled
return Results.ValidationProblem(errors);  // 400 with per-field validation errors

// MVC
return Problem(detail: "...", statusCode: 500);
return NotFound();                          // → ProblemDetails (with [ApiController])
return ValidationProblem(ModelState);       // 400 ValidationProblemDetails
```

`ValidationProblemDetails` extends the base shape with an `errors` dictionary (field → messages) — the standard response for validation failures ([08-Validation.md](08-Validation.md)):

```json
{
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "Name": ["The Name field is required."],
    "Price": ["Must be between 0.01 and 1000000."]
  }
}
```

---

## Global exception handling

The cardinal rule (CSharpBook Ch17 §13): **handle errors at a boundary, not in every handler.** `UseExceptionHandler` is that boundary — it catches any unhandled exception from the pipeline/endpoints and converts it to a ProblemDetails response:

```csharp
app.UseExceptionHandler();   // with AddProblemDetails(), unhandled exceptions → 500 problem+json
```

So your handlers can throw freely (or let exceptions bubble) and the middleware produces a clean, consistent error — no try/catch in every endpoint. For custom mapping (domain exceptions → specific status codes), implement `IExceptionHandler` (.NET 8+):

```csharp
public class DomainExceptionHandler : IExceptionHandler {
    public async ValueTask<bool> TryHandleAsync(HttpContext ctx, Exception ex, CancellationToken ct) {
        var (status, title) = ex switch {
            NotFoundException     => (StatusCodes.Status404NotFound, "Resource not found"),
            ValidationException v => (StatusCodes.Status400BadRequest, "Validation failed"),
            ConflictException     => (StatusCodes.Status409Conflict, "Conflict"),
            _                     => (StatusCodes.Status500InternalServerError, "Server error"),
        };
        ctx.Response.StatusCode = status;
        await ctx.Response.WriteAsJsonAsync(new ProblemDetails { Status = status, Title = title, Detail = ex.Message }, ct);
        return true;   // handled
    }
}
builder.Services.AddExceptionHandler<DomainExceptionHandler>();
app.UseExceptionHandler();
```

`IExceptionHandler`s run in registration order; the first to return `true` wins, else the default ProblemDetails applies. This is the modern, clean way to map exception types to HTTP responses centrally.

---

## Development vs production error detail

```csharp
if (app.Environment.IsDevelopment()) {
    app.UseDeveloperExceptionPage();   // detailed stack trace — DEV ONLY
} else {
    app.UseExceptionHandler();          // generic ProblemDetails — no internals leaked
}
```

In **development**, show full exception details (stack traces) to debug. In **production**, return a generic ProblemDetails — **never leak stack traces, exception types, or internal messages** to clients (information disclosure). The `traceId` (correlation/trace ID — [Ch02 §08](../02-BCL/08-Diagnostics.md)) lets you correlate the client's error to your logs without exposing internals.

---

## Customizing the problem body

```csharp
builder.Services.AddProblemDetails(options => {
    options.CustomizeProblemDetails = ctx => {
        ctx.ProblemDetails.Instance = ctx.HttpContext.Request.Path;
        ctx.ProblemDetails.Extensions["traceId"] = ctx.HttpContext.TraceIdentifier;
        ctx.ProblemDetails.Extensions["timestamp"] = DateTimeOffset.UtcNow;
    };
});
```

`CustomizeProblemDetails` lets you enrich every problem response (add a trace ID for support correlation, a timestamp, a documentation link in `type`). The `type` URI ideally points to docs explaining that error class. Keep extensions consistent so clients can rely on them.

---

## Status code semantics

Return the **right** status code (ProblemDetails carries the meaning, but the code drives client behavior):

| Code | Meaning | When |
|---|---|---|
| 400 | Bad Request | malformed/invalid input (validation) |
| 401 | Unauthorized | not authenticated |
| 403 | Forbidden | authenticated but not allowed |
| 404 | Not Found | resource doesn't exist |
| 409 | Conflict | state conflict (duplicate, version) |
| 422 | Unprocessable Entity | semantically invalid (sometimes preferred over 400) |
| 429 | Too Many Requests | rate limited ([14-RateLimiting.md](14-RateLimiting.md)) |
| 500 | Internal Server Error | unhandled server fault |
| 503 | Service Unavailable | overloaded/maintenance |

Use 4xx for client errors (don't return 500 for bad input), 5xx for server faults. Validation → 400; not-found → 404; unhandled exception → 500.

---

## Common gotchas

### Inconsistent error shapes

Some endpoints returning `{ "error": "..." }`, others ProblemDetails. Standardize on ProblemDetails everywhere (`AddProblemDetails` + `UseExceptionHandler` + `UseStatusCodePages`).

### Leaking internals in production

Returning stack traces / exception messages to clients is an information-disclosure risk. Generic ProblemDetails in production; log details server-side; expose only a `traceId`.

### try/catch in every handler

Re-implementing error handling per endpoint is repetitive and inconsistent. Use the global `UseExceptionHandler` / `IExceptionHandler` boundary; let handlers throw.

### Wrong status codes

Returning 500 for validation errors, or 200 with an error in the body. Map client errors to 4xx, server faults to 5xx, with the matching ProblemDetails.

### Forgetting `AddProblemDetails`

Without it, `Results.NotFound()`/exceptions may return bare bodies. `AddProblemDetails` enables the standard shape across the app.

---

## Summary

- **Problem Details (RFC 7807/9457)** is the standard machine-readable error body (`type`, `title`, `status`, `detail`, `instance`) served as `application/problem+json`; **`ValidationProblemDetails`** adds an `errors` map.
- Enable with **`AddProblemDetails()`** + **`UseExceptionHandler()`** (unhandled exceptions → problem) + **`UseStatusCodePages()`** (bare status codes → problem).
- Handle errors at the **global boundary**, not per handler; map exception types to status codes centrally with **`IExceptionHandler`** (.NET 8+).
- **Dev**: detailed errors (`UseDeveloperExceptionPage`); **prod**: generic ProblemDetails — never leak stack traces/internals; expose a `traceId` for log correlation.
- Return the **correct status code** (4xx client, 5xx server); customize the body (`CustomizeProblemDetails`) consistently.

→ Next: [13-ContentNegotiation.md](13-ContentNegotiation.md)
