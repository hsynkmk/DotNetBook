# Routing and Navigation

## `@page` — components as routable pages

A component becomes a routable page by declaring one or more route templates with the **`@page`** directive. The Blazor `Router` (in `Routes.razor`/`App.razor`) matches the browser URL to a component's route template and renders it:

```razor
@page "/products"
@page "/products/list"     @* a component can have multiple routes *@

<h1>Products</h1>
```

The router scans assemblies for `@page`-annotated components at startup and builds a route table. Unmatched URLs render the router's `NotFound` content.

---

## Route parameters

Route segments in `{braces}` become **route parameters**, bound to `[Parameter]` properties of the same name:

```razor
@page "/products/{Id:int}"
@page "/blog/{*slug}"            @* catch-all: matches the rest of the path *@

@code {
    [Parameter] public int Id { get; set; }
    [Parameter] public string? Slug { get; set; }
}
```

- **Route constraints** (`{Id:int}`, `:guid`, `:datetime`, `:bool`, etc.) restrict matches by type — a non-int `Id` won't match, falling through to another route or NotFound.
- **Optional parameters**: `{Id:int?}` matches with or without the segment.
- **Catch-all** (`{*slug}`) captures the remaining path including slashes — for hierarchical paths.

Type-constraining route parameters means your `[Parameter]` receives the right CLR type already parsed, and invalid URLs don't reach your component.

---

## Query strings

Bind query-string values with `[SupplyParameterFromQuery]`:

```razor
@page "/search"
@code {
    [Parameter, SupplyParameterFromQuery] public string? Query { get; set; }
    [Parameter, SupplyParameterFromQuery(Name = "page")] public int PageNumber { get; set; }
}
@* URL: /search?query=blazor&page=2 *@
```

Query parameters are read from the URL's query string (vs route parameters from the path). Changing a query parameter triggers `OnParametersSet` ([07-Lifecycle.md](07-Lifecycle.md)) without re-creating the component.

---

## `NavigationManager` — programmatic navigation

Inject **`NavigationManager`** to navigate in code, read the current URI, and observe navigation:

```razor
@inject NavigationManager Nav

@code {
    void GoToProduct(int id) => Nav.NavigateTo($"/products/{id}");
    void GoForceLoad() => Nav.NavigateTo("/x", forceLoad: true);   // full page reload
    void GoReplace() => Nav.NavigateTo("/y", replace: true);        // replace history entry

    protected override void OnInitialized() {
        Nav.LocationChanged += (_, e) => Console.WriteLine($"Now at {e.Location}");
    }
}
```

- `NavigateTo(uri)` — client-side navigation (no full reload) within the app.
- `forceLoad: true` — full browser navigation (reloads the app) — needed for non-Blazor URLs.
- `replace: true` — replaces the current history entry (no back-button entry).
- `Uri`/`BaseUri` — the current and base addresses; `ToAbsoluteUri`/`ToBaseRelativePath` for conversions.
- `LocationChanged` — observe navigation (remember to unsubscribe in `Dispose` — [07-Lifecycle.md](07-Lifecycle.md)).

`RegisterLocationChangingHandler` (or the `NavigationLock` component) lets you **intercept/cancel** navigation — e.g., warn about unsaved changes before leaving a form.

---

## `NavLink` — active-aware links

For navigation links, prefer the **`NavLink`** component over a raw `<a>`: it automatically applies an `active` CSS class when its href matches the current URL:

```razor
<NavLink href="products" Match="NavLinkMatch.Prefix">Products</NavLink>
<NavLink href="/" Match="NavLinkMatch.All">Home</NavLink>
```

`Match="All"` highlights only on an exact match (so "Home" isn't always active); `Match="Prefix"` highlights when the URL starts with the href (good for sections). This drives the "you are here" highlighting in navigation menus without manual class logic.

---

## Layouts

Pages render inside a **layout** — a component deriving from `LayoutComponentBase` with a `@Body` placeholder. Assign layouts via `@layout` (per page) or a `_Imports.razor`/router default (whole folder/app):

```razor
@* MainLayout.razor *@
@inherits LayoutComponentBase
<nav>...</nav>
<main>@Body</main>      @* the routed page renders here *@
```

Layouts can nest (a layout can itself specify a parent layout), letting you build shared chrome (header/nav/footer) once and compose it.

---

## Common gotchas

### Not unsubscribing from `LocationChanged`

Subscribing to `Nav.LocationChanged` (or other events) without unsubscribing in `Dispose` leaks the component — especially costly in Interactive Server. Implement `IDisposable` and detach ([07-Lifecycle.md](07-Lifecycle.md)).

### Using `<a>` instead of `NavLink` for in-app links

A raw `<a>` works but lacks active-state styling and (depending on setup) may trigger a full reload. Use `NavLink` for in-app navigation, and reserve `forceLoad` for external/non-Blazor URLs.

### Route constraint mismatch silently 404s

`/products/{Id:int}` won't match `/products/abc` — it falls through to NotFound. If you expected a match, check the constraint and the URL type.

### Expecting navigation to re-create the component

Navigating between the *same* component with different parameters (e.g., `/products/1` → `/products/2`) reuses the instance and runs `OnParametersSet`, not `OnInitialized`. Re-fetch data in `OnParametersSetAsync`, not only `OnInitializedAsync`.

### Forgetting query-parameter changes don't re-init

Changing `?page=2` updates parameters and fires `OnParametersSet`, not a fresh init — handle data reloads there.

---

## Summary

- **`@page "/route"`** makes a component routable; the **`Router`** builds a route table from `@page` annotations and renders the match (or `NotFound`).
- **Route parameters** (`{Id:int}`) bind to `[Parameter]` properties with **constraints** (type-safe), **optional** (`{Id:int?}`), and **catch-all** (`{*slug}`) forms; **query strings** bind via `[SupplyParameterFromQuery]`.
- **`NavigationManager`** navigates in code (`NavigateTo`, `forceLoad`, `replace`), exposes the current URI, raises `LocationChanged`, and can **intercept** navigation (unsaved-changes guards) — unsubscribe on dispose.
- **`NavLink`** gives active-aware links (`Match` All/Prefix); pages render inside **layouts** (`LayoutComponentBase` + `@Body`), which can nest.
- Navigating between the same component with new parameters runs **`OnParametersSet`**, not `OnInitialized` — reload data accordingly.

→ Next: [07-Lifecycle.md](07-Lifecycle.md)
