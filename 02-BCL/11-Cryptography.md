# Cryptography

## What the BCL gives you (and the rule above all rules)

`System.Security.Cryptography` provides secure random generation, hashing, HMACs, symmetric and asymmetric encryption, key derivation, and digital signatures — production-grade implementations backed by the OS crypto libraries.

> **The cardinal rule: don't invent crypto.** Use these well-vetted primitives at the lowest level, and prefer **high-level, hard-to-misuse APIs** where they exist. Most crypto vulnerabilities come from misusing correct primitives (wrong mode, reused nonce, no authentication, homemade password hashing). When possible, use a higher-level library or platform feature (ASP.NET Core Data Protection — [Ch10](../10-Identity/README.md)) instead of assembling primitives yourself.

---

## Secure random — `RandomNumberGenerator`

For **anything secret** (keys, salts, tokens, nonces), use the CSPRNG, never `Random`:

```csharp
using System.Security.Cryptography;

byte[] key = RandomNumberGenerator.GetBytes(32);            // 256-bit key
byte[] salt = RandomNumberGenerator.GetBytes(16);
int code = RandomNumberGenerator.GetInt32(0, 1_000_000);    // unbiased secure int
string token = Convert.ToHexString(RandomNumberGenerator.GetBytes(16));   // 128-bit token
string urlToken = Convert.ToBase64String(RandomNumberGenerator.GetBytes(32));
```

`Random`/`Random.Shared` is **predictable** — fine for games/sims/tests, a vulnerability for anything secret ([02-Numerics.md](02-Numerics.md)). This is the most common crypto mistake.

---

## Hashing — and what hashes are NOT for

```csharp
// One-shot hashing (SHA-256/384/512)
byte[] hash = SHA256.HashData(data);                       // static, no instance needed (.NET 5+)
string hex = Convert.ToHexString(SHA256.HashData(bytes));

// Hashing a stream
byte[] fileHash = await SHA256.HashDataAsync(fileStream, ct);
```

Use cryptographic hashes (SHA-256+) for **integrity** (detect tampering), content addressing, and as building blocks. **Do NOT use a plain hash for passwords** — SHA-256 is fast, so an attacker can brute-force billions/sec. Passwords need a deliberately *slow* KDF (below). Avoid **MD5/SHA-1** for security (broken/weak) — only acceptable for non-security checksums.

---

## HMAC — keyed authentication

A plain hash proves integrity only if the hash itself is protected; an **HMAC** uses a secret key so only key-holders can produce/verify it — for message authentication, webhook signatures, API request signing:

```csharp
byte[] mac = HMACSHA256.HashData(key, message);

// Verify in CONSTANT TIME to avoid timing attacks:
byte[] expected = HMACSHA256.HashData(key, message);
bool valid = CryptographicOperations.FixedTimeEquals(expected, providedMac);   // ← not ==
```

**Always compare MACs/secrets with `CryptographicOperations.FixedTimeEquals`** — a normal `==`/`SequenceEqual` short-circuits on the first mismatched byte, leaking timing that lets an attacker forge the value byte-by-byte. This subtlety is a frequent real-world bug.

---

## Password hashing — use a slow KDF

Never store passwords as plain or fast-hashed. Use a **password-based KDF** designed to be slow and salted:

```csharp
// PBKDF2 (built into the BCL)
byte[] salt = RandomNumberGenerator.GetBytes(16);
byte[] hash = Rfc2898DeriveBytes.Pbkdf2(
    password: password,
    salt: salt,
    iterations: 600_000,                 // high cost — tune to ~hundreds of ms
    hashAlgorithm: HashAlgorithmName.SHA256,
    outputLength: 32);
// store: salt + iterations + hash
```

PBKDF2 is in the box. Even stronger options (Argon2id, scrypt, bcrypt) come via libraries — and **ASP.NET Core Identity** handles password hashing for you correctly ([Ch10](../10-Identity/README.md)), which is what you should usually use rather than rolling your own. The key idea: passwords need a **deliberately slow, salted, per-user** derivation, not a fast hash.

---

## Symmetric encryption — prefer AEAD

For encrypting data with a shared key, use **authenticated encryption** (AEAD) so ciphertext can't be tampered with undetected:

