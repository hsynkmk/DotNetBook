# Cookie Authentication

## The classic web-app auth

Cookie authentication stores the user's identity in a **signed (and encrypted) cookie** that the browser automatically sends with every request. The server validates the cookie and reconstructs the `ClaimsPrincipal`. It's the natural fit for **server-rendered web apps** (MVC, Razor Pages, Blazor Server) where a browser maintains a session with one server-side app — and it's still the right choice there, despite tokens dominating API scenarios.

```csharp
builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(o => {
        o.LoginPath = "/account/login";
        o.ExpireTimeSpan = TimeSpan.FromHours(8);
        o.SlidingExpiration = true;
        o.Cookie.HttpOnly = true;                          // not readable by JS (XSS protection)
        o.Cookie.SecurePolicy = CookieSecurePolicy.Always;  // HTTPS only
        o.Cookie.SameSite = SameSiteMode.Lax;               // CSRF mitigation
    });

// Sign in (e.g., after validating credentials)
await HttpContext.SignInAsync(CookieAuthenticationDefaults.AuthenticationScheme, claimsPrincipal);
await HttpContext.SignOutAsync();   // sign out
```

`SignInAsync` issues the cookie (with the user's claims); the browser sends it automatically; `UseAuthentication` validates it and sets `HttpContext.User` ([01-AuthN-vs-AuthZ.md](01-AuthN-vs-AuthZ.md)).

---

## How it works (and why it's secure)

The cookie's contents (the claims) are **protected** by ASP.NET Core's **Data Protection** ([08-DataProtection.md](08-DataProtection.md)) — encrypted and signed with the app's key ring. So:
- The client **can't read or tamper** with the claims (it's encrypted, not just signed).
- The cookie is **opaque** to the browser — unlike a JWT, the client can't decode it.
- `HttpOnly` prevents JavaScript from accessing it (mitigating XSS theft); `Secure` restricts it to HTTPS; `SameSite` mitigates CSRF.

This is a key contrast with JWTs: a cookie is **encrypted server-side state-reference** (or encrypted claims) tied to the issuing app's keys, while a JWT is a **client-readable, self-contained** token. Cookies are inherently **revocable** server-side (you control the session); JWTs aren't easily ([03-JWT.md](03-JWT.md)).

---

## Cookie auth vs JWT bearer — choosing

| | Cookie auth | JWT bearer |
|---|---|---|
| Carrier | cookie (auto-sent by browser) | `Authorization` header (client adds it) |
| Best for | **server-rendered web apps** (same-origin browser) | **APIs, SPAs, mobile, microservices** |
| Client reads claims | no (encrypted) | yes (encoded) |
| Revocation | easy (server controls the session) | hard (stateless) |
| CSRF risk | yes (cookies auto-sent → need antiforgery) | no (header not auto-sent) |
| Cross-origin / cross-service | awkward | natural |

Use **cookies** when a browser talks to a single server-rendered app on the same origin — they're simpler, revocable, and the framework (Razor Pages/MVC) handles them with built-in antiforgery. Use **JWT bearer** for APIs, SPAs, mobile, and cross-service calls. (A SPA + API can use either — cookies via same-site setups, or tokens; cookies avoid storing tokens in JS-accessible storage, a real XSS consideration.)

---

## The CSRF caveat

Because the browser sends the auth cookie **automatically** on every request to the origin (including requests triggered by other sites), cookie auth is vulnerable to **Cross-Site Request Forgery (CSRF)** — a malicious site can cause the browser to make an authenticated request. Mitigations:
- **Antiforgery tokens** ([09-Antiforgery.md](09-Antiforgery.md)) — Razor Pages/MVC forms include and validate a token automatically; APIs accepting cookie auth need explicit antiforgery.
- **`SameSite` cookie attribute** (`Lax`/`Strict`) — the browser doesn't send the cookie on cross-site requests (a strong modern defense).

JWT bearer auth doesn't have this problem (the `Authorization` header isn't auto-sent cross-site), which is one reason tokens are preferred for APIs. With cookies, **always** use antiforgery + `SameSite`.

---

## Sliding expiration & sign-out

```csharp
o.ExpireTimeSpan = TimeSpan.FromHours(8);
o.SlidingExpiration = true;   // each request within the window extends the expiry (keeps active users logged in)
```

- **Absolute** expiry — the session ends at a fixed time. **Sliding** expiry — activity extends it (active users stay logged in; idle ones time out).
- **Sign-out** (`SignOutAsync`) clears the cookie. Because cookies reference server-controllable session state, you can also invalidate sessions server-side (e.g., on password change) — revocation that JWTs lack.

---

## Common gotchas

### Using cookies for an API consumed cross-origin

Cookies are awkward cross-origin and carry CSRF risk. Use JWT bearer for APIs/SPAs/mobile/cross-service.

### Forgetting CSRF protection with cookie auth

Auto-sent cookies enable CSRF. Always use antiforgery tokens + `SameSite`. (Razor Pages does antiforgery automatically; don't disable it.)

### Not setting `HttpOnly`/`Secure`/`SameSite`

A cookie readable by JS (no `HttpOnly`) is stealable via XSS; without `Secure` it leaks over HTTP; without `SameSite` it's CSRF-prone. Set all three.

### Storing too much in the cookie

The cookie carries the claims — a huge claims set bloats every request. Keep claims minimal; look up extra data server-side if needed.

### Multi-instance key ring not shared

Cookie protection uses the Data Protection key ring; across instances the keys must be **shared** (persisted to a common store) or one instance can't decrypt another's cookies → users logged out on instance switch ([08-DataProtection.md](08-DataProtection.md)).

---

## Summary

- **Cookie authentication** stores the user's identity in an **encrypted, signed cookie** (via Data Protection) that the browser auto-sends — the natural fit for **server-rendered web apps** (MVC/Razor Pages/Blazor Server).
- Unlike JWTs, the cookie is **opaque to the client** (encrypted), and sessions are **revocable** server-side; set `HttpOnly` (XSS), `Secure` (HTTPS), and `SameSite` (CSRF).
- Use **cookies** for same-origin browser apps; **JWT bearer** for APIs/SPAs/mobile/microservices.
- Cookies carry **CSRF risk** (auto-sent) → always use **antiforgery tokens + `SameSite`** ([09-Antiforgery.md](09-Antiforgery.md)); JWTs avoid this.
- Use sliding expiration for active users; share the **Data Protection key ring** across instances so cookies validate everywhere ([08-DataProtection.md](08-DataProtection.md)).

→ Next: [06-Authorization.md](06-Authorization.md)
