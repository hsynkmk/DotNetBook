# Content Negotiation

## Serving the right format

Content negotiation is how the server picks a **response format** based on what the client asks for (the `Accept` header) and what the server can produce. The same endpoint can return JSON, XML, or another format depending on the client. In practice, modern APIs are JSON-first, but understanding negotiation matters for interop and correctness.

```
Client:  GET /products/42   Accept: application/json
Server:  200 OK             Content-Type: application/json   { ... }

Client:  GET /products/42   Accept: application/xml
Server:  200 OK             Content-Type: application/xml    <Product>...</Product>   (if XML formatter added)
```

---

## How it works (formatters)

ASP.NET Core MVC uses **output formatters** to serialize responses and **input formatters** to deserialize request bodies. Negotiation:
1. The client sends `Accept: application/json, application/xml;q=0.9` (with optional quality weights).
2. MVC picks the first registered output formatter that can produce an accepted media type.
3. It serializes the result with that formatter and sets `Content-Type`.

```csharp
builder.Services.AddControllers()
    .AddXmlSerializerFormatters();    // add XML output/input formatters (JSON is built-in)
```

By default only **JSON** (System.Text.Json) is registered, so JSON is returned regardless of `Accept`. Adding XML formatters enables negotiation between JSON and XML. **Minimal APIs do not do content negotiation** — they serialize to JSON via `System.Text.Json` (you return a specific format explicitly with `Results.Json`/`Results.Text`/`Results.Bytes`).

---

## Controlling the format

```csharp
// Restrict an action to specific produced types
[Produces("application/json")]               // only JSON
[HttpGet]
public ActionResult<Product> Get(int id) => Ok(...);

// Honor the Accept header strictly (406 if no formatter matches), and map extensions
builder.Services.AddControllers(o => {
    o.RespectBrowserAcceptHeader = true;     // don't ignore browser Accept headers
    o.ReturnHttpNotAcceptable = true;         // return 406 if no acceptable formatter
});
```

- **`[Produces("...")]`** constrains what an action emits.
- **`ReturnHttpNotAcceptable = true`** returns **406 Not Acceptable** when the client wants a type the server can't produce (instead of silently returning JSON).
- **`[Consumes("...")]`** restricts which request `Content-Type`s an action accepts (415 otherwise).

---

## JSON configuration (the common case)

Since JSON dominates, configure System.Text.Json globally:

```csharp
// MVC
builder.Services.AddControllers().AddJsonOptions(o => {
    o.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
    o.JsonSerializerOptions.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull;
    o.JsonSerializerOptions.Converters.Add(new JsonStringEnumConverter());
});

// Minimal APIs
builder.Services.ConfigureHttpJsonOptions(o => {
    o.SerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
});
```

The defaults are web-friendly (camelCase, case-insensitive read). For AOT/perf, plug in a source-generated `JsonSerializerContext` ([02-MinimalAPIs.md](02-MinimalAPIs.md), [Ch02 §05](../02-BCL/05-Serialization.md)). JSON behavior (naming, converters, polymorphism, null handling) is all configured here.

---

## `Accept` header & quality values

```
Accept: application/json
Accept: application/json, application/xml;q=0.8       ← prefers JSON, accepts XML
Accept: text/html                                      ← a browser; may get JSON or 406
Accept: */*                                            ← anything (server picks default)
```

The `q` (quality) value (0–1) weights preferences; the server picks the highest-quality type it can produce. Browsers send `text/html,...` — by default MVC ignores the browser Accept header (returns JSON) unless `RespectBrowserAcceptHeader = true`, because browsers asking for HTML on an API endpoint usually still want JSON.

---

## Other formats

```csharp
// Explicit non-negotiated responses (Minimal APIs / any endpoint)
return Results.Json(data);                       // application/json
return Results.Text("plain", "text/plain");
return Results.Bytes(pdfBytes, "application/pdf");
return Results.File(stream, "text/csv", "report.csv");
```

- **CSV / custom formats** — write a custom `OutputFormatter` (MVC) or return `Results.Stream`/`File` with the right content type.
- **Protobuf / MessagePack** — for binary efficiency (often via gRPC instead — [Ch09](../09-NetworkingAndHttp/README.md)).
- **HTML** — that's server-rendered UI (Razor Pages/MVC views — [04-RazorPages.md](04-RazorPages.md)), not API content negotiation.

Most APIs simply return JSON; reach for negotiation/other formats only for genuine multi-format requirements.

---

## Common gotchas

### Expecting Minimal APIs to negotiate

Minimal APIs serialize to JSON; they don't run content negotiation. Return a specific format explicitly (`Results.Json`/`Text`/`Bytes`/`File`).

### XML/other formats not returned despite `Accept`

Only registered formatters participate. Without `AddXmlSerializerFormatters()`, `Accept: application/xml` still yields JSON. Register the formatter to enable negotiation.

### Silent JSON instead of 406

By default an unmatched `Accept` returns JSON. Set `ReturnHttpNotAcceptable = true` to return **406** if you want strict negotiation.

### Inconsistent JSON conventions

Mixing camelCase/PascalCase or differing null handling across endpoints. Configure JSON options **globally** for consistency.

### Over-engineering negotiation

Supporting XML/CSV/etc. "just in case" adds maintenance for unused paths. Be JSON-first; add formats only when a real consumer needs them.

---

## Summary

- **Content negotiation** picks the response format from the client's **`Accept`** header and the server's registered **formatters**; MVC supports it, **Minimal APIs are JSON-only** (return other formats explicitly).
- Default is **JSON** (System.Text.Json); add `AddXmlSerializerFormatters()` to negotiate XML. Use **`[Produces]`/`[Consumes]`** to constrain types and **`ReturnHttpNotAcceptable`** for strict 406 behavior.
- Configure JSON **globally** (`AddJsonOptions` / `ConfigureHttpJsonOptions`) — camelCase, converters, null handling, source-gen for AOT.
- Return other formats (CSV/PDF/binary) explicitly via `Results.Json`/`Text`/`Bytes`/`File` or custom formatters; prefer gRPC/protobuf for binary efficiency.
- Be **JSON-first**; add negotiation/extra formats only when a real consumer requires them.

→ Next: [14-RateLimiting.md](14-RateLimiting.md)
