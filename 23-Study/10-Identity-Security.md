# 10 — Identity & Security

## ⚡ 30-second answer

Two halves, never conflated: **authentication** ("who are you?" — establishes a
`ClaimsPrincipal`) and **authorization** ("what may you do?" — decides using that identity). The
pipeline order is mandatory: **routing → `UseAuthentication` → `UseAuthorization` → endpoints**.
**401** means not authenticated; **403** means authenticated but forbidden. A **JWT** is a
*signed*, self-contained set of claims — **encoded, not encrypted**, so never put secrets in it —
and you must validate **signature, `exp`, `iss` and `aud`** on every request. Prefer **delegating
to an IdP via OIDC** (Authorization Code flow **with PKCE**) over owning passwords; if you must
own them, use **ASP.NET Core Identity** rather than rolling your own. Cookies suit server-rendered
apps and **need CSRF protection**; bearer tokens suit APIs and don't. And the multi-instance
production trap: **persist the Data Protection key ring to shared storage**, or users get logged
out at random.

---

## Core mechanics

### The pipeline

```csharp
app.UseRouting();
app.UseAuthentication();     // populates HttpContext.User — who
app.UseAuthorization();      // enforces policies — what
app.MapControllers();
```

Routing first so auth can read the endpoint's `[Authorize]` metadata; authentication before
authorization because there's nothing to authorize otherwise ([04](04-AspNetCore.md)).

**Endpoints are public by default.** The secure-by-default pattern is a **fallback policy** that
requires authentication, with explicit `[AllowAnonymous]` opt-outs:

```csharp
builder.Services.AddAuthorizationBuilder()
    .SetFallbackPolicy(new AuthorizationPolicyBuilder().RequireAuthenticatedUser().Build());
```

### JWT

```text
header.payload.signature     ← all three are base64url — the payload is READABLE by anyone
```

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(o =>
    {
        o.Authority = "https://idp.example.com";   // discovery + JWKS: no shared secret needed
        o.Audience  = "api://catalog";
        o.TokenValidationParameters = new()
        {
            ValidateIssuer = true, ValidateAudience = true,
            ValidateLifetime = true, ValidateIssuerSigningKey = true
        };
    });
```

Validate **all four**: signature, expiry (`exp`), issuer (`iss`), audience (`aud`). Skipping any
one is a real vulnerability — an unvalidated audience means a token minted for another service is
accepted by yours.

**Asymmetric (RS256)** — the IdP signs with a private key, you validate with the public key from
its JWKS endpoint. That's what lets many services validate tokens with no shared secret.
**Symmetric (HS256)** only when issuer and validator are the same app.

Statelessness means **no easy revocation**: use **short-lived access tokens plus revocable refresh
tokens**, and support **key rotation** via JWKS and the `kid` header.

### OAuth 2.0 and OpenID Connect

- **OAuth 2.0** = delegated **authorization** — get access to a resource without handling the
  user's password.
- **OIDC** = OAuth plus **authentication** — adds the **ID token** (proving who the user is) and
  discovery. This is "Sign in with X" and SSO.

| Flow | Use |
|---|---|
| **Authorization Code + PKCE** | **all user-facing apps** — web, SPA, mobile |
| **Client credentials** | service-to-service, no user |
| Implicit, Resource Owner Password | **deprecated** — don't |

**PKCE** protects against authorization-code interception. Always enable it.

**Discovery** (`/.well-known/openid-configuration` + JWKS) auto-wires endpoints and public keys —
which is why an API needs only `Authority` and `Audience`.

**Access token** = call an API. **ID token** = authenticate the user in *your* app. Don't send an
ID token to an API.

### Cookies vs bearer tokens

| | Cookie auth | JWT bearer |
|---|---|---|
| Carried | automatically by the browser | explicit `Authorization` header |
| Readable by client | ❌ encrypted (Data Protection) | ✅ **payload is readable** |
| Revocable | ✅ server-side | ❌ until expiry |
| **CSRF risk** | ✅ **yes — needs antiforgery** | ❌ no (header isn't auto-sent) |
| Fits | server-rendered apps, Blazor Server | APIs, SPAs, mobile, microservices |

Cookie flags to state without hesitating: **`HttpOnly`** (blocks XSS reading it), **`Secure`**
(HTTPS only), **`SameSite`** (`Lax`/`Strict`, blocks cross-site sending).

### Authorization models

```csharp
// Policy-based — the recommended default
builder.Services.AddAuthorizationBuilder()
    .AddPolicy("CanApprove", p => p.RequireClaim("permission", "orders.approve"));

