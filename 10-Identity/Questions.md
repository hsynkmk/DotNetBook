# Chapter 10 — Identity & Security — Q & A

---

### Q1. Authentication vs authorization?

**Authentication** establishes *who you are* (verifying credentials/tokens → a `ClaimsPrincipal`). **Authorization** decides *what you can do* (using that identity). AuthN runs first; the pipeline is routing → `UseAuthentication` → `UseAuthorization` → endpoint.

---

### Q2. 401 vs 403?

**401 Unauthorized** = *not authenticated* (no/invalid credentials — re-authenticate). **403 Forbidden** = authenticated but *not authorized* (known user lacks permission — re-authenticating won't help). The names are confusing; 401 really means un-authenticated.

---

### Q3. Why use ASP.NET Core Identity instead of building your own?

Secure account management has many security-critical, easy-to-botch details: slow salted password hashing (PBKDF2/Argon2, not fast hashes), account lockout, cryptographically secure single-use reset tokens, timing-safe comparisons, 2FA. Identity implements these correctly; rolling your own risks serious vulnerabilities.

---

### Q4. Roles vs claims for authorization?

Roles are coarse-grained groups (a special claim of type `ClaimTypes.Role`). Claims are fine-grained attributes (`permission: orders:write`). Prefer **claims + policy-based authorization** for granular control — roles proliferate unmanageably when used for fine-grained permissions.

---

### Q5. What is a JWT and why is it stateless?

A signed, self-contained token of claims sent as `Authorization: Bearer`. The server validates the signature and reads claims **without a session/DB lookup** — that statelessness makes it ideal for APIs/SPAs/microservices.

---

### Q6. Is a JWT encrypted? What does the signature provide?

No — the payload is **Base64Url-encoded, not encrypted** (anyone can decode and read the claims). The signature provides **integrity** (no tampering) and authenticity (issued by a trusted party), not confidentiality. Never put secrets in a JWT.

---

### Q7. What must you validate on a JWT?

Signature (with the key — catches forgery/tampering), expiry (`exp`), issuer (`iss`), and audience (`aud`). Skipping any is a vulnerability — e.g., not validating the audience lets a token for service A be accepted by service B.

---

### Q8. Symmetric vs asymmetric JWT signing?

**Symmetric (HS256)** — one shared secret signs and validates (every validator needs the secret; a leak forges tokens). **Asymmetric (RS256)** — private key signs (issuer only), public key validates (published via JWKS). Use asymmetric for IdPs/microservices so validators hold no signing secret.

---

### Q9. How do you handle JWT revocation, given they're stateless?

You can't easily revoke a stateless JWT before expiry. Use **short-lived access tokens** (~5–15 min) + **long-lived, stored, revocable refresh tokens**: the access token limits the stolen-token window; revoking the refresh token (which is tracked) stops new access tokens. A denylist gives immediate revocation at the cost of per-request lookups.

---

### Q10. OAuth 2.0 vs OpenID Connect?

OAuth 2.0 is delegated **authorization** (access without handling passwords). OIDC is an **authentication** layer on top — it adds the **ID token** (proving who the user is) and discovery. "Sign in with X" / SSO is OIDC.

---

### Q11. Which OAuth flow should you use, and why PKCE?

**Authorization Code with PKCE** for all user-facing apps (web/SPA/mobile). The user authenticates at the IdP (your app never sees the password); **PKCE** prevents an intercepted authorization code from being redeemed by an attacker (essential for public clients; recommended for all). Implicit and password flows are deprecated.

---

### Q12. Access token vs ID token?

The **access token** authorizes API calls (sent as the bearer token). The **ID token** authenticates the user (proves identity, for the client) and should **not** be used to call APIs. Confusing them is a common mistake.

---

### Q13. How does an API validate tokens from an external IdP without a shared secret?

Via OIDC **discovery** (`/.well-known/openid-configuration` → JWKS): the API fetches the IdP's **public keys** to validate token signatures, and validates the audience. Configure just `Authority` + `Audience`; no secret needed (asymmetric signing).

---

### Q14. Cookie auth vs JWT bearer — when each?

**Cookies** for server-rendered web apps (same-origin browser): encrypted, opaque to the client, revocable server-side — but CSRF-prone. **JWT bearer** for APIs/SPAs/mobile/microservices: header-carried, client-readable, stateless, not CSRF-vulnerable, natural cross-service.

---

### Q15. Why is cookie auth vulnerable to CSRF but token auth isn't?

Browsers **auto-attach cookies** to requests for the origin (including those triggered by other sites), so a malicious site can forge an authenticated request. The `Authorization` header (bearer tokens) is **not** auto-sent cross-site, so token auth isn't CSRF-vulnerable.

---

### Q16. How do antiforgery tokens work?

The server embeds a secret token in the page (hidden field) and a paired cookie; on submit both are sent and validated to match. A cross-origin attacker can't read your page (same-origin policy) to get the token, so they can't forge a valid request. Razor Pages validate this automatically.

---

### Q17. What is the SameSite cookie attribute's role in CSRF?

`SameSite=Lax`/`Strict` tells the browser not to send the cookie on cross-site requests (Lax allows top-level navigations but blocks cross-site form posts) — a browser-level CSRF defense. Use it **alongside** antiforgery tokens for defense in depth.

---

### Q18. The four authorization models?

**Authenticated-only** (`RequireAuthorization()`), **role-based** (`[Authorize(Roles=...)]`), **claims/policy-based** (named policies of requirements — recommended), and **resource-based** (`IAuthorizationService.AuthorizeAsync(user, resource, policy)` — for per-instance ownership decisions).

---

### Q19. How do custom authorization requirements/handlers work?

Define an `IAuthorizationRequirement` (the rule + parameters) and an `AuthorizationHandler<T>` (the logic that calls `context.Succeed(req)` when met). Register the handler in DI. A policy can have multiple requirements (all must pass); handlers can inject services and inspect the resource.

---

### Q20. How do you make endpoints secure-by-default?

Set a **fallback policy** requiring an authenticated user (`o.FallbackPolicy = ...RequireAuthenticatedUser()`), so all endpoints require auth unless they opt out with `[AllowAnonymous]`. This "deny by default" posture prevents accidentally-public endpoints.

---

### Q21. What is the ClaimsPrincipal model?

The authenticated user is a **`ClaimsPrincipal`** holding one or more **`ClaimsIdentity`** objects, each with **`Claim`**s (Type/Value statements). Authentication produces claims; authorization consumes them. Read via `FindFirst`/`FindAll`/`IsInRole` on `HttpContext.User`.

---

### Q22. What is IClaimsTransformation and what's the gotcha?

A hook that runs **on every request** after authentication to add/modify claims (e.g., load DB-sourced permissions onto the principal). The gotcha: it must be **idempotent** (guard against adding duplicate claims) and cheap, because it runs per request.

---

### Q23. What is the Data Protection API for?

Authenticated **encryption + signing** of short app-managed secrets (auth cookies, antiforgery/reset/confirmation tokens, TempData) with an automatically-rotated **key ring** — no raw crypto. A **purpose** string cryptographically isolates uses.

---

### Q24. What's the critical Data Protection issue in multi-instance deployments?

The **key ring must be shared** across instances. If each instance has its own keys (default in some hosts), a cookie/token protected by instance A can't be validated by instance B → random logouts, failing reset links, broken antiforgery. Persist keys to a shared durable store (file share/DB/Redis/Blob) and encrypt them at rest.

---

### Q25. What's the overarching cryptography rule?

**Don't roll your own, and stay as high-level as possible.** Use `IDataProtection` (cookies/tokens), Identity (passwords/accounts), an IdP (login/SSO), and a secret store (secrets at rest) before touching primitives. If you must use primitives: `RandomNumberGenerator` (not `Random`), slow salted KDF for passwords, AEAD + unique nonce for encryption, `FixedTimeEquals` for secret comparison. Encrypt in transit with TLS.

---

→ Next: [Coding.md](Coding.md)
