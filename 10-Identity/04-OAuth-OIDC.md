# OAuth 2.0 & OpenID Connect

## Delegated authorization and authentication

**OAuth 2.0** is a framework for **delegated authorization** — letting an app access resources on a user's behalf without handling their password (the user authenticates with a provider, which issues the app a token). **OpenID Connect (OIDC)** is an authentication layer **on top of** OAuth — it adds a standard way to establish *who the user is* (an **id token**) and a discovery mechanism. Together they're how modern apps do "Sign in with Google/Microsoft/Auth0" and enterprise SSO, **outsourcing** auth to an identity provider (IdP).

```csharp
builder.Services.AddAuthentication(options => {
        options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
        options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
    })
    .AddCookie()
    .AddOpenIdConnect(o => {
        o.Authority = "https://login.microsoftonline.com/{tenant}/v2.0";  // the IdP (discovery)
        o.ClientId = clientId;
        o.ClientSecret = clientSecret;
        o.ResponseType = "code";          // authorization code flow
        o.UsePkce = true;                 // PKCE (security — below)
        o.Scope.Add("openid"); o.Scope.Add("profile"); o.Scope.Add("email");
        o.SaveTokens = true;
    });
```

The `AddOpenIdConnect` handler implements the whole login dance with the IdP; you mostly configure the authority, client credentials, and scopes.

---

## OAuth roles & the core terms

```
Resource Owner   — the user
Client           — your app (wants access)
Authorization Server (IdP) — issues tokens (Google, Entra ID, Auth0, your IdentityServer)
Resource Server  — the API the token grants access to
```

- **Authorization Server / IdP** — authenticates the user and issues tokens.
- **Access token** — grants the client access to APIs (sent as `Authorization: Bearer` — [03-JWT.md](03-JWT.md)).
- **ID token** (OIDC only) — a JWT proving *who* the user is (their identity claims) — this is the authentication piece OAuth alone lacks.
- **Refresh token** — exchanged for new access tokens ([03-JWT.md](03-JWT.md)).
- **Scopes** — what the token grants (`openid`, `profile`, `email`, `api.read`).

OAuth alone is about *authorization* (access); **OIDC adds the ID token for *authentication*** (login). "Sign in with X" = OIDC.

---

## The flows — use Authorization Code + PKCE

OAuth has several grant types ("flows"); the modern, secure choice for almost all apps is **Authorization Code with PKCE**:

| Flow | Use |
|---|---|
| **Authorization Code + PKCE** | **the default** — web apps, SPAs, mobile, native apps |
| **Client Credentials** | service-to-service (no user — [Ch09 §05](../09-NetworkingAndHttp/05-Authentication.md)) |
| **Device Code** | input-constrained devices (TVs, CLI) |
| ~~Implicit~~ | **deprecated** (token in URL — insecure) |
| ~~Resource Owner Password~~ | **deprecated** (app handles the password — defeats the point) |

```
Authorization Code flow:
1. App redirects user to the IdP (with client_id, scopes, a PKCE code_challenge)
2. User authenticates at the IdP (the app NEVER sees the password)
3. IdP redirects back to the app with an authorization CODE
4. App exchanges the code (+ PKCE code_verifier) at the IdP for tokens (id + access + refresh)
5. App validates the id token, establishes the user's identity
```

The user authenticates **at the IdP**, so your app never touches their credentials. The short-lived **code** (not the token) travels through the browser redirect; the actual token exchange happens server-to-server (or with PKCE for public clients).

---

## PKCE — securing public clients

**PKCE** (Proof Key for Code Exchange, "pixie") protects the authorization-code flow against code interception, essential for **public clients** (SPAs, mobile) that can't keep a client secret:

```
1. App generates a random code_verifier; sends its hash (code_challenge) with the auth request
2. IdP returns the authorization code
3. App exchanges the code + the original code_verifier
4. IdP verifies the verifier hashes to the challenge → only the app that started the flow can redeem the code
```

PKCE ensures that even if an attacker intercepts the authorization code (e.g., from a redirect), they can't exchange it for tokens without the `code_verifier` (which never left the legitimate client). It's now recommended for **all** authorization-code flows (not just public clients) — always set `UsePkce = true`. The implicit flow (which PKCE replaces) is deprecated.

