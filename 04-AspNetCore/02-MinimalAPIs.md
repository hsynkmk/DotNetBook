# Minimal APIs

## Endpoints as lambdas

Minimal APIs (.NET 6+, matured in 7/8) let you define HTTP endpoints as **route handlers** — lambdas or methods mapped to routes — with no controller class ceremony. Parameters bind automatically from the route, query, body, headers, and DI. They're the modern default for APIs and microservices.

```csharp
var app = builder.Build();

app.MapGet("/products/{id:int}", (int id, IProductService svc) =>
    svc.Get(id) is { } p ? Results.Ok(p) : Results.NotFound());

app.MapPost("/products", (CreateProduct dto, IProductService svc) => {
    var product = svc.Create(dto);
    return Results.Created($"/products/{product.Id}", product);
});

app.MapPut("/products/{id:int}", (int id, UpdateProduct dto, IProductService svc) => { ... });
app.MapDelete("/products/{id:int}", (int id, IProductService svc) => { ... });

app.Run();
```

`MapGet/MapPost/MapPut/MapDelete/MapPatch` map HTTP verbs to handlers. The handler's parameters are resolved by the framework — covered next.

---

## Parameter binding (the rules)

The framework infers where each parameter comes from:

```csharp
app.MapGet("/search/{category}", (
    string category,               // from ROUTE (matches {category})
    int page,                      // from QUERY (?page=2) — simple types default to query
    [FromHeader] string apiKey,    // from a HEADER
    ISearchService svc,            // from DI (registered services)
    CancellationToken ct)          // special: the request's cancellation token
    => svc.Search(category, page, ct));

app.MapPost("/orders", (Order order, IOrderService svc) => svc.Place(order));
//   `order` (a complex type) binds from the JSON request BODY
```

Inference rules (simplified):
- **Route** if the name matches a route parameter (`{id}`).
- **Query string** for simple types (string, int, etc.) not in the route.
- **Body** (JSON) for complex types (one per request).
- **DI** for registered service types.
- **Special** types injected automatically: `HttpContext`, `HttpRequest`, `HttpResponse`, `CancellationToken`, `ClaimsPrincipal`, `IFormFileCollection`.

Override inference with attributes: `[FromRoute]`, `[FromQuery]`, `[FromHeader]`, `[FromBody]`, `[FromServices]`, `[FromForm]`, `[AsParameters]`. (Full binding: [07-ModelBinding.md](07-ModelBinding.md).)

```csharp
// AsParameters — bundle many params into a struct/class (clean signatures)
app.MapGet("/search", ([AsParameters] SearchRequest req, ISearchService svc) => svc.Search(req));
public record struct SearchRequest(string Q, int Page, [FromHeader] string ApiKey);
```

---

## Returning results — `IResult` and `Results`/`TypedResults`

Handlers return a result that the framework turns into an HTTP response. Use the `Results`/`TypedResults` factories for correct status codes and shapes:

```csharp
return Results.Ok(product);                       // 200 + JSON
return Results.Created($"/products/{id}", product); // 201 + Location header
return Results.NoContent();                         // 204
return Results.NotFound();                          // 404
return Results.BadRequest(new { error = "..." });   // 400
return Results.Problem("Something failed");          // RFC 7807 ProblemDetails ([12])
return Results.Json(data, statusCode: 202);
return Results.File(bytes, "application/pdf");
return Results.Stream(stream, "video/mp4");

// TypedResults — strongly-typed (better for testing + OpenAPI inference)
TypedResults.Ok(product);                           // returns Ok<Product>
```

**`TypedResults`** (over `Results`) returns concrete types — better for unit testing handlers (assert on the typed result) and lets OpenAPI infer response types automatically. Returning a raw object (or string) also works — it's serialized as JSON 200.

```csharp
// Multiple possible result types — declare a union for OpenAPI + type safety
app.MapGet("/p/{id:int}", Results<Ok<Product>, NotFound> (int id, IProductService svc) =>
    svc.Get(id) is { } p ? TypedResults.Ok(p) : TypedResults.NotFound());
```

---

## Route groups

Group related endpoints under a common prefix, sharing metadata, filters, and conventions:

```csharp
var products = app.MapGroup("/api/products")
    .WithTags("Products")             // OpenAPI grouping
    .RequireAuthorization();          // auth for the whole group

products.MapGet("/", (...) => ...);
products.MapGet("/{id:int}", (...) => ...);
products.MapPost("/", (...) => ...).RequireAuthorization("Admin");   // override per endpoint

// Nested groups
var v1 = app.MapGroup("/v1");
var v1Products = v1.MapGroup("/products");   // → /v1/products
```

