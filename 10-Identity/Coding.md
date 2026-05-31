# Chapter 10 — Identity & Security — Coding Problems

JWT issuance and validation, policy-based authorization, claims transformation, and securing an API correctly.

---

## Problem 1: Validate JWTs from an IdP (API side)

Configure an API to accept and validate bearer tokens from an external IdP.

<details><summary>Solution</summary>

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(o => {
        o.Authority = "https://login.example.com";   // discovers endpoints + public keys (JWKS)
        o.Audience = "api://my-api";                   // validate the token is meant for THIS API
        o.TokenValidationParameters = new() {
            ValidateIssuerSigningKey = true,           // verify signature (fetched public key)
            ValidateLifetime = true,                   // reject expired
            ValidateIssuer = true,
            ValidateAudience = true,
            ClockSkew = TimeSpan.FromSeconds(30)
        };
        o.MapInboundClaims = false;                    // predictable claim names (no auto-remap)
    });
builder.Services.AddAuthorization();

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
app.MapGet("/secure", (ClaimsPrincipal user) => $"Hello {user.FindFirst("name")?.Value}")
   .RequireAuthorization();
```

No shared secret — the API fetches the IdP's public keys via discovery and validates signature/expiry/issuer/audience. ([03-JWT.md](03-JWT.md), [04-OAuth-OIDC.md](04-OAuth-OIDC.md).)

</details>

---

## Problem 2: Issue a JWT (token endpoint)

Issue a short-lived access token after validating credentials.

<details><summary>Solution</summary>

```csharp
public string CreateAccessToken(AppUser user, IEnumerable<string> permissions) {
    var claims = new List<Claim> {
        new(JwtRegisteredClaimNames.Sub, user.Id),
        new(JwtRegisteredClaimNames.Email, user.Email),
        new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
    };
    claims.AddRange(permissions.Select(p => new Claim("permission", p)));

    var key = new SigningCredentials(_signingKey, SecurityAlgorithms.RsaSha256);   // asymmetric
    var token = new JwtSecurityToken(
        issuer: "https://login.example.com", audience: "api://my-api",
        claims: claims,
        expires: DateTime.UtcNow.AddMinutes(15),     // SHORT-lived (pair with a refresh token)
        signingCredentials: key);
    return new JwtSecurityTokenHandler().WriteToken(token);
}
```

Short expiry (15 min) + asymmetric signing (RS256 — validators use the public key); include only necessary claims (no secrets). Issue a separate stored refresh token for renewal. (In practice, prefer Identity's `MapIdentityApi` or an IdP — [02-AspNetIdentity.md](02-AspNetIdentity.md).) ([03-JWT.md](03-JWT.md).)

</details>

---

## Problem 3: Policy-based authorization

Require a specific permission claim for an endpoint.

<details><summary>Solution</summary>

```csharp
builder.Services.AddAuthorization(o => {
    o.AddPolicy("CanWriteOrders", p => p.RequireClaim("permission", "orders:write"));
    o.AddPolicy("AdminOnly", p => p.RequireRole("Admin"));
});

app.MapPost("/orders", (CreateOrder dto, IOrderService svc) => svc.Create(dto))
   .RequireAuthorization("CanWriteOrders");

// MVC equivalent:
[Authorize(Policy = "CanWriteOrders")]
public IActionResult Create(CreateOrder dto) => ...;
```

The policy decouples *what's required* (an `orders:write` permission claim) from *where it's enforced* (the endpoint) — testable and reusable. ([06-Authorization.md](06-Authorization.md).)

</details>

---

## Problem 4: Custom authorization requirement + handler

Authorize only users at least 21, based on a date-of-birth claim.

<details><summary>Solution</summary>

```csharp
public class MinimumAgeRequirement(int age) : IAuthorizationRequirement { public int Age { get; } = age; }

public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement> {
    protected override Task HandleRequirementAsync(AuthorizationHandlerContext ctx, MinimumAgeRequirement req) {
        var dobClaim = ctx.User.FindFirst("date_of_birth")?.Value;
        if (dobClaim is not null && DateTime.TryParse(dobClaim, out var dob)
            && dob <= DateTime.UtcNow.AddYears(-req.Age))
            ctx.Succeed(req);            // requirement met
        return Task.CompletedTask;       // not calling Succeed = denied
    }
}

builder.Services.AddAuthorization(o =>
    o.AddPolicy("AtLeast21", p => p.Requirements.Add(new MinimumAgeRequirement(21))));
