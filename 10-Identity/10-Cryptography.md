# Cryptography (Security Context)

## Don't roll your own — use the right tool

Cryptography in a security context comes down to one rule: **don't invent it, and don't assemble primitives when a higher-level tool exists.** The BCL crypto primitives are covered in **[Chapter 02 §11](../02-BCL/11-Cryptography.md)** (and CSharpBook's memory chapter); this file is the **security decision guide** — which tool for which job, layered from "highest-level, hardest to misuse" down to raw primitives.

```
Need to protect a cookie/token/short app secret?   → IDataProtection ([08-DataProtection.md])
Need to manage users/passwords?                    → ASP.NET Core Identity ([02-AspNetIdentity.md])
Need auth tokens / SSO?                             → an IdP via OIDC ([04-OAuth-OIDC.md])
Need a secure random token/key?                     → RandomNumberGenerator
Need to hash a password yourself?                   → a slow KDF (PBKDF2/Argon2) — but prefer Identity
Need to encrypt data yourself?                      → AES-GCM (AEAD) — but prefer Data Protection
```

The further **up** this list you can stay, the safer — higher-level tools handle key management, rotation, salting, and timing-safe comparison correctly, which are exactly the things hand-rolled crypto gets wrong.

---

## Prefer the high-level tools

For almost every security need in a .NET app, a purpose-built tool already exists:

| Need | Use (not raw crypto) |
|---|---|
| Protect cookies, antiforgery, reset/confirmation tokens, short secrets | **`IDataProtection`** ([08](08-DataProtection.md)) — authenticated encryption + managed, rotating keys |
| User accounts & password storage | **ASP.NET Core Identity** ([02](02-AspNetIdentity.md)) — correct salted KDF hashing, lockout, secure tokens |
| Login / SSO / token issuance | **An IdP via OIDC** ([04](04-OAuth-OIDC.md)) — outsource it entirely |
| Secrets at rest (connection strings, keys) | **A secret store** (Key Vault, etc.) + config providers ([Ch03 §07](../03-HostingAndDI/07-Configuration.md)) |

These cover the vast majority of "I need crypto" moments. Reaching for raw `AesGcm`/`SHA256`/`Rfc2898DeriveBytes` should be rare and deliberate — only when no higher-level tool fits.

---

## When you must touch primitives — the rules

If you genuinely must use the BCL primitives ([Ch02 §11](../02-BCL/11-Cryptography.md)), follow these non-negotiable rules:

```csharp
using System.Security.Cryptography;

// 1. SECURE RANDOM — never `Random` for anything secret (keys, salts, tokens, nonces)
byte[] key = RandomNumberGenerator.GetBytes(32);
string token = Convert.ToHexString(RandomNumberGenerator.GetBytes(32));   // 256-bit token

// 2. PASSWORD HASHING — a slow, salted KDF (not a fast hash like SHA-256/MD5)
byte[] salt = RandomNumberGenerator.GetBytes(16);
byte[] hash = Rfc2898DeriveBytes.Pbkdf2(password, salt, iterations: 600_000, HashAlgorithmName.SHA256, 32);
//   (better: just use Identity, which does this correctly)

// 3. ENCRYPTION — AEAD (AES-GCM), unique nonce per message
using var aes = new AesGcm(key, AesGcm.TagByteSizes.MaxSize);
var nonce = RandomNumberGenerator.GetBytes(AesGcm.NonceByteSizes.MaxSize);   // UNIQUE per message!
aes.Encrypt(nonce, plaintext, ciphertext, tag);

// 4. COMPARING SECRETS/MACs — constant time (prevent timing attacks)
bool valid = CryptographicOperations.FixedTimeEquals(expectedMac, providedMac);   // NOT ==
```

The cardinal rules (detailed in [Ch02 §11](../02-BCL/11-Cryptography.md)):
- **`RandomNumberGenerator`, never `Random`**, for anything secret.
- **Passwords need a slow salted KDF** (PBKDF2/Argon2) — never a fast hash.
- **Encryption must be AEAD** (AES-GCM/ChaCha20-Poly1305) with a **unique nonce per message** — never unauthenticated modes (raw CBC).
- **Compare secrets/MACs with `FixedTimeEquals`** (constant time) to avoid timing attacks.
- **Zero out** sensitive bytes after use (`CryptographicOperations.ZeroMemory`).

---

## Why "don't roll your own" is the theme

Cryptographic code that *looks* correct is routinely insecure in subtle ways: a fast password hash that's brute-forceable, a reused GCM nonce that breaks confidentiality, a `==` comparison that leaks timing, a predictable `Random` token, a missing salt, unmanaged key rotation. These aren't bugs you find by testing — they're security holes exploited later. The high-level tools (Data Protection, Identity, IdPs) were built and audited by experts to avoid exactly these traps. The single most important security practice is **using vetted tools instead of assembling primitives** — and, above all, never inventing your own algorithm.

---

## TLS in transit (recap)

Encrypting data **in transit** is HTTPS/TLS, not application crypto:
- Use **TLS 1.2/1.3**; let Kestrel / the reverse proxy / load balancer terminate TLS ([Ch04 §01](../04-AspNetCore/01-FirstApp.md), [Ch19](../19-Deployment/README.md)).
- **Validate certificates** for outbound calls — never disable validation in production ([Ch02 §10](../02-BCL/10-Net.md)).
- For service-to-service, **mutual TLS (mTLS)** authenticates both ends.

Don't hand-encrypt payloads to "secure" them over the network when TLS already does it correctly — apply application encryption only for data at rest or end-to-end requirements TLS doesn't cover.

---

## Post-quantum (forward-looking)

.NET 10 adds **post-quantum** algorithms (ML-KEM, ML-DSA) for key exchange and signatures, as the industry begins migrating to quantum-resistant crypto for long-lived secrets ([Ch02 §11](../02-BCL/11-Cryptography.md)). Relevant if you protect data that must remain confidential for decades. For most apps, this is on the horizon, not an immediate concern — but the BCL is ready when it matters.

---

## Common gotchas

### Hand-rolling instead of using Data Protection / Identity / an IdP

The biggest mistake. Use the high-level, audited tools for cookies/tokens (Data Protection), passwords/accounts (Identity), and login/SSO (IdP) — don't assemble primitives.

### `Random` for security

Predictable → forgeable tokens/keys. Use `RandomNumberGenerator` for anything secret.

### Fast-hashing passwords

SHA-256/MD5 are brute-forceable. Use a slow salted KDF — or Identity, which does it for you.

### Unauthenticated encryption / reused nonce

Raw CBC (no MAC) is tamperable; a reused GCM nonce breaks confidentiality. Use AEAD with a unique nonce — or Data Protection.

### `==` for comparing secrets/MACs

Leaks timing → byte-by-byte forgery. Use `CryptographicOperations.FixedTimeEquals`.

### Hand-encrypting over the network

TLS already encrypts in transit. Use HTTPS/TLS; reserve app-level encryption for data at rest or specific end-to-end needs.

---

## Summary

- The security theme: **don't roll your own crypto, and stay as high-level as possible** — higher-level tools handle the error-prone parts (key management, rotation, salting, timing-safe comparison) correctly.
- Use **`IDataProtection`** for cookies/tokens/short secrets, **ASP.NET Core Identity** for passwords/accounts, an **IdP via OIDC** for login/SSO, and a **secret store** (Key Vault) for secrets at rest — before ever touching primitives.
- If you must use primitives ([Ch02 §11](../02-BCL/11-Cryptography.md)): **`RandomNumberGenerator`** (not `Random`), **slow salted KDF** for passwords, **AEAD (AES-GCM) + unique nonce** for encryption, **`FixedTimeEquals`** for secret comparison, zero out secrets.
- Encrypt **in transit** with **TLS 1.2/1.3** (validate certs; mTLS for service-to-service) — not hand-rolled payload encryption.
- The #1 practice is **using vetted tools over assembling primitives**; .NET 10 adds post-quantum algorithms for long-lived-secret scenarios.

→ Next: [Questions.md](Questions.md)
