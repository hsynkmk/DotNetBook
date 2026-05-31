# Authorization

## Deciding what an authenticated user can do

Authorization runs **after** authentication ([01-AuthN-vs-AuthZ.md](01-AuthN-vs-AuthZ.md)) and decides whether the established identity may perform an action or access a resource. ASP.NET Core offers a progression of models — from simple "must be logged in" to fine-grained, **policy-based** authorization with custom requirements and handlers. Policy-based is the recommended, flexible default.

```csharp
app.UseAuthentication();
app.UseAuthorization();

// Require authentication
app.MapGet("/profile", () => ...).RequireAuthorization();

// Role-based
app.MapGet("/admin", () => ...).RequireAuthorization(p => p.RequireRole("Admin"));

// Policy-based
builder.Services.AddAuthorization(o =>
    o.AddPolicy("CanEditOrders", p => p.RequireClaim("permission", "orders:write")));
app.MapPut("/orders/{id}", (...) => ...).RequireAuthorization("CanEditOrders");
```

---

## The models, from simple to powerful

### Authenticated-only

```csharp
[Authorize]                                       // MVC/Razor
app.MapGet("/x", () => ...).RequireAuthorization(); // Minimal API
```

Just requires `User.Identity.IsAuthenticated`. The baseline.

### Role-based

```csharp
[Authorize(Roles = "Admin,Manager")]   // user must have one of these roles
```

Coarse-grained: checks for a role claim. Simple, but roles get unwieldy for granular permissions (a role per capability) — prefer claims/policies beyond coarse grouping.

### Claims / policy-based (recommended)

```csharp
builder.Services.AddAuthorization(o => {
    o.AddPolicy("Over18", p => p.RequireClaim("age_verified", "true"));
    o.AddPolicy("EngineeringAdmin", p => p
        .RequireRole("Admin")
        .RequireClaim("department", "Engineering"));
});
[Authorize(Policy = "Over18")]
```

A **policy** is a named set of **requirements**. Policy-based authorization decouples *what's required* (the policy) from *where it's enforced* (the endpoint) — testable, composable, and the recommended model for anything beyond trivial cases.

### Resource-based

```csharp
// "Can THIS user edit THIS specific document?" — needs the resource instance
var result = await authService.AuthorizeAsync(User, document, "EditPolicy");
if (!result.Succeeded) return Results.Forbid();
```

When the decision depends on the **specific resource** (ownership: can Alice edit *this* document she owns?), use **resource-based** authorization via `IAuthorizationService.AuthorizeAsync(user, resource, policy)` — because attribute-based authorization can't see the resource instance.

---

## Custom requirements and handlers

For logic beyond claims/roles, define a **requirement** (the rule) and a **handler** (the logic that evaluates it):

```csharp
// Requirement — a marker carrying parameters
public class MinimumAgeRequirement(int age) : IAuthorizationRequirement { public int Age { get; } = age; }

// Handler — evaluates the requirement against the user (and optionally a resource)
public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement> {
    protected override Task HandleRequirementAsync(AuthorizationHandlerContext ctx, MinimumAgeRequirement req) {
        var dob = ctx.User.FindFirst("date_of_birth")?.Value;
        if (dob is not null && DateTime.Parse(dob) <= DateTime.UtcNow.AddYears(-req.Age))
            ctx.Succeed(req);          // requirement met
        return Task.CompletedTask;     // not calling Succeed = not met
    }
}

builder.Services.AddAuthorization(o => o.AddPolicy("AtLeast21", p => p.Requirements.Add(new MinimumAgeRequirement(21))));
builder.Services.AddSingleton<IAuthorizationHandler, MinimumAgeHandler>();
```

A policy can have multiple requirements (all must succeed); a requirement can have multiple handlers (any succeeding satisfies it). Handlers can inject services (DB, etc.) and access the resource (resource-based). This is the extensibility point for arbitrary authorization logic.

---

## Applying authorization

```csharp
// MVC — attributes (controller or action)
[Authorize(Policy = "CanEditOrders")]
public IActionResult Edit(int id) => ...;

// Minimal API — fluent
app.MapPut("/orders/{id}", (...) => ...).RequireAuthorization("CanEditOrders");

// Whole group
app.MapGroup("/admin").RequireAuthorization("AdminPolicy");

// Opt out
[AllowAnonymous]   // override a controller-level [Authorize] / fallback policy
```

Apply at the endpoint, controller, group, or globally. **`[AllowAnonymous]`** overrides authorization for specific endpoints.

---

## Fallback and default policies (secure by default)

Endpoints are **public by default** ([01-AuthN-vs-AuthZ.md](01-AuthN-vs-AuthZ.md)) — a forgotten `[Authorize]` leaves an endpoint open. A **fallback policy** flips this to secure-by-default:

```csharp
builder.Services.AddAuthorization(o => {
    o.FallbackPolicy = new AuthorizationPolicyBuilder().RequireAuthenticatedUser().Build();
    // Now ALL endpoints require authentication unless they opt out with [AllowAnonymous]
});
```

With a fallback policy requiring authentication, every endpoint is protected by default; you explicitly `[AllowAnonymous]` the public ones (login, health). This "deny by default" posture prevents accidentally-public endpoints — a recommended security practice. (Distinguish from `DefaultPolicy`, used when `[Authorize]` has no named policy.)

---

## Common gotchas

### Forgetting to protect an endpoint

Endpoints are public by default — a missing `[Authorize]` is an open door. Use a **fallback policy** (deny by default) + `[AllowAnonymous]` opt-outs.

### Roles for fine-grained permissions

A role per capability becomes unmanageable. Use **claims + policies** for granular permissions (e.g., a `permission` claim per action).

### Resource decisions via attributes

`[Authorize]` can't see the resource instance, so "can edit *this* document" can't be an attribute. Use **resource-based** authorization (`IAuthorizationService.AuthorizeAsync(user, resource, policy)`).

### Authorization logic in the endpoint body

Scattering `if (User.IsInRole(...))` checks through handlers is unmaintainable and untestable. Encapsulate rules in **policies/requirements/handlers**.

### Not registering the handler

A policy with a custom requirement does nothing without its `IAuthorizationHandler` registered in DI. Register handlers.

### Trusting client-supplied claims

Authorize on claims the **authentication scheme validated** (in `HttpContext.User`), not raw request data the client could forge.

---

## Summary

- **Authorization** decides what an authenticated user can do; models progress: **authenticated-only** → **role-based** (coarse) → **claims/policy-based** (recommended) → **resource-based** (per-instance ownership).
- A **policy** = a named set of **requirements**; **requirements + handlers** encapsulate arbitrary logic (handlers can inject services and inspect the resource) — testable and composable, decoupling *what's required* from *where it's enforced*.
- Apply via `[Authorize(Policy=...)]` / `RequireAuthorization(...)` at endpoint/controller/group/global level; **`[AllowAnonymous]`** opts out.
- Use **resource-based** authorization (`IAuthorizationService.AuthorizeAsync`) when the decision depends on the specific resource.
- Endpoints are **public by default** — set a **fallback policy** (deny by default) for secure-by-default; prefer **claims/policies** over roles; authorize only on **validated** claims.

→ Next: [07-Claims.md](07-Claims.md)