builder.Services.AddSingleton<IAuthorizationHandler, MinimumAgeHandler>();   // MUST register the handler

app.MapGet("/bar", () => "🍺").RequireAuthorization("AtLeast21");
```

Requirement (rule + params) + handler (logic) + DI registration. The handler can inject services and inspect the resource for resource-based checks. ([06-Authorization.md](06-Authorization.md).)

</details>

---

## Problem 5: Resource-based authorization (ownership)

Allow editing a document only if the user owns it.

<details><summary>Solution</summary>

```csharp
public class DocumentOwnerHandler : AuthorizationHandler<OperationAuthorizationRequirement, Document> {
    protected override Task HandleRequirementAsync(AuthorizationHandlerContext ctx,
        OperationAuthorizationRequirement req, Document resource) {
        if (req.Name == "Edit" && resource.OwnerId == ctx.User.FindFirst(ClaimTypes.NameIdentifier)?.Value)
            ctx.Succeed(req);
        return Task.CompletedTask;
    }
}
builder.Services.AddSingleton<IAuthorizationHandler, DocumentOwnerHandler>();

app.MapPut("/documents/{id}", async (int id, IAuthorizationService auth, ClaimsPrincipal user, IDocStore store) => {
    var doc = await store.GetAsync(id);
    if (doc is null) return Results.NotFound();
    var result = await auth.AuthorizeAsync(user, doc, new OperationAuthorizationRequirement { Name = "Edit" });
    if (!result.Succeeded) return Results.Forbid();   // 403 — authenticated but not the owner
    // ... edit ...
    return Results.NoContent();
});
```

Ownership depends on the **specific resource**, so it can't be an attribute — use `IAuthorizationService.AuthorizeAsync(user, resource, requirement)`. ([06-Authorization.md](06-Authorization.md).)

</details>

---

## Problem 6: Enrich claims with IClaimsTransformation

Load a user's permissions from the DB onto the principal each request (idempotently).

<details><summary>Solution</summary>

```csharp
public class PermissionClaimsTransformation(IPermissionStore store) : IClaimsTransformation {
    public async Task<ClaimsPrincipal> TransformAsync(ClaimsPrincipal principal) {
        var identity = (ClaimsIdentity)principal.Identity!;
        if (identity.HasClaim(c => c.Type == "perms_loaded")) return principal;   // IDEMPOTENT guard

        var userId = principal.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        if (userId is not null)
            foreach (var perm in await store.GetPermissionsAsync(userId))
                identity.AddClaim(new Claim("permission", perm));
        identity.AddClaim(new Claim("perms_loaded", "true"));
        return principal;
    }
}
builder.Services.AddScoped<IClaimsTransformation, PermissionClaimsTransformation>();
```

Runs every request after authentication — the guard prevents duplicate claims; keep the lookup cheap (cache). Keeps permissions out of the token (lean tokens). ([07-Claims.md](07-Claims.md).)

</details>

---

## Problem 7: Protect a short token with Data Protection

Generate a tamper-proof, purpose-scoped token (e.g., an unsubscribe link).

<details><summary>Solution</summary>

```csharp
public class UnsubscribeTokenService(IDataProtectionProvider provider) {
    private readonly IDataProtector _protector = provider.CreateProtector("MyApp.Unsubscribe.v1");  // purpose

    public string Create(string email) =>
        _protector.Protect($"{email}|{DateTimeOffset.UtcNow:O}");

    public string? Validate(string token, TimeSpan maxAge) {
        try {
            var parts = _protector.Unprotect(token).Split('|');   // throws if tampered/wrong purpose
            if (DateTimeOffset.UtcNow - DateTimeOffset.Parse(parts[1]) > maxAge) return null;  // expired
            return parts[0];
        } catch (CryptographicException) { return null; }   // tampered/invalid
    }
}

// Multi-instance: share the key ring!
builder.Services.AddDataProtection()
    .PersistKeysToAzureBlobStorage(blobUri)
    .ProtectKeysWithAzureKeyVault(keyId, credential);
```

Purpose-scoped (`Unsubscribe.v1`) so the token can't be repurposed; `Unprotect` throws on tampering. Persist the key ring to a shared store (encrypted at rest) for multi-instance. ([08-DataProtection.md](08-DataProtection.md).)

</details>

---

## Problem 8: Secure-by-default API + correct status codes

Make all endpoints require auth by default, opt out the public ones.

<details><summary>Solution</summary>

```csharp
builder.Services.AddAuthorization(o => {
    o.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser().Build();          // DENY BY DEFAULT — every endpoint needs auth
});

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();

