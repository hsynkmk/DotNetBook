# ASP.NET Core Identity

## A full membership system

ASP.NET Core Identity is a complete user-management framework: it stores users, hashes passwords, manages roles and claims, and handles email confirmation, password reset, two-factor auth, and account lockout. When your app **owns** its user accounts (registration, login, password management), Identity gives you the secure, battle-tested machinery so you don't build (and get wrong) password hashing and account flows yourself.

```csharp
builder.Services.AddDbContext<AppDbContext>(o => o.UseNpgsql(cs));
builder.Services.AddIdentityCore<AppUser>(o => {
        o.Password.RequiredLength = 12;
        o.Lockout.MaxFailedAccessAttempts = 5;
        o.SignIn.RequireConfirmedEmail = true;
    })
    .AddRoles<IdentityRole>()
    .AddEntityFrameworkStores<AppDbContext>();   // store users/roles in the DB via EF Core

public class AppUser : IdentityUser { public string? DisplayName { get; set; } }   // extend the user
public class AppDbContext(DbContextOptions o) : IdentityDbContext<AppUser>(o) { }
```

`IdentityDbContext` adds the user/role/claim tables; `IdentityUser` is the base user (extend it with your fields). Identity manages the schema via EF migrations ([Ch05 §05](../05-EFCore/05-Migrations.md)).

---

## Don't roll your own auth

The cardinal rule: **don't build your own user/password system.** Secure account management is full of subtle, security-critical details that are easy to get catastrophically wrong:
- **Password hashing** — must be a slow, salted, per-user KDF (PBKDF2/bcrypt/Argon2), not a fast hash like SHA-256 ([Ch02 §11](../02-BCL/11-Cryptography.md)). Identity uses PBKDF2 with proper salting and iteration counts, and rehashes on upgrade.
- **Account lockout** — throttle brute-force attempts.
- **Secure token generation** — for password reset / email confirmation (cryptographically random, time-limited, single-use).
- **Timing-safe comparisons**, **2FA**, **password policies**.

Identity implements all of this correctly. Reimplementing it risks plaintext/weak-hash storage, timing attacks, token-guessing, and brute-force vulnerabilities. Use Identity (or an external IdP — [04-OAuth-OIDC.md](04-OAuth-OIDC.md)) for account management.

---

## Users, passwords, sign-in

The core managers do the work:

```csharp
public class AccountService(UserManager<AppUser> users, SignInManager<AppUser> signIn) {
    public async Task<IdentityResult> RegisterAsync(string email, string password) {
        var user = new AppUser { UserName = email, Email = email };
        return await users.CreateAsync(user, password);   // hashes the password securely, validates policy
    }

    public async Task<bool> LoginAsync(string email, string password) {
        var result = await signIn.PasswordSignInAsync(email, password,
            isPersistent: false, lockoutOnFailure: true);   // checks hash, enforces lockout
        return result.Succeeded;   // also: result.IsLockedOut, RequiresTwoFactor, IsNotAllowed
    }
}
```

- **`UserManager<TUser>`** — create/find/update users, manage passwords, roles, claims, tokens, 2FA.
- **`SignInManager<TUser>`** — sign in/out, validate credentials (with lockout), handle 2FA and external logins.
- `IdentityResult` reports success or detailed errors (password too weak, email taken, etc.).

You never touch raw password hashes — `CreateAsync`/`PasswordSignInAsync` handle hashing and verification securely.

---

## Roles and claims

Identity supports both **roles** (coarse-grained groups) and **claims** (fine-grained attributes) — the inputs to authorization ([06-Authorization.md](06-Authorization.md), [07-Claims.md](07-Claims.md)):

```csharp
await users.AddToRoleAsync(user, "Admin");                              // role
await users.AddClaimAsync(user, new Claim("department", "Engineering")); // claim
await users.AddClaimAsync(user, new Claim("permission", "orders:write"));

// These flow into the ClaimsPrincipal at sign-in, enabling [Authorize(Roles="Admin")] / policies
```

Prefer **claims/policies** over roles for anything beyond coarse grouping — claims are more flexible (e.g., a `permission` claim per capability) and policy-based authorization is the recommended model. Roles are a special kind of claim (`ClaimTypes.Role`).

