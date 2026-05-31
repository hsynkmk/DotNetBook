# Validation

## Checking incoming data

Validation ensures request data meets your rules **before** it reaches business logic, returning a clear **400 Bad Request** (with details) when it doesn't. This is *request/input* validation — distinct from options validation ([Ch03 §10](../03-HostingAndDI/10-Validation.md)) and argument guards (CSharpBook Ch17 §08). The three approaches: DataAnnotations, FluentValidation, and Minimal-API validation.

---

## DataAnnotations (attributes on the model)

Decorate the DTO with validation attributes; the framework checks them after binding:

```csharp
public class CreateProduct {
    [Required, StringLength(100, MinimumLength = 1)]
    public string Name { get; set; } = "";

    [Range(0.01, 1_000_000)]
    public decimal Price { get; set; }

    [Required, EmailAddress]
    public string ContactEmail { get; set; } = "";

    [Url] public string? Website { get; set; }

    [RegularExpression(@"^[A-Z]{2}\d{4}$")]
    public string Sku { get; set; } = "";
}
```

Common attributes: `[Required]`, `[StringLength]`/`[MaxLength]`/`[MinLength]`, `[Range]`, `[EmailAddress]`, `[Url]`, `[Phone]`, `[RegularExpression]`, `[Compare]` (match another property), `[CreditCard]`. Custom rules via a `ValidationAttribute` subclass or `IValidatableObject` (for cross-property logic).

---

## MVC: automatic 400 with `[ApiController]`

In MVC, `[ApiController]` (controller [03-MVC.md](03-MVC.md)) **automatically** returns a 400 ProblemDetails when `ModelState` is invalid — before your action runs:

```csharp
[ApiController]
public class ProductsController : ControllerBase {
    [HttpPost]
    public ActionResult<Product> Create(CreateProduct dto) {
        // If dto is invalid, this NEVER runs — framework already returned 400 ProblemDetails.
        return Ok(svc.Create(dto));
    }
}
```

Without `[ApiController]` you'd check manually:

```csharp
if (!ModelState.IsValid) return BadRequest(ModelState);   // only needed without [ApiController]
```

The automatic 400 uses a **ValidationProblemDetails** body (RFC 7807 — [12-ProblemDetails.md](12-ProblemDetails.md)) listing each invalid field and its errors — a consistent, machine-readable error shape.

---

## Minimal APIs: validation isn't automatic — add it

Minimal APIs do **not** run DataAnnotations automatically by default. Options:

```csharp
// Option A: an endpoint filter that validates (reusable)
public class ValidationFilter<T> : IEndpointFilter {
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext ctx, EndpointFilterDelegate next) {
        var model = ctx.Arguments.OfType<T>().FirstOrDefault();
        if (model is not null) {
            var results = new List<ValidationResult>();
            if (!Validator.TryValidateObject(model, new ValidationContext(model), results, true)) {
                var errors = results.ToDictionary(r => r.MemberNames.FirstOrDefault() ?? "", r => new[] { r.ErrorMessage ?? "" });
                return Results.ValidationProblem(errors);   // 400 ValidationProblemDetails
            }
        }
        return await next(ctx);
    }
}
app.MapPost("/products", (CreateProduct dto) => ...).AddEndpointFilter<ValidationFilter<CreateProduct>>();
```

```csharp
// Option B (.NET 10): built-in Minimal API validation
builder.Services.AddValidation();          // enables DataAnnotation validation for endpoints
// then [Required] etc. on parameters/DTOs are enforced, returning 400 automatically
```

.NET 10 adds first-class Minimal API validation (`AddValidation()`), so DataAnnotations on Minimal endpoints are enforced like MVC. For earlier versions or richer rules, use an endpoint filter or FluentValidation. Either way, return a **`Results.ValidationProblem`** (RFC 7807) for consistency.

---

## FluentValidation (rich, testable rules)

For complex validation — conditional rules, cross-property, collections, custom messages — **FluentValidation** is the popular choice:

