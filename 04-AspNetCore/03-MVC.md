# MVC Controllers

## Convention-based endpoints in classes

MVC (Model-View-Controller) organizes endpoints into **controller classes** with **action methods**. It predates Minimal APIs and remains the choice for larger APIs/apps that benefit from conventions, attribute routing on classes, built-in model binding/validation, and the rich filter pipeline. For APIs you typically use **API controllers** (the "VC" without server-rendered views).

```csharp
// Program.cs
builder.Services.AddControllers();       // register MVC services
var app = builder.Build();
app.MapControllers();                     // map attribute-routed controllers

// ProductsController.cs
[ApiController]
[Route("api/[controller]")]               // → /api/products
public class ProductsController(IProductService svc) : ControllerBase {

    [HttpGet("{id:int}")]                  // GET /api/products/42
    public ActionResult<Product> Get(int id) =>
        svc.Get(id) is { } p ? Ok(p) : NotFound();

    [HttpPost]                             // POST /api/products
    public ActionResult<Product> Create(CreateProduct dto) {
        var product = svc.Create(dto);
        return CreatedAtAction(nameof(Get), new { id = product.Id }, product);
    }
}
```

A controller is a class deriving from `ControllerBase` (API) or `Controller` (with view support). Action methods map to routes via attributes; the framework binds parameters, runs filters, and serializes the result.

---

## `[ApiController]` — the API conventions

The `[ApiController]` attribute opts a controller into API-specific behaviors that remove boilerplate:

- **Automatic model validation** — if `ModelState` is invalid, returns **400 with a ProblemDetails** body automatically (no manual `if (!ModelState.IsValid)` — [08-Validation.md](08-Validation.md)).
- **Binding source inference** — complex types from body, simple types from route/query (like Minimal APIs), so you rarely need `[FromBody]`.
- **`ProblemDetails` for error status codes** — non-success results get RFC 7807 bodies ([12-ProblemDetails.md](12-ProblemDetails.md)).
- **Attribute routing required** — no convention-based fallback routes.

Always use `[ApiController]` for API controllers — it makes them behave like a well-formed HTTP API with minimal code.

---

## Attribute routing

Routes are declared with attributes on the controller and actions:

```csharp
[Route("api/v{version:apiVersion}/[controller]")]   // [controller] → "products"
public class ProductsController : ControllerBase {
    [HttpGet]                          // GET    /api/v1/products
    [HttpGet("{id:int}")]              // GET    /api/v1/products/42
    [HttpPost]                         // POST   /api/v1/products
    [HttpPut("{id:int}")]              // PUT    /api/v1/products/42
    [HttpDelete("{id:int}")]           // DELETE /api/v1/products/42
    [HttpGet("search")]                // GET    /api/v1/products/search
}
```

`[controller]`/`[action]` tokens fill in the names; route constraints (`:int`, `:guid`) and parameters work as in [06-Routing.md](06-Routing.md). Each action's HTTP verb attribute (`[HttpGet]`, etc.) defines its method and sub-route.

---

## Action results

Actions return `IActionResult`, `ActionResult<T>`, or a value:

```csharp
public ActionResult<Product> Get(int id) {
    var p = svc.Get(id);
    if (p is null) return NotFound();              // 404
    return Ok(p);                                   // 200 + JSON  (or just `return p;`)
}

// Helper methods on ControllerBase:
return Ok(data);                  // 200
return CreatedAtAction(nameof(Get), new { id }, data);  // 201 + Location
return NoContent();               // 204
return BadRequest(ModelState);    // 400
return NotFound();                // 404
return Unauthorized();            // 401
return Forbid();                  // 403
return Conflict();                // 409
return Problem("detail", statusCode: 500);  // RFC 7807
return File(bytes, "application/pdf");
```

`ActionResult<T>` lets an action return either a typed value (serialized 200) or an `IActionResult` (a status result) — giving both type clarity (for OpenAPI) and flexibility.

---

## Model binding & validation

MVC binds action parameters from route/query/body/form/header (same sources as Minimal APIs) and runs validation on the bound model:

```csharp
public class CreateProduct {
    [Required] public string Name { get; set; } = "";
    [Range(0, 100000)] public decimal Price { get; set; }
}

[HttpPost]
public ActionResult<Product> Create(CreateProduct dto) {
    // With [ApiController], invalid dto → automatic 400 ProblemDetails BEFORE this runs
    return Ok(svc.Create(dto));
}
```

