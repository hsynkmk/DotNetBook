# Data Protection

## Encrypting app-managed secrets safely

The **Data Protection** API (`IDataProtector`) is ASP.NET Core's built-in system for **encrypting and signing** short pieces of data the app needs to protect — auth cookies, antiforgery tokens, password-reset/email-confirmation tokens, TempData, and any small app secret. It manages encryption keys (a **key ring**) with automatic rotation, so you get authenticated encryption without touching raw crypto ([Ch02 §11](../02-BCL/11-Cryptography.md)).

```csharp
public class TokenService(IDataProtectionProvider provider) {
    private readonly IDataProtector _protector = provider.CreateProtector("MyApp.ResetTokens.v1");

    public string Protect(string data) => _protector.Protect(data);          // encrypt + sign
    public string Unprotect(string token) => _protector.Unprotect(token);     // verify + decrypt (throws if tampered)
}
```

`Protect` encrypts and authenticates the payload; `Unprotect` verifies and decrypts (throwing if it was tampered with or protected by a different purpose/app). You never handle keys or algorithms directly.

---

## Purposes — isolation by intent

The string passed to `CreateProtector` is a **purpose** — it scopes the protector so data protected for one purpose can't be unprotected by another, even within the same app:

```csharp
var resetProtector = provider.CreateProtector("PasswordReset");
var emailProtector = provider.CreateProtector("EmailConfirm");
// A token from resetProtector.Protect(...) CANNOT be unprotected by emailProtector — purposes differ
```

Purposes provide **cryptographic isolation**: a password-reset token can't be repurposed as an email-confirmation token (or vice versa), limiting blast radius if one flow is exploited. Use distinct, stable purpose strings per use case (and version them, e.g., `"...v1"`, if you need to evolve). This is a key safety feature — always use meaningful, separate purposes.

---

## What uses Data Protection (mostly invisibly)

You often don't call `IDataProtector` directly — the framework uses it under the hood:
- **Authentication cookies** ([05-Cookies.md](05-Cookies.md)) — the cookie's claims are encrypted with the key ring (why a cookie is opaque to the client).
- **Antiforgery tokens** ([09-Antiforgery.md](09-Antiforgery.md)).
- **Identity tokens** — password reset, email confirmation, 2FA tokens ([02-AspNetIdentity.md](02-AspNetIdentity.md)).
- **TempData** (cookie-based), session, and other transient protected state.

So Data Protection underpins much of the auth stack. Its configuration (the key ring) directly affects whether cookies/tokens work — especially across multiple instances (below).

---

## The key ring & persistence (the critical multi-instance issue)

Data Protection generates and rotates a **key ring** (a set of keys, with one active for new protection and older ones retained to unprotect existing data). **Where the key ring is stored matters enormously in multi-instance deployments**:

```csharp
// ✗ — default in some hosts: keys stored locally / in memory → each instance has DIFFERENT keys
//     → instance A can't decrypt instance B's cookies → users randomly logged out; tokens fail

// ✓ — persist keys to a SHARED store so all instances use the same key ring
builder.Services.AddDataProtection()
    .PersistKeysToFileSystem(new DirectoryInfo("/shared/keys"))     // shared volume
    .ProtectKeysWithCertificate(cert);                               // encrypt the keys at rest
// or to a database / Redis / Azure Blob Storage:
builder.Services.AddDataProtection().PersistKeysToAzureBlobStorage(blobUri).ProtectKeysWithAzureKeyVault(...);
```

The #1 Data Protection bug: in a load-balanced/scaled-out app (multiple instances, Kubernetes pods, container restarts), if the key ring **isn't shared**, each instance generates its own keys → an auth cookie or token created by instance A **can't be validated by instance B** → users are logged out on instance switch, password-reset links fail, antiforgery breaks. **Persist the key ring to a shared, durable store** (file share, DB, Redis, Azure Blob) so all instances share keys — and **protect the keys at rest** (certificate / Key Vault) so the persisted keys themselves are encrypted.

---

## Key rotation & lifetime

```csharp
builder.Services.AddDataProtection()
    .SetDefaultKeyLifetime(TimeSpan.FromDays(90));   // new keys auto-created; old ones retained to read existing data
```

Data Protection **rotates keys automatically** — periodically a new key becomes active for *new* protection, while old keys are kept so existing protected data (cookies, tokens) still unprotects until it expires. You don't manually rotate; you can tune the lifetime. This automatic rotation is good security hygiene (limits the data protected by any single key) and is handled for you — another reason to use Data Protection rather than a hand-rolled key.

---

## Application isolation

By default, each app has its own key ring (isolated by application name). If you intentionally want **multiple apps to share** protected data (e.g., a web app and a separate API decrypting the same cookie), set a common application name:

```csharp
builder.Services.AddDataProtection().SetApplicationName("MySharedApp");   // share keys across apps
```

Conversely, the default isolation prevents one app from unprotecting another's data. Set a shared application name **only** when you deliberately want cross-app sharing (and they share the key store).

---

## Common gotchas

### Key ring not shared across instances

The #1 bug — each instance with its own keys can't validate others' cookies/tokens → random logouts, failing reset links, broken antiforgery. **Persist keys to a shared store** (file share/DB/Redis/Blob).

### Keys not protected at rest

A persisted key ring in plaintext is a serious exposure (whoever reads it can forge cookies/tokens). Encrypt keys at rest (`ProtectKeysWithCertificate`/Key Vault).

### Reusing one purpose for everything

A single purpose loses the isolation benefit. Use **distinct purposes** per use case so tokens can't be cross-used.

### Hand-rolling encryption instead of using Data Protection

Reaching for raw `AesGcm` for cookies/tokens reinvents (often insecurely) what Data Protection does correctly with key management and rotation. Use `IDataProtector` for app-managed protected data.

### Expecting Unprotect to "just work" after key loss

If the key that protected data is gone (key ring reset, not persisted), `Unprotect` fails — existing cookies/tokens become invalid. Persist and back up the key ring.

### Using Data Protection for long-term/large data

It's designed for **short-lived, small** payloads (tokens, cookies), not long-term storage of large data. For that, use proper encryption + key management ([Ch02 §11](../02-BCL/11-Cryptography.md)).

---

## Summary

- **Data Protection** (`IDataProtector`) provides authenticated **encryption + signing** for short app-managed secrets (auth cookies, antiforgery/reset/confirmation tokens, TempData) with an automatically-rotated **key ring** — no raw crypto.
- **`Protect`/`Unprotect`** with a **purpose** string that cryptographically isolates uses (a reset token can't be repurposed as a confirmation token) — use distinct, stable purposes.
- It **underpins the auth stack** (cookies, antiforgery, Identity tokens); the **critical** config is **persisting the key ring to a shared, durable store** in multi-instance deployments (else instances can't validate each other's cookies/tokens → random logouts) — and **encrypting keys at rest**.
- Keys **rotate automatically** (old keys retained to read existing data); apps are **isolated** by default (set a shared application name only to deliberately share).
- Use it for short, small protected data; prefer it over hand-rolled encryption for cookies/tokens.

→ Next: [09-Antiforgery.md](09-Antiforgery.md)