app.MapGet("/health", () => "ok").AllowAnonymous();   // explicit public opt-out
app.MapPost("/login", LoginHandler).AllowAnonymous();
app.MapGet("/orders", GetOrders);                      // protected by the fallback policy (no attribute needed)
app.MapPost("/orders", CreateOrder).RequireAuthorization("CanWriteOrders");
```

The fallback policy flips the default to secure (no forgotten-`[Authorize]` holes); public endpoints opt out with `AllowAnonymous`. The framework returns 401 (unauthenticated) / 403 (authenticated but unauthorized) automatically. ([01-AuthN-vs-AuthZ.md](01-AuthN-vs-AuthZ.md), [06-Authorization.md](06-Authorization.md).)

</details>

---

## Problem 9: Spot the security mistakes

Find the issues in this configuration.

```csharp
.AddJwtBearer(o => {
    o.TokenValidationParameters = new() {
        ValidateIssuerSigningKey = false,    // (1)
        ValidateAudience = false,            // (2)
        ValidateLifetime = false             // (3)
    };
});
var token = $"user={username}";              // (4) home-made "token"
if (providedHash == storedHash) { }          // (5)
```

<details><summary>Solution</summary>

1. **`ValidateIssuerSigningKey = false`** — anyone can forge a token (no signature check). **Must be true.**
2. **`ValidateAudience = false`** — a token issued for another service is accepted here. **Validate the audience.**
3. **`ValidateLifetime = false`** — expired (possibly stolen long ago) tokens are accepted. **Validate lifetime.**
4. **Home-made plaintext "token"** — trivially forgeable, no signature, no expiry. Use a real signed JWT (or Data Protection / Identity).
5. **`==` for comparing a hash/secret** — leaks timing → byte-by-byte forgery. Use `CryptographicOperations.FixedTimeEquals`. (And don't compare password hashes yourself — use Identity.)

Fixed: validate signature/audience/lifetime, use real JWTs from Identity/an IdP, and `FixedTimeEquals` for any secret comparison. ([03-JWT.md](03-JWT.md), [10-Cryptography.md](10-Cryptography.md).)

</details>

---

## Problem 10: Choose the auth approach

For each, pick authentication + carrier and justify.
1. A server-rendered admin portal (Razor Pages), staff log in with username/password.
2. A public REST API consumed by mobile apps and third-party integrations.
3. A SPA + API where you want SSO with the company's Entra ID and no tokens in browser storage.
4. Service A calling service B internally, no user involved.

<details><summary>Solution</summary>

1. **ASP.NET Core Identity + cookie auth** — server-rendered, same-origin browser; cookies are simple, revocable, with built-in antiforgery. Identity owns the staff accounts. ([02](02-AspNetIdentity.md)/[05](05-Cookies.md).)
2. **JWT bearer tokens** (issued by Identity/`MapIdentityApi` or an IdP) — APIs/mobile use the `Authorization` header; stateless, not CSRF-vulnerable, natural for third parties. ([03](03-JWT.md).)
3. **OIDC with Entra ID** (Authorization Code + PKCE); use **cookie** auth between SPA and its backend (avoids storing tokens in JS-accessible storage, an XSS concern) with antiforgery, or a BFF pattern. SSO + outsourced auth. ([04](04-OAuth-OIDC.md)/[05](05-Cookies.md).)
4. **OAuth client-credentials** (or **managed identity** if both on Azure — no secret) — service-to-service, no user; the caller gets an access token and B validates it. ([04](04-OAuth-OIDC.md), [Ch09 §05](../09-NetworkingAndHttp/05-Authentication.md).)

The principle: cookies for same-origin server-rendered apps, bearer tokens for APIs/mobile, OIDC to delegate/SSO, client-credentials/managed-identity for service-to-service.

</details>

---

You can now validate and issue JWTs, build policy-based and resource-based authorization with custom requirements/handlers, enrich claims, protect tokens with Data Protection (shared key ring), secure an API by default, and choose the right authentication approach per scenario — while avoiding the classic crypto/validation mistakes.

→ Back to [Chapter 10 README](README.md) · Next chapter: [Chapter 11 — Resilience](../11-Resilience/README.md)
