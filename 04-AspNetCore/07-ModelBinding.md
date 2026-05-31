# Model Binding

## From HTTP to typed parameters

Model binding converts raw HTTP data (route values, query string, headers, request body, form fields) into the typed parameters and model objects your handlers/actions receive. It's why you write `(int id, Product product)` instead of parsing `HttpRequest` by hand.

```csharp
app.MapPost("/products/{id:int}", (
    int id,              // ← route value {id}
    string? sort,        // ← query string ?sort=
    Product product,     // ← JSON request body
    IProductSvc svc) =>  // ← DI
    svc.Update(id, product));
```

The framework inspects each parameter and pulls its value from the appropriate source, converting strings to the target type. Binding works the same conceptually in Minimal APIs and MVC; this file covers both with their differences noted.

---

## Binding sources & inference

| Source | Where data comes from | Default for |
|---|---|---|
| **Route** | path segments (`{id}`) | params matching a route name |
| **Query** | `?key=value` | simple types not in the route |
| **Body** | request body (usually JSON) | complex types (one per request) |
| **Form** | `multipart/form-data` / `x-www-form-urlencoded` | form posts, file uploads |
| **Header** | HTTP headers | explicit `[FromHeader]` |
| **Services** | the DI container | registered service types |

Inference (simplified): route-name match → route; simple type → query; complex type → body; registered service → DI. Override with attributes:

```csharp
app.MapGet("/x", (
    [FromRoute] int id,
    [FromQuery] string q,
    [FromHeader(Name = "X-Api-Key")] string apiKey,
    [FromBody] Payload body,
    [FromServices] IClock clock,
    [FromForm] IFormFile file) => ...);
```

Only **one body** parameter is allowed (the body is a single stream). MVC's `[ApiController]` and Minimal APIs share these inference rules.

---

## Simple type conversion

The binder converts route/query/header **strings** to target types via `TryParse`/`IParsable<T>` (CSharpBook Ch04 §06):

```csharp
app.MapGet("/items", (int page, bool active, DateTime since, Guid id) => ...);
//   "?page=2&active=true&since=2026-05-22&id=..." → typed values
```

Any type implementing **`IParsable<T>`** (or with a `TryParse(string, out T)`) binds from a string automatically — including your own types:

```csharp
public readonly record struct ProductId(int Value) : IParsable<ProductId> {
    public static ProductId Parse(string s, IFormatProvider? p) => new(int.Parse(s, p));
    public static bool TryParse(string? s, IFormatProvider? p, out ProductId r) {
        if (int.TryParse(s, p, out var v)) { r = new(v); return true; }
        r = default; return false;
    }
}
app.MapGet("/p/{id}", (ProductId id) => ...);   // "/p/42" binds to ProductId(42)
```

Conversion failures produce a 400 (Minimal APIs) or a `ModelState` error (MVC) — not an exception in your code.

---

## Complex types from the body (JSON)

```csharp
app.MapPost("/orders", (CreateOrder dto) => ...);   // dto bound from JSON body via System.Text.Json
public record CreateOrder(int CustomerId, List<LineItem> Items, decimal Total);
```

Complex types bind from the JSON body using **System.Text.Json** (the same serializer/options as [Ch13 §05](../13-IO/05-SystemTextJson.md) / [Ch02 §05](../02-BCL/05-Serialization.md)). Records work great as DTOs (immutable, bind via constructor). Configure binding-wide JSON options:

```csharp
builder.Services.ConfigureHttpJsonOptions(o => {
    o.SerializerOptions.PropertyNameCaseInsensitive = true;
    o.SerializerOptions.Converters.Add(new MyConverter());
});
```

For AOT, register a source-generated `JsonSerializerContext` so body binding doesn't use reflection ([02-MinimalAPIs.md](02-MinimalAPIs.md)).

---

## Custom binding

### `BindAsync` (Minimal APIs)

A type can fully control its own binding by implementing a static `BindAsync`:

