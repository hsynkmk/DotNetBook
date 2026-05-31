# Routing

## Matching URLs to endpoints

Routing maps an incoming request's **method + path** to an **endpoint** (a Minimal API handler, MVC action, Razor Page, or other terminal). Modern ASP.NET Core uses **endpoint routing**: `UseRouting` matches the request to an endpoint early in the pipeline, then `UseEndpoints`/the terminal middleware runs it — so middleware in between (auth, etc.) can see the matched endpoint's metadata.

```csharp
app.MapGet("/products/{id:int}", (int id) => ...);       // method + template + handler
app.MapGet("/products/{category}/{id:int?}", ...);        // multiple params, optional
app.MapPost("/orders", ...);
```

A **route template** (`/products/{id:int}`) has literal segments and **parameters** (`{id}`) with optional **constraints** (`:int`). The router picks the **most specific** matching endpoint.

---

## Route parameters & constraints

```csharp
"{id:int}"                  // must parse as int
"{id:guid}"                 // must be a GUID
"{slug}"                    // any string
"{id:int?}"                 // optional (matches with or without)
"{page:int=1}"              // default value (1 if absent)
"{name:minlength(3)}"       // string constraint
"{price:decimal:min(0)}"    // chained constraints
"{*rest}"                   // catch-all (captures the remaining path, incl. slashes)
```

Common constraints: `int`, `long`, `guid`, `bool`, `datetime`, `decimal`, `alpha`, `regex(...)`, `min(n)`, `max(n)`, `minlength(n)`, `maxlength(n)`, `range(a,b)`, `required`. Constraints **filter matching** (a non-int `id` won't match `{id:int}`) but are **not validation** — they're for routing, not business rules (validate values in the handler/model — [08-Validation.md](08-Validation.md)).

```csharp
app.MapGet("/files/{*path}", (string path) => ...);   // catch-all: /files/a/b/c.txt → path = "a/b/c.txt"
```

---

## Route values, query, and precedence

```csharp
// /products/42?sort=price&page=2
app.MapGet("/products/{id:int}", (int id, string? sort, int page = 1) => ...);
//   id    ← route value (from {id})
//   sort  ← query string (?sort=)
//   page  ← query string, default 1
```

Route parameters come from the path; everything else (simple types not in the route) comes from the query string. When multiple routes could match, the router prefers the **most specific** (more literal segments, tighter constraints) over the more general — `/products/featured` beats `/products/{id}`.

---

## Route groups (Minimal APIs)

```csharp
var api = app.MapGroup("/api/v1");
var products = api.MapGroup("/products").WithTags("Products").RequireAuthorization();

products.MapGet("/", () => ...);           // GET  /api/v1/products
products.MapGet("/{id:int}", (int id) => ...); // GET  /api/v1/products/42
products.MapPost("/", (...) => ...);        // POST /api/v1/products
```

`MapGroup` ([Ch04 §02](02-MinimalAPIs.md)) shares a path prefix and conventions (auth, filters, OpenAPI tags) across endpoints, and **nests**. The canonical way to organize Minimal API routes by resource/version.

---

## Attribute routing (MVC)

```csharp
[Route("api/[controller]")]                  // [controller] → "products"
public class ProductsController : ControllerBase {
    [HttpGet("{id:int}")]                     // GET /api/products/42
    [HttpGet("search")]                       // GET /api/products/search
    [HttpPost]                                // POST /api/products
}
```

MVC uses route templates on attributes (`[Route]`, `[HttpGet("...")]`) with the same template/constraint syntax. `[controller]`/`[action]` tokens substitute the names. (MVC: [03-MVC.md](03-MVC.md).)

---

## Link generation (don't hardcode URLs)

Generate URLs from **named** routes/endpoints rather than concatenating strings — so URL changes don't break links:

```csharp
// Name an endpoint
app.MapGet("/products/{id:int}", (int id) => ...).WithName("GetProduct");

// Generate its URL by name (LinkGenerator)
app.MapPost("/products", (Product p, LinkGenerator links, HttpContext ctx) => {
    var url = links.GetUriByName(ctx, "GetProduct", new { id = p.Id });
    return Results.Created(url!, p);
});
```

```csharp
// MVC: CreatedAtAction / Url.Action
return CreatedAtAction(nameof(Get), new { id = product.Id }, product);
```

`LinkGenerator` (Minimal APIs) and `Url.Action`/`CreatedAtAction` (MVC) build correct URLs from route names + values, surviving route refactors. Razor uses `asp-page`/`asp-route-*` tag helpers ([04-RazorPages.md](04-RazorPages.md)).

---

## How matching works (briefly)

ASP.NET Core builds a **route table** of all endpoints at startup and matches requests using an efficient tree (a "DFA" over path segments) — so matching is fast and independent of endpoint count. `UseRouting` performs the match and stores the selected endpoint on `HttpContext`; subsequent middleware (auth, etc.) read its metadata (`[Authorize]`, rate-limit policies, CORS policy) before the endpoint executes. This is why routing sits **before** auth in the pipeline ([05-Middleware.md](05-Middleware.md)).

---

## API versioning (brief)

Public APIs evolve; versioning lets you change without breaking clients. The `Asp.Versioning` library adds version-aware routing:

```csharp
builder.Services.AddApiVersioning();
// URL-based: /api/v1/products, /api/v2/products
// or header/query-based versioning
```

Strategies: URL segment (`/v1/`), query (`?api-version=1.0`), or header. URL-segment versioning is the most explicit and cache-friendly. Plan versioning early for public APIs. (API evolution: [Ch22 §09](../22-BestPractices/README.md), CSharpBook Ch17 §04.)

---

## Common gotchas

### Constraints aren't validation

`{id:int}` ensures the route only matches integer-looking values — it doesn't validate business rules (range, existence). A non-matching value yields **404** (not 400). Validate real constraints in the handler/model.

### Ambiguous routes

Two endpoints that match the same request → an `AmbiguousMatchException`. Make routes distinct (different templates/constraints) or order specificity correctly.

### Hardcoding URLs

String-concatenated URLs break when routes change. Use `LinkGenerator`/`Url.Action`/named routes.

### Catch-all eating everything

`{*path}` matches greedily, including sub-paths — place it carefully so it doesn't shadow more specific routes.

### Routing after auth

Auth needs the matched endpoint's metadata, so `UseRouting` must come **before** `UseAuthentication`/`UseAuthorization`. Wrong order means auth can't see `[Authorize]`.

### Trailing-slash / case assumptions

Routing is case-insensitive by default; trailing slashes and casing are normalized, but don't rely on a specific form for security decisions.

---

## Summary

- **Endpoint routing** maps method + path to an endpoint via **route templates** (`/products/{id:int}`) with **parameters** and **constraints**; the **most specific** match wins.
- Constraints (`:int`, `:guid`, `min(n)`, `regex(...)`, `{*catchall}`) **filter matching** (mismatch → 404) — they're **not validation**.
- Route params come from the path, other simple params from the query; **`MapGroup`** (Minimal) and **attribute routes** (MVC) organize endpoints.
- **Generate URLs** with `LinkGenerator`/`Url.Action`/named routes — never hardcode them.
- `UseRouting` matches early so middleware (auth) sees endpoint metadata — hence routing **before** auth.
- Use **`Asp.Versioning`** for public-API versioning (URL-segment is clearest); plan it early.

→ Next: [07-ModelBinding.md](07-ModelBinding.md)