`MapGroup` (`.NET 7+`) reduces repetition and applies cross-cutting concerns (auth, filters, OpenAPI tags) to a set of endpoints at once. Essential for organizing larger APIs.

---

## Endpoint filters

Cross-cutting logic for endpoints (validation, logging, short-circuiting) — the Minimal-API equivalent of MVC action filters:

```csharp
products.MapPost("/", (CreateProduct dto, IProductService svc) => svc.Create(dto))
    .AddEndpointFilter(async (ctx, next) => {
        var dto = ctx.GetArgument<CreateProduct>(0);
        if (string.IsNullOrWhiteSpace(dto.Name))
            return Results.BadRequest("Name required");
        return await next(ctx);          // proceed to the handler
    });

// Reusable filter as a class
public class ValidationFilter<T> : IEndpointFilter where T : IValidatable {
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext ctx, EndpointFilterDelegate next) {
        var arg = ctx.Arguments.OfType<T>().FirstOrDefault();
        var errors = arg?.Validate();
        return errors?.Any() == true ? Results.ValidationProblem(...) : await next(ctx);
    }
}
products.MapPost("/", ...).AddEndpointFilter<ValidationFilter<CreateProduct>>();
```

Filters run around the handler (before and after), can short-circuit, and stack. (Filters in depth, incl. MVC: [09-Filters.md](09-Filters.md).)

---

## Metadata & OpenAPI integration

Fluent methods attach metadata used by OpenAPI, auth, and tooling:

```csharp
products.MapGet("/{id:int}", (int id, IProductService svc) => ...)
    .WithName("GetProduct")                       // named route (for link generation)
    .WithSummary("Get a product by id")
    .WithTags("Products")
    .Produces<Product>(StatusCodes.Status200OK)
    .Produces(StatusCodes.Status404NotFound)
    .RequireAuthorization()
    .WithOpenApi();                               // enrich the OpenAPI operation
```

These build the API's self-description (OpenAPI — [10-OpenAPI.md](10-OpenAPI.md)) and apply behavior (auth, caching, rate limits) declaratively per endpoint or group.

---

## Minimal APIs and AOT

Minimal APIs have first-class **Native AOT** support (the **Request Delegate Generator** source-generates the binding/handler glue at compile time, avoiding the reflection that MVC uses). For AOT/trimmed APIs, Minimal APIs + STJ source generation are the recommended path:

```csharp
var builder = WebApplication.CreateSlimBuilder(args);   // slim builder for AOT/minimal footprint
builder.Services.ConfigureHttpJsonOptions(o =>
    o.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default));   // source-gen JSON
```

This is a key reason Minimal APIs are favored for cloud-native/serverless .NET. (AOT: [Ch19](../19-Deployment/README.md), CSharpBook Ch14.)

---

## Common gotchas

### Wrong binding source inference

A complex type binds from the body, a simple type from query/route. If binding isn't what you expect, be explicit (`[FromQuery]`, `[FromBody]`, etc.). Only one body parameter per request.

### Returning raw objects when you need status control

Returning `product` gives 200; to return 201/404/etc. use `Results`/`TypedResults`. For testability and OpenAPI, prefer `TypedResults` and `Results<T1,T2>` unions.

### Forgetting `CancellationToken`

Accept and forward the request's `CancellationToken` to async calls so work stops when the client disconnects ([Ch01 §08](../01-Runtime/08-Threading.md)).

### Not grouping / repeating concerns

Repeating `RequireAuthorization`/prefixes on every endpoint. Use `MapGroup` to apply them once.

### Heavy logic in the lambda

Inline handlers with lots of logic become unreadable and untestable. Delegate to injected services; keep the handler thin (bind → call service → return result).

---

## Summary

- **Minimal APIs** map HTTP verbs to **route-handler lambdas** (`MapGet/Post/Put/Delete`), with parameters auto-bound from **route, query, body, headers, and DI** (override with `[From*]`; `[AsParameters]` to bundle).
- Return **`Results`/`TypedResults`** for correct status codes (`Ok`, `Created`, `NotFound`, `Problem`); prefer `TypedResults` and `Results<T1,T2>` unions for testability and OpenAPI inference.
- **`MapGroup`** organizes endpoints under a prefix with shared auth/filters/tags; **endpoint filters** add cross-cutting logic (validation, logging) around handlers.
- Fluent **metadata** methods (`WithName`, `Produces`, `RequireAuthorization`, `WithOpenApi`) drive OpenAPI and behavior.
- Minimal APIs have first-class **Native AOT** support (Request Delegate Generator + STJ source-gen) — the cloud-native default. Keep handlers thin; delegate to services.

→ Next: [03-MVC.md](03-MVC.md)
