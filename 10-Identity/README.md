# Chapter 10 — Identity & Security

> Authentication, authorization, ASP.NET Core Identity, JWTs, OAuth/OIDC, Data Protection. How to know who's calling and what they can do.

**Prerequisites**: Chapter 04 (ASP.NET Core).

**Time to read**: ~8-10 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-AuthN-vs-AuthZ.md](01-AuthN-vs-AuthZ.md) | The two halves: authentication (who?), authorization (what can?). |
| [02-AspNetIdentity.md](02-AspNetIdentity.md) | The Identity framework: users, roles, claims, password hashing. |
| [03-JWT.md](03-JWT.md) | JSON Web Tokens — structure, validation, key rotation, refresh patterns. |
| [04-OAuth-OIDC.md](04-OAuth-OIDC.md) | OAuth 2.0 flows, OpenID Connect, the role of the IdP, PKCE. |
| [05-Cookies.md](05-Cookies.md) | Cookie authentication — when it still wins. |
| [06-Authorization.md](06-Authorization.md) | Policy-based authorization, requirements, handlers, attribute vs minimal API forms. |
| [07-Claims.md](07-Claims.md) | The claims principal model; transformations; mapping from external IdPs. |
| [08-DataProtection.md](08-DataProtection.md) | `IDataProtector` — key ring, persistence, distributed setups. |
| [09-Antiforgery.md](09-Antiforgery.md) | CSRF protection in ASP.NET Core. |
| [10-Cryptography.md](10-Cryptography.md) | When to roll your own (almost never) and which BCL APIs to use. |
| [Questions.md](Questions.md) | Drilling. |
| [Coding.md](Coding.md) | JWT issuance + validation, custom auth policy, certificate auth. |

→ Begin: [01-AuthN-vs-AuthZ.md](01-AuthN-vs-AuthZ.md)