```csharp
// AES-GCM — authenticated encryption (preferred)
byte[] key = RandomNumberGenerator.GetBytes(32);           // 256-bit
byte[] nonce = RandomNumberGenerator.GetBytes(AesGcm.NonceByteSizes.MaxSize);  // UNIQUE per message!
byte[] ciphertext = new byte[plaintext.Length];
byte[] tag = new byte[AesGcm.TagByteSizes.MaxSize];

using var aes = new AesGcm(key, tag.Length);
aes.Encrypt(nonce, plaintext, ciphertext, tag);            // tag authenticates the ciphertext
// decrypt: aes.Decrypt(nonce, ciphertext, tag, decrypted) — throws if tampered
```

**Use AES-GCM (or ChaCha20-Poly1305)** — they encrypt *and* authenticate. Plain `AES-CBC` without a MAC is vulnerable to tampering/padding-oracle attacks. The **nonce must be unique per message** under a given key (reusing a GCM nonce is catastrophic — it breaks confidentiality). Generate a fresh random nonce each time and store it alongside the ciphertext.

> Even better: for app-data protection (cookies, tokens, short secrets), use **ASP.NET Core Data Protection** — it manages keys, rotation, and AEAD correctly so you don't touch `AesGcm` directly ([Ch10](../10-Identity/README.md)).

---

## Asymmetric crypto & signatures

For key exchange, signatures, and certificates — RSA and ECDSA:

```csharp
// ECDSA signature (sign with private key, verify with public)
using var ecdsa = ECDsa.Create(ECCurve.NamedCurves.nistP256);
byte[] signature = ecdsa.SignData(data, HashAlgorithmName.SHA256);
bool ok = ecdsa.VerifyData(data, signature, HashAlgorithmName.SHA256);

// RSA
using var rsa = RSA.Create(2048);
byte[] sig = rsa.SignData(data, HashAlgorithmName.SHA256, RSASignaturePadding.Pkcs1);
```

Use **ECDSA/ECDH** (smaller, faster) for new systems where possible; RSA where required for interop. These underpin TLS, JWT signing, and certificates. **X509Certificate2** loads/uses certificates for TLS and signing.

### Post-quantum (.NET 10)

.NET 10 adds support for **post-quantum algorithms** (ML-KEM for key encapsulation, ML-DSA for signatures) via `System.Security.Cryptography`, as the industry begins migrating to quantum-resistant crypto. Relevant for long-lived secrets that must resist future quantum attacks.

---

## Clearing secrets from memory

Sensitive bytes linger in memory until GC. Zero them when done:

```csharp
byte[] key = RandomNumberGenerator.GetBytes(32);
try { UseKey(key); }
finally { CryptographicOperations.ZeroMemory(key); }   // clear, don't leave the key in memory
```

For maximum protection (rare), keep secrets off the GC heap entirely. For most apps, `ZeroMemory` after use plus not logging secrets is sufficient.

---

## Common gotchas

### Using `Random` for secrets

Predictable → forgeable tokens/keys. Use `RandomNumberGenerator`.

### Comparing secrets/MACs with `==`

Leaks timing → byte-by-byte forgery. Use `CryptographicOperations.FixedTimeEquals`.

### Fast-hashing passwords (SHA-256, MD5)

Brute-forceable. Use a slow salted KDF (PBKDF2/Argon2) or ASP.NET Core Identity.

### Unauthenticated encryption (AES-CBC, no MAC)

Tampering/padding-oracle vulnerable. Use AEAD (AES-GCM / ChaCha20-Poly1305).

### Reusing a GCM nonce

Catastrophic — breaks confidentiality. Fresh random nonce per message.

### MD5/SHA-1 for security

Broken/weak. Use SHA-256+ (or only for non-security checksums).

### Rolling your own crypto / key management

The biggest risk. Prefer ASP.NET Core Data Protection / Identity / a vetted library over assembling primitives.

---

## Summary

- **Don't invent crypto** — use the BCL primitives correctly, and prefer high-level APIs (ASP.NET Core **Data Protection** / **Identity**) over hand-assembling them.
- **Secure random** = `RandomNumberGenerator` (never `Random`) for keys/salts/tokens/nonces.
- **Hashing** (SHA-256+) for integrity, **HMAC** for keyed authentication — compare with **`FixedTimeEquals`** (constant time).
- **Passwords** need a slow salted **KDF** (PBKDF2/Argon2), never a fast hash.
- **Symmetric** = AEAD (**AES-GCM**/ChaCha20-Poly1305) with a **unique nonce per message**; **asymmetric** = ECDSA/RSA for signatures/key exchange.
- **Zero secrets** with `CryptographicOperations.ZeroMemory`; .NET 10 adds **post-quantum** algorithms.

→ Next: [12-Threading.md](12-Threading.md)
