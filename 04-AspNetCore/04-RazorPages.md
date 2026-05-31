# Razor Pages

## Page-based server-rendered UI

Razor Pages is ASP.NET Core's model for **page-centric, server-rendered web UI**. Where MVC routes to controller *actions*, Razor Pages routes to *pages* — each page is a `.cshtml` template (markup + Razor syntax) paired with a `PageModel` class (its handlers and state). It's simpler than MVC for form-driven, page-oriented apps (admin panels, CRUD UIs, content sites).

```csharp
// Program.cs
builder.Services.AddRazorPages();
var app = builder.Build();
app.MapRazorPages();
```

```razor
@* Pages/Products/Edit.cshtml *@
@page "{id:int}"
@model EditModel
<h1>Edit @Model.Product.Name</h1>
<form method="post">
    <input asp-for="Product.Name" />
    <span asp-validation-for="Product.Name"></span>
    <button type="submit">Save</button>
</form>
```

```csharp
// Pages/Products/Edit.cshtml.cs — the PageModel (code-behind)
public class EditModel(IProductService svc) : PageModel {
    [BindProperty] public Product Product { get; set; } = default!;

    public IActionResult OnGet(int id) {                 // handles GET /Products/Edit/42
        var p = svc.Get(id);
        if (p is null) return NotFound();
        Product = p;
        return Page();                                    // render the .cshtml
    }

    public IActionResult OnPost(int id) {                 // handles POST (form submit)
        if (!ModelState.IsValid) return Page();           // re-render with validation errors
        svc.Update(Product);
        return RedirectToPage("Index");                   // PRG: redirect after post
    }
}
```

---

## Routing — folder structure is the route

Razor Pages routing is **convention-based on the file path** under `Pages/`:

```
Pages/Index.cshtml              → /
Pages/Products/Index.cshtml     → /Products
Pages/Products/Edit.cshtml      → /Products/Edit
Pages/Products/Edit.cshtml + @page "{id:int}"  → /Products/Edit/42
```

The `@page` directive at the top makes a `.cshtml` a routable page; an optional route template (`@page "{id:int}"`) adds parameters/constraints (same syntax as [06-Routing.md](06-Routing.md)). No attribute routing needed — the folder structure *is* the URL structure, which makes navigation intuitive.

---

## Handler methods — `OnGet`/`OnPost` (and named handlers)

A `PageModel`'s **handler methods** respond to HTTP verbs by naming convention:

```csharp
public IActionResult OnGet() { ... }              // GET
public IActionResult OnPost() { ... }             // POST
public async Task<IActionResult> OnPostAsync() { ... }   // async variant
public IActionResult OnPostDelete(int id) { ... } // NAMED handler: ?handler=Delete
```

```razor
<button type="submit" asp-page-handler="Delete">Delete</button>  @* posts to OnPostDelete *@
```

`OnGet`/`OnPost` map to GET/POST automatically; **named handlers** (`OnPost{Name}`) let one page have multiple POST actions (Save, Delete, Approve) selected via `asp-page-handler`. This keeps a page's related operations together.

---

## Model binding with `[BindProperty]`

Razor Pages binds form/route data to `PageModel` properties marked `[BindProperty]`:

```csharp
[BindProperty] public Product Product { get; set; } = default!;   // bound on POST by default
[BindProperty(SupportsGet = true)] public string? Filter { get; set; }  // also bind on GET

public IActionResult OnPost() {
    // Product is populated from the submitted form fields (asp-for binds them)
    if (!ModelState.IsValid) return Page();
    ...
}
```

`[BindProperty]` binds on POST by default (use `SupportsGet = true` for query/route binding on GET). Handler method parameters (like `OnGet(int id)`) bind from route/query too. Validation runs on the bound model and surfaces in `ModelState` and `asp-validation-for` (see [08-Validation.md](08-Validation.md)).

---

## Tag Helpers — server-aware HTML

Razor Pages (and MVC views) use **Tag Helpers** — attributes that generate correct, model-bound HTML:

```razor
<form method="post">
    <input asp-for="Product.Name" />                  @* name/id/value bound to the model *@
    <span asp-validation-for="Product.Name"></span>   @* shows the validation message *@
    <select asp-for="Product.CategoryId" asp-items="Model.Categories"></select>
    <a asp-page="Index">Back</a>                        @* generates the correct URL *@
    <button type="submit">Save</button>
</form>
@section Scripts { <partial name="_ValidationScriptsPartial" /> }  @* client-side validation *@
```

