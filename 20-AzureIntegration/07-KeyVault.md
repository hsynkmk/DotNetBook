# Azure Key Vault

## Centralized secrets, keys, and certificates

**Azure Key Vault** is Azure's managed service for safeguarding **secrets** (connection strings, API keys, passwords), **keys** (cryptographic keys for encryption/signing), and **certificates** (TLS certs, with lifecycle management). It's the production answer to "where do secrets live?" ([Ch13 §07](../13-Configuration/07-Secrets.md)): not in the repo, not in config files, not in environment variables — in a hardened, access-controlled, audited vault, accessed via **managed identity** ([01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)) so there's **no secret needed to get the secrets**. Key Vault integrates as a **configuration provider** so vault secrets become ordinary `IConfiguration` keys.

```csharp
// Integrate as a configuration provider — secrets become config keys:
builder.Configuration.AddAzureKeyVault(
    new Uri("https://myvault.vault.azure.net/"),
    new DefaultAzureCredential());           // keyless — managed identity in prod, dev login locally

var dbConn = builder.Configuration.GetConnectionString("Db");   // value came from Key Vault
```

---

## Three kinds of objects

Key Vault stores three object types, each with its own client/API:

| Object | For | Client |
|---|---|---|
| **Secrets** | arbitrary sensitive strings (connection strings, API keys, passwords) | `SecretClient` |
| **Keys** | cryptographic keys (RSA/EC) for **encrypt/decrypt/sign/verify** — the key never leaves the vault | `KeyClient` / `CryptographyClient` |
| **Certificates** | X.509 certs with lifecycle (creation, renewal, rotation) | `CertificateClient` |

The crucial property of **keys**: the private key material **never leaves the vault** — you send data *to* the vault to encrypt/sign, so the key is never exposed to your app or extracted ([Ch10 §10](../10-Identity/10-Cryptography.md) covers cryptography). This is stronger than holding a key in app memory. **HSM-backed** vaults store keys in hardware security modules for the highest assurance.

---

## The configuration provider integration

The most common use is the **configuration provider** ([Ch13 §02](../13-Configuration/02-Providers.md)) — `AddAzureKeyVault` loads vault secrets into `IConfiguration`, so consuming code reads them by key with **no Key Vault-specific code**:

```csharp
builder.Configuration.AddAzureKeyVault(vaultUri, new DefaultAzureCredential());

// Anywhere — just configuration, the value transparently comes from the vault:
var apiKey = builder.Configuration["Payment:ApiKey"];
```

Key Vault secret names use `--` as the section separator (since `:` isn't allowed in secret names): a secret named `Payment--ApiKey` maps to config key `Payment:ApiKey`. This means you can **swap a secret's source** (appsettings → Key Vault) without changing the code that reads it — the power of the unified configuration model ([Ch13 §01](../13-Configuration/01-IConfiguration.md)).

### App Service Key Vault references

Alternatively, **App Service** ([02-AppService.md](02-AppService.md)) resolves an App Setting whose value is `@Microsoft.KeyVault(SecretUri=...)` from the vault using the app's managed identity — so the secret lives in the vault but appears as a normal App Setting, without adding the provider in code.

---

## Access via managed identity + RBAC

Key Vault access follows the chapter's theme: the app's **managed identity** ([01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)) authenticates, and **RBAC roles** (e.g., "Key Vault Secrets User" for read, "Key Vault Secrets Officer" for manage) govern what it can do. There's **no secret stored in the app** to access the vault — solving the bootstrapping problem (you don't need a secret to fetch your secrets). Grant **least-privilege** roles (a web app usually needs only *read* on secrets), and access is **audited** (every access logged) and **revocable** (remove the role assignment).

---

## Rotation and reload

Secrets get **rotated** (periodically, or after a suspected leak). Key Vault + the config provider can pick up rotations:

- Configure the Key Vault provider to **reload on an interval** ([Ch13 §04](../13-Configuration/04-Reload.md)) so a rotated secret flows in without a redeploy.
- Read rotatable secrets via **`IOptionsMonitor<T>`** (live) rather than `IOptions<T>` (read-once — [Ch13 §05](../13-Configuration/05-Options.md)) so the new value is reflected.
- Don't reload too aggressively — vault calls have latency/cost.

Certificates similarly support **auto-renewal/rotation** via the vault's lifecycle management, so TLS certs renew without manual steps.

---

## Common gotchas

### Storing a secret to access Key Vault

If you authenticate to the vault with a client secret stored in config, you've just moved the problem. Use **managed identity** (`DefaultAzureCredential`) so there's **no secret** to store ([01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)).

### Missing RBAC role

Managed identity authenticates, but without the right **Key Vault role** (e.g., "Key Vault Secrets User") it gets **403**. Grant least-privilege access ([01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)).

### `:` in secret names

Secret names can't contain `:`; the provider maps `--` to `:`. Name secrets like `Section--Key` to bind to `Section:Key`.

### `IOptions<T>` for rotatable secrets

`IOptions<T>` reads once and won't reflect a rotated secret. Use `IOptionsMonitor<T>` + provider reload for secrets that rotate ([Ch13 §05](../13-Configuration/05-Options.md)).

### Extracting keys instead of using vault crypto

For cryptographic keys, do **encrypt/sign in the vault** (`CryptographyClient`) so the key never leaves it — don't download the key into your app (defeats the protection).

---

## Summary

- **Azure Key Vault** safeguards **secrets**, **keys** (crypto — private material **never leaves the vault**, optionally HSM-backed), and **certificates** (with lifecycle/auto-renewal) — the production home for secrets ([Ch13 §07](../13-Configuration/07-Secrets.md)), not the repo/config/env.
- It integrates as a **configuration provider** (`AddAzureKeyVault`) so vault secrets become ordinary `IConfiguration` keys (secret `A--B` → key `A:B`) — consuming code reads them unchanged; **App Service Key Vault references** offer a code-free alternative.
- Access uses **managed identity + RBAC** (`DefaultAzureCredential`, least-privilege roles) — **no secret stored to access the vault** (solves the bootstrapping problem), audited and revocable.
- Handle **rotation** with provider **reload** + **`IOptionsMonitor<T>`** (not `IOptions<T>`); for keys, perform crypto **in the vault** (`CryptographyClient`) rather than extracting the key.

→ Next: [08-EventGrid-EventHubs.md](08-EventGrid-EventHubs.md)