---

## Discovery & key fetching

OIDC providers expose a **discovery document** at `/.well-known/openid-configuration` listing endpoints (authorization, token, userinfo) and the **JWKS** URI (public signing keys). The handler uses this to:
- Find the right endpoints (you just configure the `Authority`).
- Fetch and cache the IdP's **public keys** to validate token signatures ([03-JWT.md](03-JWT.md)) — and refresh them on rotation automatically.

```csharp
o.Authority = "https://accounts.google.com";   // handler discovers endpoints + keys from /.well-known/openid-configuration
```

This is why you only configure the `Authority` — discovery wires up the rest. It also means validating tokens from an IdP requires **no shared secret** (you fetch the public keys).

---

## Validating tokens from an IdP (API side)

An API that accepts tokens from an external IdP just configures JWT bearer with the IdP's authority:

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(o => {
        o.Authority = "https://login.microsoftonline.com/{tenant}/v2.0";   // discovers keys
        o.Audience = "api://my-api";                                        // validate aud
    });
```

The API validates incoming access tokens against the IdP's published keys (fetched via discovery) and the expected audience — no secret needed. This is the standard pattern: an IdP issues tokens, your APIs validate them statelessly. Use **`Microsoft.Identity.Web`** for Entra ID (it adds token caching, on-behalf-of flows, and managed-identity integration).

---

## When to use an IdP vs own Identity

| Approach | When |
|---|---|
| **External IdP via OIDC** (Entra ID, Auth0, Okta, Google) | SSO, social login, **outsource** auth — the modern default |
| **ASP.NET Core Identity** ([02](02-AspNetIdentity.md)) | you must own the user store/flows |
| **Be an IdP** (Duende IdentityServer, OpenIddict) | you issue tokens to multiple clients yourself |

Delegating to an IdP offloads the hardest, riskiest work (account security, MFA, threat detection, compliance, password resets) and enables SSO/social login — increasingly the default. Run your own IdP (IdentityServer/OpenIddict) only when you need to *be* the authorization server for multiple apps.

---

## Common gotchas

### Using deprecated flows

The **implicit** flow (token in the redirect URL) and **resource-owner-password** flow are insecure/deprecated. Use **Authorization Code + PKCE** for user login, client credentials for service-to-service.

### Not using PKCE

Without PKCE, an intercepted authorization code can be redeemed by an attacker (especially for public clients). Always `UsePkce = true`.

### Confusing access token and ID token

The **access token** authorizes API calls; the **ID token** authenticates the user (proves identity) and is for the client, not for calling APIs. Don't send the ID token to APIs as the bearer token.

### Hardcoding/leaking the client secret

Confidential clients' secrets go in a secret store/Key Vault, never source. Public clients (SPAs/mobile) **can't keep a secret** — rely on PKCE instead.

### Not validating audience on the API

An API must validate the token's `aud` so a token issued for another service isn't accepted. Set `Audience`.

### Reinventing the IdP

Building your own OAuth/OIDC server is a major security undertaking. Use an external IdP, or a vetted library (IdentityServer/OpenIddict) if you must be one.

---

## Summary

- **OAuth 2.0** = delegated **authorization** (access without handling passwords); **OpenID Connect** adds **authentication** (the **ID token** proving who the user is) + discovery — together, "Sign in with X" and SSO, **outsourcing** auth to an **IdP**.
- Use the **Authorization Code flow with PKCE** for all user-facing apps (the user authenticates at the IdP; your app never sees the password); **client credentials** for service-to-service. Implicit and password flows are **deprecated**.
- **PKCE** protects against authorization-code interception (essential for public clients; recommended for all) — always enable it.
- OIDC **discovery** (`/.well-known/openid-configuration` + JWKS) auto-wires endpoints and public keys, so APIs validate IdP tokens statelessly with **no shared secret** (just `Authority` + `Audience`).
- Prefer **delegating to an external IdP** (offloads the hardest security work, enables SSO) over owning auth; distinguish **access tokens** (call APIs) from **ID tokens** (authenticate the user); never reinvent the IdP.

→ Next: [05-Cookies.md](05-Cookies.md)