```csharp
public class CreateProductValidator : AbstractValidator<CreateProduct> {
    public CreateProductValidator(ICategoryRepo categories) {   // can inject dependencies!
        RuleFor(x => x.Name).NotEmpty().MaximumLength(100);
        RuleFor(x => x.Price).GreaterThan(0).LessThanOrEqualTo(1_000_000);
        RuleFor(x => x.ContactEmail).NotEmpty().EmailAddress();
        RuleFor(x => x.CategoryId)
            .MustAsync(async (id, ct) => await categories.ExistsAsync(id, ct))   // async DB check
            .WithMessage("Category does not exist");
        RuleForEach(x => x.Tags).NotEmpty();                    // validate collections
        When(x => x.IsOnSale, () => RuleFor(x => x.SalePrice).NotNull());  // conditional
    }
}
builder.Services.AddScoped<IValidator<CreateProduct>, CreateProductValidator>();
```

FluentValidation advantages: rules live in a separate, **unit-testable** class; supports **async** rules (DB/remote checks), conditionals (`When`), collections (`RuleForEach`), and great messages. Invoke it in an endpoint filter / action filter or via the FluentValidation ASP.NET integration, returning a 400 with the failures.

```csharp
// Using the injected validator in a Minimal endpoint
app.MapPost("/products", async (CreateProduct dto, IValidator<CreateProduct> validator) => {
    var result = await validator.ValidateAsync(dto);
    if (!result.IsValid) return Results.ValidationProblem(result.ToDictionary());
    ...
});
```

---

## Validate DTOs, not entities

Always validate (and bind to) a **DTO** with exactly the fields a client may supply — never your domain entity:

```csharp
// ✗ — binding/validating the entity exposes internal fields (mass assignment) and couples your API to your schema
public IActionResult Create(Product entity) { ... }

// ✓ — a DTO with only the allowed inputs; map to the entity after validation
public IActionResult Create(CreateProductDto dto) { var entity = dto.ToEntity(); ... }
```

This prevents over-posting (a client setting `IsAdmin`/`Id`/`CreatedAt`) and keeps the API contract decoupled from the persistence model ([Ch07 ModelBinding §07], [Ch22](../22-BestPractices/README.md)).

---

## Layered validation (defense in depth)

Validation happens at multiple layers, each with a purpose:
- **Request validation** (this file) — shape/format of incoming data → **400**.
- **Domain invariants** — business rules enforced in the domain model (e.g., "an order can't ship before payment") → domain exceptions or result types (CSharpBook Ch17 §13).
- **Database constraints** — last line of defense (unique, FK, not-null) → handle conflicts → **409**.

Request validation gives fast, user-friendly feedback; don't rely on it alone for business rules (those belong in the domain). Don't duplicate every DB constraint as request validation, but do validate what gives the best UX.

---

## Common gotchas

### Expecting automatic validation in Minimal APIs (pre-.NET 10)

MVC's `[ApiController]` auto-validates; Minimal APIs historically did not. Add an endpoint filter, use `.AddValidation()` (.NET 10), or FluentValidation.

### Validating the entity (over-posting)

Binding/validating a domain entity lets clients set fields they shouldn't. Use DTOs.

### Inconsistent error shapes

Returning ad-hoc error JSON from some endpoints and ProblemDetails from others confuses clients. Standardize on **ValidationProblemDetails** (RFC 7807) everywhere ([12-ProblemDetails.md](12-ProblemDetails.md)).

### Business rules as DataAnnotations

`[Range]` can't express "category must exist" or "sale price required when on sale." Use FluentValidation (async/conditional) or domain logic for those.

### Trusting client validation

Browser/JS validation is UX, not security. Always validate on the server — clients can be bypassed.

---

## Summary

- **Request validation** checks incoming data and returns **400 (ValidationProblemDetails, RFC 7807)** — distinct from options validation and argument guards.
- **DataAnnotations** (`[Required]`, `[Range]`, `[EmailAddress]`, …) are declarative; **MVC `[ApiController]` auto-returns 400** on invalid `ModelState`. **Minimal APIs** need an endpoint filter, `.AddValidation()` (.NET 10), or FluentValidation.
- **FluentValidation** handles complex rules (async DB checks, conditionals, collections) in testable, DI-injectable validator classes.
- **Validate DTOs, not entities** (prevent over-posting); standardize error responses on ProblemDetails.
- Use **layered validation**: request validation for UX/format, domain invariants for business rules, DB constraints as the backstop. Never trust client-side validation alone.

→ Next: [09-Filters.md](09-Filters.md)