`asp-for` wires an input to a model property (correct `name` for binding, current value, validation attributes); `asp-page`/`asp-page-handler` generate URLs; `asp-validation-for` displays errors. Tag Helpers keep markup clean and refactor-safe (renaming a property updates the binding). Custom Tag Helpers let you build reusable UI components.

---

## The Post-Redirect-Get pattern

Razor Pages encourages **PRG**: after a successful POST, **redirect** (rather than render), so a browser refresh doesn't re-submit the form:

```csharp
public IActionResult OnPost() {
    if (!ModelState.IsValid) return Page();        // re-render on validation failure
    svc.Save(Product);
    return RedirectToPage("Index");                 // ← redirect after successful post
}
```

`RedirectToPage`/`RedirectToPage("Name", new { id })` issues a redirect. PRG avoids duplicate submissions and gives clean URLs — a fundamental web-form practice that Razor Pages makes natural.

---

## Anti-forgery (CSRF) protection

Razor Pages **automatically** validates anti-forgery tokens on POST (the `<form>` tag helper injects a hidden token; the framework checks it). This protects against Cross-Site Request Forgery out of the box — a key reason Razor Pages is safer than hand-rolled forms. Don't disable it without reason. (Security: [Ch10](../10-Identity/README.md).)

---

## Layouts, partials, and view components

```razor
@* _Layout.cshtml — shared shell *@
<html><body>@RenderBody() @await RenderSectionAsync("Scripts", required: false)</body></html>

@* a page sets its layout (usually in _ViewStart.cshtml) *@
@{ Layout = "_Layout"; }

<partial name="_ProductCard" model="product" />     @* reusable markup fragment *@
<vc:cart-summary />                                   @* a View Component (logic + view) *@
```

Layouts provide a shared page shell (`@RenderBody`); **partials** reuse markup; **View Components** encapsulate reusable UI with its own logic (like a mini-controller for a widget). These compose a UI from reusable pieces.

---

## Razor Pages vs MVC vs Blazor

| | Razor Pages | MVC | Blazor |
|---|---|---|---|
| Model | page-per-URL | controller/action | component-based |
| Best for | form-driven, page-centric server UI | complex apps, shared logic across views | rich interactive UI |
| Interactivity | server round-trips (forms) | server round-trips | interactive (SignalR/WASM) |
| Complexity | simplest for pages | more structure | richest, steeper |

Use **Razor Pages** for straightforward server-rendered, form-based pages (CRUD admin, content). Use **MVC** when many endpoints share logic or you need fine-grained controller control. Use **Blazor** ([Ch14](../14-Blazor/README.md)) for app-like interactivity. They can coexist.

---

## Common gotchas

### Forgetting `@page`

A `.cshtml` without the `@page` directive isn't a routable Razor Page (it's treated as an MVC view). The directive must be the first line.

### Not redirecting after POST

Rendering directly after a successful POST means a refresh re-submits. Use PRG (`RedirectToPage`).

### Forgetting `[BindProperty]`

A property not marked `[BindProperty]` won't be populated from the form on POST. Mark bound properties (and `SupportsGet = true` for GET binding).

### Disabling anti-forgery

Razor Pages' automatic CSRF protection is a feature — don't turn it off. Keep the `<form>` tag helper (which injects the token).

### Putting logic in the page

Like fat controllers, fat `PageModel`s are a smell. Delegate to injected services; keep handlers thin.

---

## Summary

- **Razor Pages** = page-centric server-rendered UI: a `.cshtml` (markup) + a `PageModel` (handlers/state), routed by **folder structure** + the `@page` directive.
- Handlers follow conventions: **`OnGet`/`OnPost`** (and **named handlers** `OnPost{Name}` via `asp-page-handler`); bind form/route data with **`[BindProperty]`**.
- **Tag Helpers** (`asp-for`, `asp-validation-for`, `asp-page`) generate model-bound, refactor-safe HTML; **anti-forgery (CSRF) protection is automatic** on POST.
- Follow **Post-Redirect-Get** (`RedirectToPage` after a successful POST) to avoid duplicate submissions; compose UI with **layouts, partials, and view components**.
- Choose Razor Pages for form-driven server UI, MVC for complex multi-endpoint apps, **Blazor** for interactivity. Keep `PageModel`s thin.

→ Next: [05-Middleware.md](05-Middleware.md)
