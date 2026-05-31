# JSON Web Tokens (JWT)

## Stateless, self-contained tokens

A JWT is a signed, self-contained token carrying **claims** about the caller. The server issues it at login; the client sends it on each request (`Authorization: Bearer <token>`); the server **validates the signature** and reads the claims — without a database lookup or server-side session. This statelessness makes JWTs the default for APIs, SPAs, mobile apps, and microservices.

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(o => {
        o.TokenValidationParameters = new TokenValidationParameters {
            ValidateIssuer = true,           ValidIssuer = "https://my-issuer",
            ValidateAudience = true,         ValidAudience = "my-api",
            ValidateLifetime = true,          // reject expired tokens
            ValidateIssuerSigningKey = true,  IssuerSigningKey = signingKey,
            ClockSkew = TimeSpan.FromSeconds(30)
        };
    });
```

After this, a valid bearer token populates `HttpContext.User` with the token's claims ([01-AuthN-vs-AuthZ.md](01-AuthN-vs-AuthZ.md)).

---

## Structure: header.payload.signature

A JWT is three Base64Url-encoded parts joined by dots:

```
eyJhbGc...   .   eyJzdWI...   .   SflKxwRJ...
  HEADER          PAYLOAD          SIGNATURE
{alg, typ}    {claims: sub,      HMAC/RSA over
              name, role,        header+payload
              exp, iss, aud}     using the key
```

- **Header** — algorithm (`alg`: HS256, RS256, ...) and type.
- **Payload** — the **claims**: standard ones (`sub` = subject/user id, `exp` = expiry, `iss` = issuer, `aud` = audience, `iat` = issued-at) plus custom claims (name, roles, permissions).
- **Signature** — a cryptographic signature over header+payload, proving the token wasn't tampered with and was issued by a trusted party.

Critical: **the payload is encoded, not encrypted** — anyone can decode and read the claims (paste a JWT into jwt.io). The signature ensures *integrity* (no tampering), not *confidentiality*. **Never put secrets in a JWT.** Only the signature is secret-dependent.

---

## Validation — what the server checks

Validation is the heart of JWT security. The server must verify **every** request's token:

1. **Signature** — recompute it with the key; reject if it doesn't match (catches tampering/forgery).
2. **Expiry (`exp`)** — reject expired tokens (`ValidateLifetime`).
3. **Issuer (`iss`)** — was it issued by a trusted authority? (`ValidateIssuer`).
4. **Audience (`aud`)** — is this token meant for *this* API? (`ValidateAudience`).
5. **Not-before (`nbf`)**, clock skew tolerance.

```csharp
ValidateIssuerSigningKey = true,   // ← the most important: verify the signature
ValidateLifetime = true,
ValidateIssuer = true, ValidateAudience = true,
```

The `JwtBearer` handler does all this per request. Skipping any check is a vulnerability — e.g., not validating the signature means anyone can forge a token; not validating the audience means a token for service A is accepted by service B. **Validate signature, expiry, issuer, and audience.**

---

## Symmetric vs asymmetric signing

```csharp
// Symmetric (HS256) — same secret signs and validates. Simple, but the validator must KNOW the secret.
IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(secret))

// Asymmetric (RS256/ES256) — PRIVATE key signs (issuer only), PUBLIC key validates (anyone).
// The standard for distributed systems / external IdPs.
```

- **Symmetric (HS256)** — one shared secret for signing and validating. Fine when the issuer and validator are the same app, but every validator needs the secret (a leak forges tokens).
- **Asymmetric (RS256/ES256)** — the issuer signs with a **private** key; validators verify with the **public** key. Validators never hold a signing secret, and the public key can be published (via a **JWKS** endpoint). This is the standard for OIDC and microservices — services validate tokens from an IdP using its published public keys without sharing secrets.

For external IdPs / multiple services, use **asymmetric** signing. The handler can auto-fetch keys from the IdP's discovery/JWKS endpoint ([04-OAuth-OIDC.md](04-OAuth-OIDC.md)).

---

## Issuing a token

```csharp
var claims = new[] {
    new Claim(JwtRegisteredClaimNames.Sub, user.Id),
    new Claim(JwtRegisteredClaimNames.Email, user.Email),
    new Claim(ClaimTypes.Role, "Admin"),
    new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())   // unique token id
};
var token = new JwtSecurityToken(
    issuer: "https://my-issuer", audience: "my-api",
    claims: claims,
    expires: DateTime.UtcNow.AddMinutes(15),                            // SHORT-lived!
    signingCredentials: new SigningCredentials(key, SecurityAlgorithms.HmacSha256));