With `[ApiController]`, validation failures short-circuit to a 400 automatically. Binding details: [07-ModelBinding.md](07-ModelBinding.md); validation: [08-Validation.md](08-Validation.md).

---

## The filter pipeline

MVC has a rich **filter pipeline** that runs around action execution — authorization, resource, action, exception, and result filters. This is MVC's main advantage over Minimal APIs for cross-cutting concerns:

```csharp
[Authorize]                                  // authorization filter
[ServiceFilter(typeof(AuditFilter))]         // a DI-resolved action filter
public class OrdersController : ControllerBase {
    [HttpPost]
    [ValidateAntiForgeryToken]
    public IActionResult Create(...) { ... }
}

public class AuditFilter(ILogger<AuditFilter> log) : IActionFilter {
    public void OnActionExecuting(ActionExecutingContext ctx) => log.LogInformation("→ {Action}", ctx.ActionDescriptor.DisplayName);
    public void OnActionExecuted(ActionExecutedContext ctx) { }
}
```

Filters apply at the action, controller, or global level. Order and types (authorization → resource → action → exception → result) are covered in [09-Filters.md](09-Filters.md).

---

## MVC vs Minimal APIs — choosing

| | Minimal APIs | MVC Controllers |
|---|---|---|
| Ceremony | low (lambdas) | higher (classes/attributes) |
| Best for | APIs, microservices, AOT | large APIs/apps, conventions |
| Cross-cutting | endpoint filters | rich filter pipeline |
| Native AOT | first-class (RDG) | not AOT-friendly (reflection-based) |
| Views (server UI) | no | yes (`Controller` + Razor views) |
| Familiarity | newer | mature, widely known |

Use **Minimal APIs** for new APIs (especially AOT/cloud-native); use **MVC** for large applications with many endpoints, server-rendered views, or teams invested in the controller/filter model. They can coexist in one app. The underlying routing, binding, validation, and DI are shared.

---

## Server-rendered views (brief)

`Controller` (vs `ControllerBase`) adds view support — returning `View(model)` renders a Razor `.cshtml` template to HTML:

```csharp
public class HomeController : Controller {
    public IActionResult Index() => View(new HomeViewModel { ... });   // renders Views/Home/Index.cshtml
}
```

For page-centric server-rendered UI, **Razor Pages** ([04-RazorPages.md](04-RazorPages.md)) is usually a better fit than MVC views. For interactive UI, **Blazor** ([Ch14](../14-Blazor/README.md)). MVC views remain common in existing apps.

---

## Common gotchas

### Forgetting `[ApiController]`

Without it you lose automatic 400 validation, binding inference, and ProblemDetails — you'd hand-write `if (!ModelState.IsValid) return BadRequest(...)`. Add it to API controllers.

### Forgetting `AddControllers()` / `MapControllers()`

Both are required — register MVC services and map the controllers. Missing either means no controller routes.

### Fat controllers

Controllers should be thin: bind → call a service → return a result. Business logic belongs in services (avoid the "fat controller" anti-pattern — [Ch22](../22-BestPractices/README.md)).

### Using MVC where Minimal APIs fit (and AOT)

For small APIs or AOT targets, MVC's overhead and reflection-based pipeline are unnecessary. Minimal APIs are leaner and AOT-friendly.

### Mixing `Controller` and `ControllerBase`

Use `ControllerBase` for APIs (no view machinery), `Controller` only when you render views. `ControllerBase` is lighter.

---

## Summary

- **MVC** organizes endpoints into **controller classes** with **action methods**, mapped via **attribute routing**; use **`[ApiController]`** + `ControllerBase` for APIs.
- `[ApiController]` adds automatic **400 validation (ProblemDetails)**, **binding-source inference**, and ProblemDetails error bodies — register with `AddControllers()` + `MapControllers()`.
- Return **`ActionResult<T>`**/`IActionResult` with helpers (`Ok`, `NotFound`, `CreatedAtAction`, `Problem`); binding/validation work as in Minimal APIs.
- MVC's **rich filter pipeline** (authorization/resource/action/exception/result) is its edge for cross-cutting concerns.
- Choose **Minimal APIs** for new/AOT/microservice APIs, **MVC** for large apps, server-rendered views, or convention-heavy teams; they share routing/binding/validation/DI and can coexist.

→ Next: [04-RazorPages.md](04-RazorPages.md)
