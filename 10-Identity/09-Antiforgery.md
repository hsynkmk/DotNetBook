# Antiforgery (CSRF Protection)

## Stopping cross-site request forgery

**CSRF** (Cross-Site Request Forgery) tricks a logged-in user's browser into making an unwanted authenticated request to your app. Because browsers **automatically attach cookies** to requests for your origin, a malicious site can cause the user's browser to submit a state-changing request (transfer money, change email) using the victim's existing session — without the attacker ever seeing the credentials. **Antiforgery tokens** defend against this and are built into ASP.NET Core.

```
Victim is logged into bank.com (auth cookie set)
Victim visits evil.com → evil.com auto-submits a hidden form POST to bank.com/transfer
Browser attaches bank.com's auth cookie automatically → request looks authenticated → money transferred (CSRF!)
```

CSRF specifically affects **cookie-based** auth ([05-Cookies.md](05-Cookies.md)), because cookies are auto-sent cross-site. **Token (JWT bearer) auth is not vulnerable** — the `Authorization` header isn't auto-attached by the browser.

---

## How antiforgery tokens work

The defense: require a **secret token** on state-changing requests that an attacker's site **can't know or read** (due to the same-origin policy):

```
1. The server embeds an antiforgery token in the form (a hidden field) AND sets a paired cookie token
2. On submit, the browser sends BOTH the form token and the cookie token
3. The server validates they match (and are cryptographically valid)
4. A cross-site attacker can't read your page to get the form token → can't forge a valid request
```

This is the **synchronizer/double-submit token** pattern: the request must carry a token that only a page served *by your origin* could contain. The attacker's cross-origin page can't read your token (same-origin policy blocks it), so it can't construct a valid request. The tokens are protected via Data Protection ([08-DataProtection.md](08-DataProtection.md)).

---

## Built-in in Razor Pages & MVC

For server-rendered apps, antiforgery is **largely automatic**:

```razor
@* Razor Pages: the form tag helper injects the antiforgery token automatically *@
<form method="post">
    @* hidden __RequestVerificationToken field added automatically *@
    <button type="submit">Save</button>
</form>
```

- **Razor Pages** validate antiforgery tokens on POST **by default** — the `<form>` tag helper injects the token; the framework checks it. (Don't disable this.)
- **MVC** — use `[ValidateAntiForgeryToken]` (or `[AutoValidateAntiforgeryToken]` globally) on POST actions; the `<form>`/`@Html.AntiForgeryToken()` emits the token.

```csharp
builder.Services.AddControllersWithViews(o => o.Filters.Add<AutoValidateAntiforgeryTokenAttribute>());
//   validate antiforgery on all unsafe (POST/PUT/DELETE) requests automatically
```

This built-in protection is a major reason server-rendered forms in ASP.NET Core are safe by default — the framework handles CSRF for you.

---

## SameSite cookies — the modern complement

The **`SameSite`** cookie attribute is a browser-level CSRF defense: it controls whether cookies are sent on **cross-site** requests:

```csharp
o.Cookie.SameSite = SameSiteMode.Lax;     // (or Strict) — don't send the cookie on cross-site requests
```

- **`Strict`** — the cookie is never sent on cross-site requests (strongest, but can break legitimate cross-site navigation/links to your app).
- **`Lax`** (a common default) — sent on top-level cross-site **navigations** (clicking a link) but not on cross-site **form posts / subresource requests** — blocking the classic CSRF POST while allowing normal navigation.
- **`None`** — sent cross-site (requires `Secure`); needed for some legitimate cross-site scenarios.

`SameSite=Lax`/`Strict` blocks the cross-site auto-submitted request that CSRF relies on. Use it **alongside** antiforgery tokens (defense in depth) — `SameSite` is broad browser-enforced protection, antiforgery tokens are app-level validation; together they're robust.

---

## APIs and SPAs

- **Token (bearer) APIs** — not CSRF-vulnerable (the `Authorization` header isn't auto-sent cross-site), so they generally **don't need antiforgery tokens**. This is one reason tokens are preferred for APIs ([05-Cookies.md](05-Cookies.md)).
- **Cookie-authenticated APIs / SPAs** — *are* CSRF-vulnerable and **do** need antiforgery. A SPA reads the antiforgery token (e.g., from a cookie the server sets) and sends it in a header on requests; the server validates it. ASP.NET Core supports header-based antiforgery for this.

```csharp
// SPA scenario: expose the token so the JS client can send it in a header
builder.Services.AddAntiforgery(o => o.HeaderName = "X-XSRF-TOKEN");
```

The rule: **cookie auth → need CSRF protection; token auth → don't.** Choose based on how the client authenticates.

---

## Common gotchas

### Disabling antiforgery in Razor Pages

It's on by default for a reason. Disabling it (or stripping the `<form>` tag helper's token) opens CSRF holes. Keep it enabled.

### Thinking token APIs need antiforgery

Bearer-token APIs aren't CSRF-vulnerable (header not auto-sent). Adding antiforgery there is unnecessary; the concern is cookie auth.

### Forgetting antiforgery on a cookie-authenticated API/SPA

Cookie-authenticated APIs *are* vulnerable. Use header-based antiforgery (or `SameSite` + token) — don't assume "it's an API so it's safe."

### Relying on `SameSite` alone (or antiforgery alone)

`SameSite` depends on browser support and has edge cases; antiforgery tokens are app-enforced. Use **both** for defense in depth.

### Confusing CSRF with XSS

CSRF (forged cross-site requests) and XSS (injecting scripts) are different attacks. Antiforgery stops CSRF; XSS needs output encoding/CSP and `HttpOnly` cookies. (XSS can defeat antiforgery by reading the token, so prevent XSS too.)

### `SameSite=Strict` breaking legitimate flows

`Strict` blocks the cookie even on legitimate cross-site navigation to your app (e.g., following an emailed link logs them out). `Lax` is usually the right balance.

---

## Summary

- **CSRF** tricks a logged-in user's browser into making unwanted authenticated requests, exploiting that **cookies are auto-sent cross-site** — it affects **cookie auth**, not token (bearer) auth.
- **Antiforgery tokens** defend it: a secret token embedded in the page (and a paired cookie) that a cross-origin attacker can't read (same-origin policy) → forged requests lack a valid token. Protected via Data Protection.
- **Razor Pages validate antiforgery by default**; MVC uses `[ValidateAntiForgeryToken]`/`[AutoValidateAntiforgeryToken]` — keep it enabled.
- **`SameSite` cookies** (`Lax`/`Strict`) are a browser-level complement that blocks cross-site cookie sending — use **alongside** antiforgery tokens (defense in depth).
- **Cookie auth needs CSRF protection; bearer-token APIs don't** (header isn't auto-sent). Cookie-authenticated SPAs/APIs need header-based antiforgery.

→ Next: [10-Cryptography.md](10-Cryptography.md)
