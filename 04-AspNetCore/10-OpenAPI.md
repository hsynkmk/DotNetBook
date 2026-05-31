# OpenAPI

## Describing your API

OpenAPI (formerly Swagger) is a standard, machine-readable description of an HTTP API — its endpoints, parameters, request/response shapes, and status codes — as a JSON/YAML document. From it, tools generate interactive docs, client SDKs, and test collections. ASP.NET Core can produce this document for you.

```csharp
// .NET 9+ built-in OpenAPI document generation
builder.Services.AddOpenApi();
var app = builder.Build();
if (app.Environment.IsDevelopment())
    app.MapOpenApi();          // serves the doc at /openapi/v1.json
app.Run();
```

That exposes `/openapi/v1.json` — a complete description of your endpoints, generated from your code, route metadata, and types.

---

## Built-in OpenAPI (.NET 9+) vs Swashbuckle/NSwag

For years, **Swashbuckle** (Swagger) and **NSwag** were the de-facto OpenAPI libraries. .NET 9 added **`Microsoft.AspNetCore.OpenApi`** — built-in document generation (no third-party dependency) that integrates with Minimal APIs' metadata and source-generation:

| | Built-in `AddOpenApi` (.NET 9+) | Swashbuckle / NSwag |
|---|---|---|
| Document generation | ✓ (built-in, AOT-friendlier) | ✓ |
| Swagger UI (interactive page) | ✗ (doc only — add a UI separately) | ✓ (bundled UI) |
| Client SDK generation | ✗ | NSwag ✓ |
| Maturity | newer | mature, feature-rich |

The built-in package generates the **document**; for the interactive **Swagger UI** you add a UI package (e.g., Swashbuckle's UI, or modern alternatives like **Scalar**) pointed at the doc. For new projects, the built-in document + a lightweight UI is the modern path; Swashbuckle/NSwag remain common where you need their richer features or client generation.

```csharp
// Add an interactive UI on top of the built-in document (example with Scalar)
app.MapScalarApiReference();   // or use Swashbuckle's UI middleware pointed at /openapi/v1.json
```

---

## Enriching the document

OpenAPI quality depends on the metadata you attach. Minimal APIs use fluent methods; the framework also infers a lot from `TypedResults` and parameter types:

```csharp
app.MapGet("/products/{id:int}", (int id, IProductSvc svc) =>
        svc.Get(id) is { } p ? TypedResults.Ok(p) : TypedResults.NotFound())
    .WithName("GetProduct")
    .WithSummary("Get a product by id")
    .WithDescription("Returns the product, or 404 if not found.")
    .WithTags("Products")
    .Produces<Product>(StatusCodes.Status200OK)
    .Produces(StatusCodes.Status404NotFound);

// Or let TypedResults + result unions infer responses automatically:
app.MapGet("/p/{id:int}", Results<Ok<Product>, NotFound> (int id) => ...);
//   OpenAPI infers 200→Product and 404 from the return type
```

- **`WithSummary`/`WithDescription`/`WithTags`** add human docs and grouping.
- **`Produces<T>(statusCode)`** declares response types/codes (or use `TypedResults`/`Results<T1,T2>` to infer them).
- **`Accepts<T>("media-type")`** declares the request body type.

For MVC, XML doc comments + `[ProducesResponseType(typeof(Product), 200)]` attributes drive the document.

---

## Schemas, examples, and security

The document also describes **data shapes** (schemas, derived from your DTOs) and can include examples and security schemes:

```csharp
builder.Services.AddOpenApi(o => {
    o.AddDocumentTransformer((doc, ctx, ct) => {
        doc.Info.Title = "Orders API";
        doc.Info.Version = "v1";
        // add a bearer security scheme so the UI shows an "Authorize" button
        doc.Components ??= new();
        doc.Components.SecuritySchemes["bearer"] = new() { Type = ..., Scheme = "bearer" };
        return Task.CompletedTask;
    });
});
```

Document/operation **transformers** let you post-process the generated doc (titles, servers, security schemes, examples). DTO property attributes (`[Required]`, `[Range]`, XML docs) flow into the schemas, so good DTOs and validation attributes ([08-Validation.md](08-Validation.md)) produce a good schema for free.

---

## What OpenAPI enables

A good OpenAPI document powers:
- **Interactive docs** — a UI to explore and try endpoints.
- **Client SDK generation** — generate typed C#/TypeScript/etc. clients (NSwag, `kiota`, openapi-generator), so consumers don't hand-write HTTP calls.
- **Contract testing & mocking** — validate requests/responses against the schema; spin up mock servers.
- **API gateways / tooling** — import into Postman, gateways, etc.

This is why investing in accurate metadata pays off — the document is the contract consumers (and tools) rely on.

---

## Versioning & multiple documents

For versioned APIs ([06-Routing.md](06-Routing.md)), generate a document per version:

```csharp
builder.Services.AddOpenApi("v1");
builder.Services.AddOpenApi("v2");
// → /openapi/v1.json and /openapi/v2.json
```

Each version gets its own document so clients target a stable contract.

---

## Common gotchas

### Built-in package doesn't include a UI

`AddOpenApi`/`MapOpenApi` generate the **document** only. For an interactive page, add a UI (Scalar, Swashbuckle UI, etc.) pointed at the doc — don't expect a UI out of the box (unlike old Swashbuckle).

### Exposing the doc/UI in production

OpenAPI docs reveal your API surface. Guard `MapOpenApi`/UI with `IsDevelopment()` (or auth) unless public docs are intended.

### Inaccurate response types

If you return raw objects without declaring types, OpenAPI may show `200` with no schema or miss `404`. Use **`TypedResults`/`Results<T1,T2>`** or `Produces<T>`/`[ProducesResponseType]` so the doc matches reality.

### Drift between docs and behavior

Hand-written docs rot. Generating from code keeps the document in sync — keep metadata accurate as endpoints change.

### Leaking internal types

OpenAPI schemas come from your response types. Returning domain entities exposes internal shape; return **DTOs** so the contract is intentional ([08-Validation.md](08-Validation.md)).

---

## Summary

- **OpenAPI** is a standard, machine-readable description of your API (endpoints, params, schemas, status codes) that powers docs, client SDK generation, and tooling.
- **.NET 9+ built-in** `AddOpenApi`/`MapOpenApi` generates the **document** (AOT-friendlier, no third-party dep); add a separate **UI** (Scalar/Swashbuckle UI) for interactivity. Swashbuckle/NSwag remain for richer features and client generation.
- Enrich with **`WithSummary`/`Produces<T>`/`WithTags`** (Minimal) or `[ProducesResponseType]` + XML docs (MVC); **`TypedResults`/`Results<T1,T2>`** let response types be inferred.
- DTO attributes/validation flow into **schemas**; transformers customize the doc (titles, security schemes).
- Generate **per-version** documents; guard docs/UI in production; return **DTOs** (not entities) so the contract is intentional and accurate.

→ Next: [11-StaticFiles.md](11-StaticFiles.md)
