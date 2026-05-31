# Authentication vs Authorization

## The two halves of access control

Security in a web app has two distinct questions, and conflating them is a common source of bugs and vulnerabilities:

- **Authentication (AuthN)** — *"Who are you?"* Establishing the caller's identity (verifying credentials, validating a token, reading a cookie).
- **Authorization (AuthZ)** — *"What are you allowed to do?"* Deciding whether the (already-authenticated) caller may perform an action or access a resource.

```
Request → [ Authentication ]  → establishes WHO (a ClaimsPrincipal: "alice, role=admin")
       → [ Authorization ]    → decides IF alice (with role=admin) may do this
       → endpoint
```

Authentication comes **first** (you can't decide what someone can do until you know who they are), then authorization. In ASP.NET Core this maps to two middleware in order: `UseAuthentication()` then `UseAuthorization()` ([Ch04 §05](../04-AspNetCore/05-Middleware.md)).

---

## Authentication — establishing identity

Authentication produces a **`ClaimsPrincipal`** (the `HttpContext.User`) — a set of **claims** (statements about the user: their id, name, roles, email, etc. — [07-Claims.md](07-Claims.md)). How identity is established varies by **scheme**:

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)   // default scheme
    .AddJwtBearer(...)        // validate a JWT bearer token ([03-JWT.md])
    .AddCookie(...);          // read an auth cookie ([05-Cookies.md])

var app = builder.Build();
app.UseAuthentication();      // runs the scheme → populates HttpContext.User
```

Common schemes:
- **JWT bearer** ([03-JWT.md](03-JWT.md)) — validate a signed token in the `Authorization: Bearer` header. The default for APIs and SPAs.
- **Cookie** ([05-Cookies.md](05-Cookies.md)) — a signed cookie holds the identity. The default for server-rendered web apps.
- **OAuth/OIDC** ([04-OAuth-OIDC.md](04-OAuth-OIDC.md)) — delegate login to an identity provider (Google, Entra ID, Auth0).
- **Certificate** — mutual-TLS client certificates.

After `UseAuthentication`, `HttpContext.User` is a `ClaimsPrincipal` — authenticated (with claims) or anonymous (`User.Identity.IsAuthenticated == false`).

---

## Authorization — deciding access

Authorization uses the established identity (the `ClaimsPrincipal` and its claims) to decide access. ASP.NET Core offers several models ([06-Authorization.md](06-Authorization.md)):

```csharp
// Simplest — require any authenticated user
app.MapGet("/profile", () => ...).RequireAuthorization();

// Role-based
[Authorize(Roles = "Admin")]
public IActionResult AdminPanel() => ...;

// Policy-based (the recommended, flexible model)
builder.Services.AddAuthorization(o =>
    o.AddPolicy("Over18", p => p.RequireClaim("age_verified", "true")));
app.MapGet("/restricted", () => ...).RequireAuthorization("Over18");

app.UseAuthorization();   // enforces [Authorize]/RequireAuthorization, using HttpContext.User
```

Models: **authenticated-only** (`RequireAuthorization()`), **role-based** (`[Authorize(Roles=...)]`), **claims/policy-based** (the flexible default — requirements + handlers), and **resource-based** (can *this* user edit *this specific* document?). Policy-based is the recommended approach — it's testable, composable, and decouples "what's required" from "who has it." Covered in [06-Authorization.md](06-Authorization.md).

---

## The pipeline order (critical)

```csharp
app.UseRouting();          // 1. match the endpoint (so auth can read its [Authorize] metadata)
app.UseAuthentication();   // 2. WHO are you? → populate HttpContext.User
app.UseAuthorization();    // 3. are you ALLOWED? → enforce the endpoint's requirements
app.MapControllers();      // 4. the endpoint
```

The order is mandatory: **routing → authentication → authorization → endpoint**. Authentication must run before authorization (you need the identity to authorize). Routing must run before authentication so the auth/authz middleware can see the matched endpoint's `[Authorize]` metadata. Getting this wrong (e.g., authorization before authentication) is a classic security bug — endpoints end up unprotected or auth fails silently ([Ch04 §05](../04-AspNetCore/05-Middleware.md)).

---

## 401 vs 403 — the precise distinction

The status codes reflect the AuthN/AuthZ split exactly:

| Code | Meaning | Cause |
|---|---|---|
| **401 Unauthorized** | *not authenticated* (despite the name) | no/invalid credentials — "I don't know who you are" |
| **403 Forbidden** | *authenticated but not authorized* | known user lacks permission — "I know who you are, but you can't" |

- **401** ("Unauthorized" — a misnomer; it really means *un-authenticated*) → the caller should authenticate (log in, provide a valid token).
- **403** → the caller is authenticated but lacks the required permission; logging in again won't help.

ASP.NET Core returns these automatically: an unauthenticated request to a protected endpoint → 401; an authenticated request failing an authorization policy → 403. Returning the wrong one (403 for missing auth, or 401 for insufficient permission) confuses clients about whether to re-authenticate.

---

## Common gotchas

### Confusing AuthN and AuthZ

They're distinct: authentication establishes *who*, authorization decides *what they can do*. A valid token (authenticated) doesn't mean the user is allowed to do everything (authorization is separate).

### Wrong middleware order

`UseAuthorization` before `UseAuthentication`, or both before `UseRouting`, breaks access control. The order is routing → authentication → authorization → endpoint.

### Returning the wrong status code

401 = not authenticated (re-authenticate); 403 = authenticated but forbidden (don't bother re-authenticating). Don't swap them.

### Authorizing on un-validated data

Authorization decisions must use the **validated** identity (`HttpContext.User` after authentication), not raw headers/claims the client could forge. Trust only what the authentication scheme verified.

### Assuming authentication implies authorization

`RequireAuthorization()` only checks the user is authenticated — it doesn't check permissions. Add role/policy requirements for actual access control.

### Forgetting to protect endpoints

An endpoint without `[Authorize]`/`RequireAuthorization` is **public** by default. Be deliberate; consider a fallback policy requiring auth by default and opting specific endpoints out (`[AllowAnonymous]`).

---

## Summary

- Security has two halves: **authentication** ("who are you?" — establishes a `ClaimsPrincipal`) and **authorization** ("what can you do?" — decides access using that identity). Don't conflate them.
- Authentication uses a **scheme** (JWT bearer, cookie, OAuth/OIDC, certificate) to populate `HttpContext.User`; authorization uses **policies/roles/claims** to enforce access.
- Pipeline order is mandatory: **routing → `UseAuthentication` → `UseAuthorization` → endpoint** (auth before authz; routing first so metadata is visible).
- **401** = not authenticated (re-authenticate); **403** = authenticated but forbidden. Return the right one.
- Authorize only on the **validated** identity; endpoints are **public by default** — protect them deliberately (a fallback "require auth" policy + `[AllowAnonymous]` opt-outs is a safe pattern).

→ Next: [02-AspNetIdentity.md](02-AspNetIdentity.md)