string jwt = new JwtSecurityTokenHandler().WriteToken(token);
```

In practice, you usually let **Identity**, an **IdP**, or a token server issue tokens rather than hand-crafting them. Key choices: include only necessary claims (no secrets, no huge payloads), set a **short expiry** (below), and a unique `jti` for revocation/dedup.

---

## The expiry/revocation problem and refresh tokens

JWTs are **stateless** — that's their strength and their weakness. Once issued, a JWT is valid until it **expires**; you **can't easily revoke** it (no server-side session to invalidate). So a stolen token works until expiry. The mitigation:

- **Short-lived access tokens** (~5–15 minutes) — limit the window a stolen token is useful.
- **Long-lived refresh tokens** — a separate, **stored, revocable** token the client exchanges for a new access token when it expires. Refresh tokens *are* tracked server-side (so they can be revoked), giving you a revocation point without checking every access token against a DB.

```
Login → short access token (15 min) + refresh token (stored, e.g., 7 days)
Access token expires → client POSTs the refresh token → new access token
Logout / compromise → revoke the refresh token (it's stored) → no new access tokens issued
```

This balances statelessness (fast, no per-request DB lookup for access tokens) with revocability (via the stored refresh token). For immediate access-token revocation you'd need a denylist (checking a store per request — partly defeating statelessness) — usually short expiry + refresh-token revocation suffices. `MapIdentityApi` ([02-AspNetIdentity.md](02-AspNetIdentity.md)) provides login/refresh endpoints implementing this.

---

## Key rotation

Signing keys must be **rotatable** (in case of compromise, and as good hygiene). With asymmetric keys + a JWKS endpoint, the IdP publishes multiple keys (each with a `kid` = key id); tokens reference the key they were signed with, so validators pick the right published key — enabling rotation without downtime (old tokens validate with the old key until they expire, new tokens use the new key). The `JwtBearer` handler caches and refreshes keys from the JWKS endpoint automatically. Don't hardcode a single static signing key forever.

---

## Common gotchas

### Putting secrets in the payload

JWTs are **encoded, not encrypted** — anyone can read the claims. Never include passwords, secrets, or sensitive PII. The signature protects integrity, not confidentiality.

### Not validating the signature (or audience/issuer)

Skipping signature validation lets anyone forge tokens; skipping audience lets a token for one service be used on another. Validate signature, expiry, issuer, **and** audience.

### Long-lived access tokens

A long expiry means a stolen token is usable for a long time and can't be revoked. Use **short** access tokens + refresh tokens.

### Expecting easy revocation

Stateless JWTs can't be revoked before expiry without a denylist. Use short expiry + revocable refresh tokens; only add a denylist if immediate revocation is required.

### Symmetric key shared widely

With HS256, every validator needs the secret — a leak anywhere forges tokens. Use **asymmetric** (RS256) for multiple validators/services so only the issuer holds the private key.

### Hardcoded, never-rotated key

A static signing key forever is a liability. Support rotation (JWKS + `kid`); fetch keys dynamically for external IdPs.

---

## Summary

- A **JWT** is a signed, self-contained token of **claims** (`header.payload.signature`); the client sends it as `Authorization: Bearer`, the server **validates the signature** and reads claims **statelessly** (no session/DB lookup) — the default for APIs/SPAs/microservices.
- The payload is **encoded, not encrypted** — anyone can read it; **never put secrets in it**. The signature ensures integrity, not confidentiality.
- **Validate** signature, expiry (`exp`), issuer (`iss`), and audience (`aud`) on every request — skipping any is a vulnerability.
- Use **asymmetric** signing (RS256, private signs / public validates via JWKS) for IdPs/microservices; symmetric (HS256) only when issuer = validator.
- Statelessness means **no easy revocation** → use **short-lived access tokens + revocable refresh tokens**; support **key rotation** (JWKS + `kid`). Prefer Identity/an IdP to issue tokens over hand-crafting them.

→ Next: [04-OAuth-OIDC.md](04-OAuth-OIDC.md)
