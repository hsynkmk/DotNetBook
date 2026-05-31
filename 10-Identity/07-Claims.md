# Claims

## The identity model

A **claim** is a statement about a user — `name = Alice`, `email = alice@x.com`, `role = Admin`, `permission = orders:write`. The authenticated user is a **`ClaimsPrincipal`** (`HttpContext.User`) containing one or more **`ClaimsIdentity`** objects, each holding **`Claim`**s. This claims model is the universal currency of .NET auth: authentication produces claims, authorization consumes them.

```csharp
ClaimsPrincipal user = httpContext.User;
string? name = user.FindFirst(ClaimTypes.Name)?.Value;
string? email = user.FindFirst(ClaimTypes.Email)?.Value;
bool isAdmin = user.IsInRole("Admin");                    // role is a claim of type ClaimTypes.Role
var permissions = user.FindAll("permission").Select(c => c.Value);
bool authenticated = user.Identity?.IsAuthenticated == true;
```

Whatever the scheme (cookie, JWT, OIDC), the result is the same: a `ClaimsPrincipal` of claims that downstream code reads.

---

## ClaimsPrincipal → ClaimsIdentity → Claim

```
ClaimsPrincipal (the user)
 ├── ClaimsIdentity (from the cookie/JWT/OIDC scheme; AuthenticationType = "Bearer"/"Cookies"/...)
 │    ├── Claim { Type = "name",       Value = "Alice" }
 │    ├── Claim { Type = "role",       Value = "Admin" }
 │    └── Claim { Type = "permission", Value = "orders:write" }
 └── (possibly more identities, e.g., from multiple schemes)
```

- **`Claim`** — a `Type`/`Value` pair (e.g., `("email", "alice@x.com")`), plus issuer info.
- **`ClaimsIdentity`** — a set of claims from one authentication source, with an `AuthenticationType` (and `IsAuthenticated` is true iff it has an auth type).
- **`ClaimsPrincipal`** — the user, holding one or more identities (usually one).

A principal can hold **multiple identities** (e.g., authenticated by two schemes), and `FindFirst`/`FindAll`/`IsInRole` search across all of them.

---

## Building a principal (issuing claims)

When you sign a user in (cookie or token), you build the claims:

```csharp
var claims = new List<Claim> {
    new(ClaimTypes.NameIdentifier, user.Id),       // the user id (sub)
    new(ClaimTypes.Name, user.UserName),
    new(ClaimTypes.Email, user.Email),
    new(ClaimTypes.Role, "Admin"),                  // a role
    new("permission", "orders:write"),              // a custom claim
    new("tenant_id", user.TenantId.ToString())      // app-specific
};
var identity = new ClaimsIdentity(claims, authenticationType: "Cookies");
var principal = new ClaimsPrincipal(identity);
await HttpContext.SignInAsync(principal);            // cookie scheme
```

For JWTs, these become the token's payload claims ([03-JWT.md](03-JWT.md)); for cookies, they're encrypted into the cookie ([05-Cookies.md](05-Cookies.md)). Authorization policies ([06-Authorization.md](06-Authorization.md)) then check these claims.

---

## Standard claim types & mapping from IdPs

Claim types are URIs/strings; .NET defines standard ones (`ClaimTypes.*`), and OIDC/JWT use short standard names (`sub`, `name`, `email`, `role`). When authenticating via an external IdP, the IdP's claim names may differ from what your app expects — you **map** them:

```csharp
// OIDC: map IdP claim names to your app's expectations
.AddOpenIdConnect(o => {
    o.TokenValidationParameters.NameClaimType = "name";
    o.TokenValidationParameters.RoleClaimType = "roles";   // the IdP uses "roles", not the .NET default
    o.ClaimActions.MapJsonKey("tenant_id", "tid");          // map IdP "tid" → your "tenant_id"
});
```

Different IdPs emit different claim names/shapes; configure `NameClaimType`/`RoleClaimType` and `ClaimActions` so `User.Identity.Name`, `IsInRole`, and your policies work. (Note: .NET historically remapped some JWT claim types automatically — you can disable that mapping for predictability with `MapInboundClaims = false`.)