app.MapPost("/orders/{id}/approve", …).RequireAuthorization("CanApprove");

// Resource-based — when the decision depends on the specific instance
var result = await authz.AuthorizeAsync(User, order, "SameOwner");
if (!result.Succeeded) return TypedResults.Forbid();
```

The progression: **authenticated-only** → **role-based** (coarse) → **claims/policy-based**
(recommended) → **resource-based** (per-instance ownership). A **policy** is a named set of
**requirements**; requirements plus handlers encapsulate arbitrary logic, can inject services, and
are unit-testable — which decouples *what is required* from *where it's enforced*.

### Claims

A **claim** is a `Type`/`Value` statement. The authenticated user is a **`ClaimsPrincipal`**
holding `ClaimsIdentity`s of `Claim`s — authentication produces claims, authorization consumes
them.

Map external IdP claim names to what your app expects (`NameClaimType`, `RoleClaimType`,
`ClaimActions`); consider `MapInboundClaims = false` for predictable names.
**`IClaimsTransformation`** enriches the principal with database-sourced claims after
authentication — it runs **per request**, so make it cheap and **idempotent**.

Prefer **fine-grained permission claims** over coarse roles, keep tokens lean, and never put
secrets in claims.

### Data Protection

`IDataProtector` gives authenticated **encryption + signing** for short app-managed secrets — auth
cookies, antiforgery tokens, password-reset and email-confirmation tokens, TempData — with an
automatically rotated key ring.

```csharp
var protector = provider.CreateProtector("PasswordReset.v1");   // purpose isolates uses
```

The **purpose string** cryptographically isolates uses, so a reset token can't be replayed as a
confirmation token.

**The critical production configuration**: in a multi-instance deployment, **persist the key ring
to shared, durable storage** (a file share, blob storage, Redis) and **encrypt it at rest**.
Otherwise each instance generates its own keys, can't decrypt the others' cookies, and users are
logged out apparently at random. Apps are isolated by default; set a shared application name only
when you deliberately want to share.

### CSRF / antiforgery

CSRF tricks a logged-in user's browser into making an authenticated request they didn't intend,
exploiting the fact that **cookies are sent automatically cross-site**. An **antiforgery token**
is embedded in the page and paired with a cookie; a cross-origin attacker can't read it
(same-origin policy), so the forged request lacks a valid token.

Razor Pages validate antiforgery **by default**; MVC uses `[ValidateAntiForgeryToken]` or
`[AutoValidateAntiforgeryToken]`. **`SameSite`** cookies are the browser-level complement — use
both.

**Cookie auth needs CSRF protection; bearer-token APIs don't**, because a header isn't sent
automatically. A cookie-authenticated SPA does need header-based antiforgery.

### Crypto posture

Stay as high-level as possible: **Data Protection** for cookies and short-lived tokens,
**Identity** for passwords and accounts, an **IdP via OIDC** for login and SSO, a **secret store**
(Key Vault) for secrets at rest. Only then, primitives — and by the rules in
[02](02-BCL-Essentials.md): `RandomNumberGenerator`, a slow salted KDF for passwords, AEAD with a
unique nonce, `FixedTimeEquals`. Encrypt in transit with TLS 1.2/1.3, mTLS between services.

---

## 🪤 Traps & gotchas

- **`UseAuthorization()` before `UseAuthentication()`** — there's no identity to authorize yet, so
  everything is anonymous and `[Authorize]` rejects valid users.
- **Auth middleware before `UseRouting()`** — it can't see the endpoint's `[Authorize]` metadata.
- **Assuming endpoints are protected by default** — they are public. One forgotten `[Authorize]`
  is an open endpoint. Set a fallback policy.
- **Putting secrets in a JWT** — the payload is base64url, not encrypted. Anyone with the token
  reads every claim. Paste one into jwt.io and it's all there.
- **Not validating the audience** — a token minted for a different service in the same IdP is
  accepted by yours. Same for issuer.
- **Not validating the signature (`ValidateIssuerSigningKey = false`)** or accepting `alg: none` —
  anyone can mint a token with any claims.
- **Long-lived access tokens** — a stateless token can't be revoked, so a stolen one is valid for
  its whole lifetime. Short access tokens plus revocable refresh tokens.
- **Storing JWTs in `localStorage`** — readable by any XSS. An `HttpOnly` cookie isn't.
- **Using the implicit or password grant** — both deprecated. Authorization Code + PKCE.
- **Skipping PKCE on a public client** — an intercepted authorization code can be redeemed by the
  attacker.
- **Sending an ID token to an API** — it authenticates a user to *your* app; it isn't an access
  token and the audience won't match.
- **Not persisting the Data Protection key ring** — the single most common ASP.NET Core
  production auth bug. Each instance makes its own keys, so a cookie issued by one is rejected by
  another and users are randomly signed out. Also breaks antiforgery and reset tokens.
- **Reusing a Data Protection purpose string** across features — a token issued for one purpose is
  accepted for another.
- **Rolling your own password hashing** — a fast hash, no salt, or a home-made scheme. Use
  Identity, which does PBKDF2 with a per-user salt, lockout, and timing-safe verification.
- **Comparing tokens or hashes with `==`** — timing leak. `FixedTimeEquals`.
- **Turning off antiforgery** because it was "getting in the way" of a cookie-authenticated form
  or SPA.
- **Cookies without `HttpOnly`/`Secure`/`SameSite`** — respectively readable by XSS, sniffable over
  HTTP, and sent cross-site.
- **Roles used for fine-grained permission** — role explosion (`OrdersApproverEurope`) and a
  redeploy for every permission change. Permission claims plus policies.
- **Trusting an unvalidated claim** — a tenant id or user id from a header or the request body
  rather than the validated principal is horizontal privilege escalation
  ([22](22-BestPractices-Architecture.md)).
- **Expensive or non-idempotent `IClaimsTransformation`** — it runs on **every request**, so a
  database call there is a per-request tax, and appending a claim each time grows the principal.
- **Returning 401 where 403 is meant** (or vice versa) — 401 tells the client to re-authenticate,
  which loops forever if the real problem is insufficient permission.
- **Leaking which part of a login failed** ("no such user" vs "wrong password") — user
  enumeration.

---

## ❓ Likely questions

**Q: Authentication vs authorization?**
A: Authentication establishes **who** the caller is, producing a `ClaimsPrincipal`. Authorization
decides **what** that identity may do, using policies, claims or roles. 401 means not
authenticated; 403 means authenticated but not permitted.

**Q: What is a JWT and how is it validated?**
A: A base64url-encoded `header.payload.signature` carrying claims, sent as a bearer token. The
server validates the **signature**, **expiry**, **issuer** and **audience** — statelessly, with no
session or database lookup, which is why it suits APIs and microservices.

**Q: Is a JWT encrypted?**
A: No — it's **signed**, not encrypted. The payload is readable by anyone holding the token. The
signature guarantees integrity and origin, not confidentiality, so never put secrets in it.

**Q: RS256 or HS256?**
A: RS256 (asymmetric) when the issuer and validators differ — the IdP signs with its private key
and every service validates with the public key from JWKS, so no shared secret is distributed.
HS256 (symmetric) only when the same app both issues and validates.

**Q: How do you revoke a JWT?**
A: You largely can't — that's the cost of statelessness. The practical answer is short-lived
access tokens (minutes) plus longer-lived **refresh tokens that are stored and revocable**, so
revocation takes effect within one access-token lifetime. A denylist of `jti` values works but
reintroduces the state you were avoiding.

**Q: OAuth vs OpenID Connect?**
A: OAuth 2.0 is delegated **authorization** — getting access to a resource without the user's
password. OIDC is a layer on top that adds **authentication**: the ID token proving who the user
is, plus discovery and a userinfo endpoint. OAuth alone doesn't tell you who logged in.

**Q: Which OAuth flow, and why PKCE?**
A: Authorization Code with PKCE for every user-facing app. PKCE binds the authorization code to
the client that requested it, so an intercepted code can't be redeemed by an attacker. Implicit
and password grants are deprecated.

**Q: Access token vs ID token?**
A: An access token is for calling an API — the API validates it and checks the audience. An ID
token authenticates the user to *your* client application and shouldn't be sent to an API at all.

**Q: Cookies or JWTs?**
A: Cookies for same-origin server-rendered apps — opaque to the client, revocable server-side,
handled automatically by the browser, but they carry CSRF risk. JWTs for APIs, SPAs, mobile and
service-to-service — stateless and CSRF-free, but not revocable and readable by whoever holds
them.

**Q: What is CSRF and who needs to defend against it?**
A: An attacker's site causes the victim's browser to make an authenticated request, exploiting
that cookies are attached automatically to cross-site requests. **Cookie-authenticated apps need
antiforgery tokens**; bearer-token APIs don't, because the header isn't sent automatically.

**Q: How do antiforgery tokens work?**
A: The server embeds a token in the page and pairs it with a cookie. A cross-origin attacker can't
read the page because of the same-origin policy, so a forged request can't include a valid token.
`SameSite` cookies are the complementary browser-level defence — use both.

**Q: What is Data Protection and what's the classic production bug?**
A: An API for authenticated encryption of short app-managed secrets — auth cookies, antiforgery
and reset tokens — with an auto-rotated key ring and purpose-based isolation. The classic bug is
not persisting the key ring to shared storage in a multi-instance deployment: each instance
generates its own keys, so cookies issued by one are rejected by another and users get logged out
seemingly at random.

**Q: Roles or claims-based policies?**
A: Policies over claims. Roles are coarse and lead to role explosion plus a redeploy for every
permission change. A policy is a named set of requirements, handlers can inject services and
inspect the resource, and the whole thing is unit-testable.

**Q: When do you need resource-based authorization?**
A: When the decision depends on the specific instance — "may this user edit *this* document?" You
can't answer that from claims alone, so you load the resource and call
`IAuthorizationService.AuthorizeAsync` with it.

**Q: Should you build your own login?**
A: No. Delegate to an IdP over OIDC if you can — it offloads the hardest security work and gives
you SSO and MFA. If you must own accounts, use ASP.NET Core Identity, which already handles salted
PBKDF2 hashing, lockout, secure token generation and timing-safe comparison.

**Q: How should passwords be stored?**
A: A deliberately slow, salted key-derivation function — PBKDF2, bcrypt or Argon2 — with a
per-user salt and a tuned work factor. Never a fast hash, which a GPU computes billions of times
per second.

---

## 🎓 Senior Extra

- **Secure by default is a configuration, not a discipline.** A fallback authorization policy plus
  explicit `[AllowAnonymous]` turns "we forgot `[Authorize]`" from an open endpoint into a 401.
- **Token lifetime is a trade between revocation latency and IdP load.** Five to fifteen minutes
  for access tokens is the usual landing spot; the refresh token is where you put the revocable
  state and the rotation detection.
- **Refresh-token rotation with reuse detection** — issue a new refresh token on each use and
  invalidate the family if an old one reappears. That's how you detect a stolen token rather than
  merely limiting its lifetime.
- **Claims transformation runs per request**, so cache what it looks up. And make it idempotent —
  a naive implementation that appends a claim adds one more copy on each call.
- **`MapInboundClaims = false`** stops the legacy WS-Fed mapping from renaming `sub` to a long
  URI, which is the reason so many people can't find the claim they expect.
- **Key rotation is why you validate against JWKS rather than a pinned key** — the IdP publishes
  the new key before it starts using it, and the `kid` header tells you which one to use, so
  rotation is invisible.
- **mTLS between services** authenticates both ends at the transport layer and removes a whole
  class of token-handling code — increasingly the default inside a service mesh.
- **Encrypt the Data Protection key ring at rest** (Key Vault, DPAPI, a certificate) — persisting
  it to a blob container is necessary but not sufficient, since anyone who reads that blob can mint
  your auth cookies.
- **Threat-model the token's journey**, not just its contents: where it's stored on the client
  (`localStorage` is XSS-readable, an `HttpOnly` cookie isn't), whether it crosses a proxy that
  logs headers, and how long it survives a logout.
- **Authorization belongs at the boundary *and* in the domain.** An endpoint policy stops the
  wrong caller; an aggregate that refuses to transition state protects you when a new endpoint
  forgets ([22](22-BestPractices-Architecture.md)).

→ Deeper: [`../10-Identity/`](../10-Identity/README.md)
