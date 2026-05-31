# Authentication and Authorization

## Identity in a component world

Blazor builds on ASP.NET Core's authentication/authorization ([Ch10](../10-Identity/README.md)) — the same `ClaimsPrincipal`, policies, and Identity system — but surfaces it through **components**: `AuthorizeView` to show/hide UI, `[Authorize]` to gate pages, and `AuthenticationStateProvider` to access the current user. The mechanics differ by render mode (where the user's identity is established and how it reaches the browser), which is the part that trips people up.

```razor
<AuthorizeView>
    <Authorized>Hello, @context.User.Identity!.Name!</Authorized>
    <NotAuthorized>Please log in.</NotAuthorized>
</AuthorizeView>
```

---

## `AuthenticationStateProvider` — who is the user?

The current user's identity is delivered as an `AuthenticationState` (wrapping a `ClaimsPrincipal`) via the **`AuthenticationStateProvider`** service, exposed to components as a **cascading `Task<AuthenticationState>`** ([03-Components.md](03-Components.md)). To read it in code:

```razor
@code {
    [CascadingParameter] private Task<AuthenticationState>? AuthState { get; set; }

    protected override async Task OnParametersSetAsync() {
        var user = (await AuthState!).User;
        if (user.Identity?.IsAuthenticated == true) { /* ... */ }
    }
}
```

To make this cascading value available, the app wraps its router in **`<CascadingAuthenticationState>`** (or uses `AddCascadingAuthenticationState()` in DI). The `AuthenticationStateProvider` differs by render mode: in Server it derives from the authenticated HTTP request/circuit; in WebAssembly a client-side provider obtains the user (e.g., from a token).

---

## `AuthorizeView` — conditional UI

`AuthorizeView` renders different content based on authentication/authorization state — the declarative way to show/hide UI:

```razor
<AuthorizeView Roles="Admin,Manager">
    <Authorized><AdminPanel /></Authorized>
    <NotAuthorized>You lack permission.</NotAuthorized>
</AuthorizeView>

<AuthorizeView Policy="CanEditOrders">
    <Authorized><EditButton /></Authorized>
</AuthorizeView>
```

It supports `Roles` and `Policy` filters (same policies as the rest of ASP.NET Core — [Ch10 §06](../10-Identity/06-Authorization.md)), and the `context` exposes the `User`. **This is UI gating, not security** — hiding a button doesn't protect the underlying operation; the server-side API/handler must still authorize the actual action.

---

## `[Authorize]` on pages and the `AuthorizeRouteView`

Gate whole routable pages with the `[Authorize]` attribute (works like in MVC — [Ch10 §06](../10-Identity/06-Authorization.md)):

```razor
@page "/admin"
@attribute [Authorize(Roles = "Admin")]
```

For this to take effect, the router uses **`AuthorizeRouteView`** (instead of `RouteView`), which renders an `Authorizing` view while auth state resolves and a `NotAuthorized` view for denied users (typically redirecting to login):

```razor
<AuthorizeRouteView RouteData="routeData" DefaultLayout="typeof(MainLayout)">
    <NotAuthorized>Redirecting to login…</NotAuthorized>
    <Authorizing>Checking permissions…</Authorizing>
</AuthorizeRouteView>
```

---

## Imperative authorization

For logic that isn't a simple show/hide, inject `IAuthorizationService` and check policies in code:

```razor
@inject IAuthorizationService Auth
@code {
    async Task<bool> CanEdit(Order o) {
        var user = (await AuthState!).User;
        var result = await Auth.AuthorizeAsync(user, o, "OrderOwnerPolicy");
        return result.Succeeded;
    }
}
```

This is the same resource-based authorization as elsewhere ([Ch10 §06](../10-Identity/06-Authorization.md)) — evaluate a policy against a specific resource (this order) and the user.

---

## Render-mode considerations

- **Interactive Server**: the user is authenticated by the normal ASP.NET Core pipeline (cookie/Identity) during the initial HTTP request; the circuit carries that identity. Auth "just works" via the server's authenticated context.
- **WebAssembly**: the app runs in the browser sandbox, so it authenticates via **tokens** (OIDC — [Ch10 §04](../10-Identity/04-OAuth-OIDC.md)) obtained from an identity provider, and calls APIs with a bearer token. A client-side `AuthenticationStateProvider` tracks the token's principal.
- **Auto/mixed**: components must obtain auth state through the cascading `Task<AuthenticationState>` so they work in both environments — don't assume server-only auth context.
- **Login UI**: Blazor Web App templates can use ASP.NET Core Identity with Razor/SSR pages for the actual login/logout flow, then surface the authenticated principal to interactive components.

The non-negotiable rule across all modes: **authorize on the server where the operation actually happens.** Component-level `AuthorizeView`/`[Authorize]` improves UX (and prevents accidental access), but real enforcement is at the API/data layer.

---

## Common gotchas

### Treating `AuthorizeView` as security

Hiding UI doesn't secure the operation — a user can still call the API. Enforce authorization server-side ([Ch10](../10-Identity/README.md)); component gating is UX.

### Missing `CascadingAuthenticationState`/`AuthorizeRouteView`

Without wiring the cascading auth state (and `AuthorizeRouteView` for `[Authorize]` pages), `AuthorizeView` and page `[Authorize]` won't work. Set them up in the router.

### Reading auth state in the constructor

The cascading `Task<AuthenticationState>` isn't ready in the ctor — await it in `OnInitializedAsync`/`OnParametersSetAsync`.

### Assuming server auth context in WebAssembly

WASM has no server HTTP context; it authenticates via tokens and a client-side provider. Get identity from the cascading auth state, not a server-only mechanism — crucial for Auto/mixed apps.

### Storing tokens insecurely (WASM)

Tokens in `localStorage` are exposed to XSS. Follow the framework's recommended token handling (e.g., BFF pattern, secure storage) — see [Ch10 §04](../10-Identity/04-OAuth-OIDC.md).

---

## Summary

- Blazor uses ASP.NET Core's auth system ([Ch10](../10-Identity/README.md)) through components: the current user is a **`ClaimsPrincipal`** delivered via **`AuthenticationStateProvider`** as a **cascading `Task<AuthenticationState>`** (enabled by `CascadingAuthenticationState`).
- **`AuthorizeView`** (with `Roles`/`Policy`) conditionally renders UI; **`[Authorize]`** on `@page` gates routes via **`AuthorizeRouteView`** (`Authorizing`/`NotAuthorized` views); **`IAuthorizationService`** does imperative/resource-based checks.
- **Render mode matters**: Interactive Server uses the authenticated HTTP request/circuit; **WebAssembly** authenticates via **tokens (OIDC)** with a client-side provider; Auto/mixed components must read auth via the cascading state to work in both.
- **Component-level auth is UX, not security** — always enforce authorization on the **server** where the operation runs; never store tokens insecurely on WASM.

→ Next: [12-Testing.md](12-Testing.md)