---

## Claims transformation

Sometimes you need to **add or modify claims** after authentication — e.g., enrich the principal with permissions looked up from a database, or derive claims from existing ones. Use **`IClaimsTransformation`**:

```csharp
public class PermissionClaimsTransformation(IPermissionStore store) : IClaimsTransformation {
    public async Task<ClaimsPrincipal> TransformAsync(ClaimsPrincipal principal) {
        var identity = (ClaimsIdentity)principal.Identity!;
        if (identity.HasClaim(c => c.Type == "permissions_loaded")) return principal;  // idempotent guard

        var userId = principal.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        foreach (var perm in await store.GetPermissionsAsync(userId!))
            identity.AddClaim(new Claim("permission", perm));
        identity.AddClaim(new Claim("permissions_loaded", "true"));
        return principal;
    }
}
builder.Services.AddScoped<IClaimsTransformation, PermissionClaimsTransformation>();
```

`IClaimsTransformation.TransformAsync` runs **on every request** after authentication — so make it **idempotent** (guard against adding duplicate claims) and **cheap** (it's per-request; cache lookups). Use it to enrich the principal with data not in the token/cookie (DB-sourced permissions, computed claims) rather than bloating the token itself.

---

## Claims-based authorization (recap)

Claims are the input to authorization ([06-Authorization.md](06-Authorization.md)):

```csharp
o.AddPolicy("CanWriteOrders", p => p.RequireClaim("permission", "orders:write"));
o.AddPolicy("EngTenant", p => p.RequireClaim("tenant_id").RequireClaim("department", "Engineering"));
```

Prefer **fine-grained permission claims** (`permission: orders:write`) over coarse roles for flexible authorization — a user's capabilities are expressed as claims, and policies check them.

---

## Common gotchas

### Expecting IdP claim names to match .NET defaults

External IdPs emit their own claim names (`roles` vs `role`, `tid` vs `tenant_id`). Map them (`NameClaimType`/`RoleClaimType`/`ClaimActions`) so `IsInRole`/policies work.

### Non-idempotent claims transformation

`IClaimsTransformation` runs every request — adding a claim without guarding leads to **duplicate** claims accumulating. Guard with a marker claim; keep it cheap (it's per-request).

### Bloating the token with too many claims

Every claim is carried in the cookie/JWT (size, every request). Put essentials in the token; load extra data (permissions) via claims transformation or server-side lookup keyed by the user id.

### Putting sensitive data in claims

JWT claims are client-readable ([03-JWT.md](03-JWT.md)); don't put secrets/sensitive PII in claims. Cookies encrypt claims, but still keep them minimal.

### Trusting unvalidated claims

Only trust claims from `HttpContext.User` (produced by a validated authentication scheme), not raw request data. The scheme's validation is what makes claims trustworthy.

### Using roles where claims fit better

Roles are just claims of type `ClaimTypes.Role`; for granular permissions, custom claims + policies are more flexible than proliferating roles.

---

## Summary

- A **claim** is a `Type`/`Value` statement about the user; the authenticated user is a **`ClaimsPrincipal`** holding **`ClaimsIdentity`**(s) of **`Claim`**s — the universal .NET identity model (authentication produces claims, authorization consumes them).
- Build claims at sign-in (cookie/JWT payload); read them via `FindFirst`/`FindAll`/`IsInRole` on `HttpContext.User`.
- **Map** external IdP claim names to your app's expectations (`NameClaimType`/`RoleClaimType`/`ClaimActions`); consider `MapInboundClaims = false` for predictability.
- Use **`IClaimsTransformation`** to enrich the principal with DB-sourced/computed claims after authentication — make it **idempotent** and cheap (runs per request).
- Prefer **fine-grained permission claims + policies** over coarse roles; keep tokens lean (load extra data via transformation/lookup); never put secrets in claims; trust only validated claims.

→ Next: [08-DataProtection.md](08-DataProtection.md)