---

## Identity + tokens vs Identity + cookies

Identity establishes *who the user is*; how that identity is then carried per request depends on the app type:

- **Server-rendered web apps** → **cookie** authentication ([05-Cookies.md](05-Cookies.md)). `SignInManager` issues an auth cookie; the browser sends it automatically. Use `AddDefaultIdentity` / `AddIdentity` (which wires up cookies + UI).
- **APIs / SPAs / mobile** → **token** authentication. Identity manages the users, but you issue **JWTs** ([03-JWT.md](03-JWT.md)) (or use Identity's API endpoints / a token server). .NET 8 added **`MapIdentityApi`** — built-in token-based Identity endpoints (register/login/refresh) for APIs:

```csharp
// .NET 8+ — token-based Identity endpoints for APIs (bearer tokens, no cookies)
builder.Services.AddIdentityApiEndpoints<AppUser>().AddEntityFrameworkStores<AppDbContext>();
app.MapIdentityApi<AppUser>();   // /register, /login (returns a bearer token), /refresh, etc.
```

Choose based on the client: cookies for browser-rendered apps, tokens for APIs/SPAs/mobile.

---

## When to use Identity vs an external IdP

| Approach | Use when |
|---|---|
| **ASP.NET Core Identity** | your app owns accounts; you want full control of the user store and login flows |
| **External IdP** (Entra ID, Auth0, Okta, Google) via OIDC | you want SSO, social login, or to **outsource** auth entirely ([04-OAuth-OIDC.md](04-OAuth-OIDC.md)) |
| **Duende IdentityServer** / OpenIddict | you need to **be** an OAuth/OIDC provider (issue tokens to multiple clients) |

A major architectural decision: **own** authentication (Identity) or **delegate** it (an external IdP via OpenID Connect). Delegating to an IdP offloads the hardest security work (account management, MFA, threat detection, compliance) and enables SSO/social login — increasingly the default for new apps. Use Identity when you specifically need to own the user store and flows. Either way, **don't hand-roll** password handling.

---

## Common gotchas

### Rolling your own user/password system

The biggest security mistake — fast/unsalted password hashing, no lockout, guessable reset tokens. Use Identity (or an external IdP). It gets the hard parts right.

### Weak password-hashing if bypassing Identity

If you store passwords yourself, never use a plain/fast hash (SHA-256/MD5) — use a slow salted KDF. Better: don't store passwords; use Identity or an IdP.

### Roles for fine-grained permissions

Roles get unwieldy for granular permissions (a role per capability). Use **claims + policy-based authorization** for fine-grained control.

### Not confirming email / no lockout

Skipping email confirmation and account lockout invites fake accounts and brute-force attacks. Enable `RequireConfirmedEmail` and lockout.

### Cookies for an API / tokens for a server-rendered app

Match the carrier to the client: cookies for browser apps, bearer tokens for APIs/SPAs/mobile. Mixing them awkwardly causes auth headaches.

### Storing secrets/PII insecurely

Identity hashes passwords, but *your* extra fields (PII) need appropriate protection (encryption at rest, access control) — and never log them.

---

## Summary

- **ASP.NET Core Identity** is a full membership system (users, secure password hashing, roles, claims, 2FA, lockout, email confirmation, reset tokens) for apps that **own** their accounts — backed by EF Core stores.
- **Don't roll your own** auth — Identity gets the security-critical parts right (PBKDF2 salted hashing, lockout, secure tokens, timing-safe checks); reimplementing risks serious vulnerabilities.
- Use **`UserManager`/`SignInManager`** for account/login operations; manage **roles** (coarse) and **claims** (fine-grained — prefer for authorization).
- Carry identity via **cookies** (server-rendered apps) or **tokens/JWT** (APIs/SPAs/mobile — `MapIdentityApi` for built-in token endpoints).
- Decide: **own** auth (Identity) vs **delegate** to an external IdP via OIDC ([04-OAuth-OIDC.md](04-OAuth-OIDC.md)) — delegating offloads the hardest security work and enables SSO. Enable email confirmation + lockout; prefer claims/policies over roles.

→ Next: [03-JWT.md](03-JWT.md)