```csharp
public record PagingParams(int Page, int Size) {
    public static ValueTask<PagingParams?> BindAsync(HttpContext ctx) {
        int page = int.TryParse(ctx.Request.Query["page"], out var p) ? p : 1;
        int size = int.TryParse(ctx.Request.Query["size"], out var s) ? Math.Min(s, 100) : 20;
        return ValueTask.FromResult<PagingParams?>(new(page, size));
    }
}
app.MapGet("/items", (PagingParams paging) => ...);   // binds via PagingParams.BindAsync
```

### `[AsParameters]` (bundle into a struct)

```csharp
app.MapGet("/search", ([AsParameters] SearchQuery q) => ...);
public record struct SearchQuery(string? Term, int Page, [FromHeader] string ApiKey);
```

`[AsParameters]` collapses many bound parameters into one type (each property bound by its own rules) — keeping handler signatures clean. (MVC uses `IModelBinder`/`[ModelBinder]` for custom binding.)

---

## File uploads & forms

```csharp
app.MapPost("/upload", async (IFormFile file, IStorage storage) => {
    await using var stream = file.OpenReadStream();
    await storage.SaveAsync(file.FileName, stream);
    return Results.Ok(new { file.FileName, file.Length });
}).DisableAntiforgery();   // for APIs (forms in Razor keep antiforgery)

app.MapPost("/multi", (IFormFileCollection files, [FromForm] string description) => ...);
```

`IFormFile`/`IFormFileCollection` bind uploaded files; `[FromForm]` binds form fields. **Stream large uploads** (`OpenReadStream`) rather than buffering; enforce size limits (Kestrel `MaxRequestBodySize` and `[RequestSizeLimit]`). Forms in browser apps require anti-forgery tokens (Razor handles this — [04-RazorPages.md](04-RazorPages.md)).

---

## Special injected types

These bind automatically without attributes:

```csharp
app.MapGet("/me", (
    HttpContext ctx,            // the full request context
    ClaimsPrincipal user,       // the authenticated user
    CancellationToken ct) =>    // request cancellation (forward to async calls!)
    user.Identity?.Name);
```

`HttpContext`, `HttpRequest`, `HttpResponse`, `ClaimsPrincipal`, `CancellationToken`, `IFormFileCollection`, and `Stream`/`PipeReader` (raw body) are recognized and supplied by the framework.

---

## Common gotchas

### Wrong source inference

A type binds from the body when you expected query (or vice versa). Be explicit with `[FromQuery]`/`[FromBody]`/`[FromRoute]` when inference isn't what you want. Only one body param allowed.

### Binding failure ≠ exception

A non-parseable value yields a 400 / `ModelState` error, not an exception you can catch in the handler. Check `ModelState` (MVC) or rely on automatic 400 (`[ApiController]` / Minimal APIs).

### Forgetting `CancellationToken`

Not accepting/forwarding it means work continues after the client disconnects — wasted resources. Accept `CancellationToken` and pass it to async calls.

### Buffering large uploads

Reading an `IFormFile` fully into memory blows memory for big files. Stream via `OpenReadStream`; set size limits.

### Over-binding (mass assignment)

Binding a domain entity directly from the body lets clients set fields they shouldn't (e.g., `IsAdmin`). Bind to a **DTO** with only the allowed fields, then map to the entity.

### Case sensitivity

JSON body binding is case-insensitive by default in ASP.NET; query/route are case-insensitive too. Don't rely on exact casing.

---

## Summary

- **Model binding** converts HTTP data (route, query, body, form, headers, DI) into typed parameters/models, using **source inference** (route-name → route, simple → query, complex → body, service → DI), overridable with `[From*]`.
- Simple types convert via **`IParsable<T>`/`TryParse`** (incl. your own types); complex types bind from the **JSON body** via System.Text.Json (configure via `ConfigureHttpJsonOptions`; source-gen for AOT).
- Customize binding with **`BindAsync`** (Minimal APIs) or bundle params with **`[AsParameters]`**; `IFormFile`/`[FromForm]` for uploads (stream them; limit size).
- Special types (`HttpContext`, `ClaimsPrincipal`, **`CancellationToken`**) inject automatically — forward the token to async calls.
- **Bind to DTOs, not entities** (avoid mass-assignment); binding failures yield 400/`ModelState` errors, not catchable exceptions.

→ Next: [08-Validation.md](08-Validation.md)
