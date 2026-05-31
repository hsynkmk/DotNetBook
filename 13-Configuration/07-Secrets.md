# Secrets Management

## Secrets are configuration — but they must never live in the repo

Connection strings, API keys, signing keys, client secrets, certificates — these are configuration values ([01-IConfiguration.md](01-IConfiguration.md)), but with one absolute rule: **they must never be committed to source control.** A secret in `appsettings.json` (or worse, in git history) is a leak waiting to happen — public repos get scraped within minutes, and `git rm` doesn't erase history. This file covers how to keep secrets *out* of the repo while still feeding them through the same `IConfiguration` pipeline your code already reads.

The strategy differs by stage:

| Stage | Where secrets come from |
|---|---|
| **Local development** | User Secrets (per-developer, outside the repo) |
| **CI/CD** | Pipeline secret store / masked variables |
| **Production** | A secret manager (Azure Key Vault, AWS Secrets Manager) + managed identity, or environment variables injected by the platform |

---

## Local dev: User Secrets

The .NET **Secret Manager** stores secrets in a JSON file in your *user profile* — outside the project directory, so it can't be committed:

```bash
dotnet user-secrets init                                  # adds a UserSecretsId to the .csproj
dotnet user-secrets set "Smtp:Password" "s3cr3t"          # store a secret
dotnet user-secrets set "ConnectionStrings:Db" "Server=..."
dotnet user-secrets list
```

The file lives at `%APPDATA%\Microsoft\UserSecrets\<id>\secrets.json` (Windows) / `~/.microsoft/usersecrets/<id>/secrets.json` (Linux/macOS). The `WebApplication`/host **automatically loads User Secrets in the Development environment** ([03-Layering.md](03-Layering.md)), layered so they override `appsettings.json`:

```csharp
// Done automatically in Development by the default host builder; explicitly:
builder.Configuration.AddUserSecrets<Program>();
```

User Secrets are **not encrypted** — they're just stored outside the repo. They solve "don't commit secrets," not "secrets at rest are protected." They're for local dev only; never a production mechanism.

---

## Production: Azure Key Vault

In production, secrets belong in a dedicated **secret manager**. Azure Key Vault integrates as a configuration provider ([02-Providers.md](02-Providers.md)) — secrets become ordinary `IConfiguration` keys:

```csharp
var keyVaultUri = new Uri($"https://{builder.Configuration["KeyVaultName"]}.vault.azure.net/");
builder.Configuration.AddAzureKeyVault(keyVaultUri, new DefaultAzureCredential());

// Now read like any config — the value comes from Key Vault:
var dbConn = builder.Configuration.GetConnectionString("Db");
```

Key Vault secret names use `--` as the section separator (since `:` isn't allowed in secret names): a secret named `Smtp--Password` maps to config key `Smtp:Password`. The code reading configuration doesn't change — only the *source* of the value does. This is the power of the unified configuration model: swap the provider, keep the consumers.

---

## Managed identity — secrets without a secret

The bootstrapping problem: to read secrets from Key Vault, you need a credential — but that credential is itself a secret. **Managed identity** solves it: Azure assigns your app (App Service, AKS, VM, Container App) an identity that Azure AD authenticates *without any credential in your code or config*.

```csharp
// DefaultAzureCredential tries, in order: environment vars → managed identity →
// Azure CLI login (dev) → Visual Studio login → etc.
builder.Configuration.AddAzureKeyVault(keyVaultUri, new DefaultAzureCredential());
```

`DefaultAzureCredential` uses the managed identity in production (zero secrets in your app) and falls back to your developer login (Azure CLI / Visual Studio) locally — the *same code* works in both environments. You grant the managed identity `Get`/`List` permission on the vault's secrets, and there is **no secret to leak**: no connection string, no client secret, nothing in config. This is the gold standard — the chain of secrets terminates at a platform-managed identity.

---

## Other production options

- **Environment variables** — the platform (Kubernetes Secrets, Docker, App Service app settings) injects secrets as env vars, picked up by the environment variable provider ([02-Providers.md](02-Providers.md)). Simple and universal, but env vars are visible to the process tree and can leak via crash dumps/logs.
- **AWS Secrets Manager / Parameter Store**, **HashiCorp Vault**, **Google Secret Manager** — each has a community/official configuration provider following the same pattern as Key Vault.
- **Kubernetes Secrets** mounted as files — read via the file/key-per-file configuration provider.

The common thread: a dedicated secret store + an identity-based (not secret-based) way to authenticate to it.

---

## Rotation and caching

Secrets get rotated (periodically, or in response to a leak). Two considerations:

- **Reload**: the Key Vault provider can be configured to reload on an interval, so a rotated secret flows in without a redeploy — combine with `IOptionsMonitor` ([04-Reload.md](04-Reload.md)) to pick up the new value. (Don't poll too aggressively — Key Vault calls are network/cost.)
- **Don't cache secrets in `IOptions<T>`** (read-once) if they rotate — use `IOptionsMonitor<T>` so a reloaded secret is reflected.

---

## Common gotchas

### Secrets in `appsettings.json` (or git history)

The cardinal sin. Even after deletion, secrets persist in git history — they must be **rotated** (invalidated), not just removed. Keep secrets out from the start (User Secrets locally, a secret manager in prod).

### Committing the User Secrets ID — that's fine; the secrets file — impossible

The `UserSecretsId` in the `.csproj` is *not* a secret (it's just a folder name) and is meant to be committed. The actual `secrets.json` lives outside the repo and can't be committed accidentally.

### Using User Secrets in production

User Secrets are a Development-only, unencrypted convenience. Production needs a real secret manager. The host only auto-loads them in Development.

### Storing a Key Vault access secret to access Key Vault

If you authenticate to Key Vault with a client secret stored in config, you've just moved the problem. Use **managed identity** (`DefaultAzureCredential`) so there's no secret to store.

### Logging configuration

Never log the full configuration or connection strings — secrets end up in logs ([Ch12](../12-Observability/README.md)). Redact secret values in any config-dump diagnostics.

### Secrets that rotate read via `IOptions<T>`

`IOptions<T>` reads once and never updates ([04-Reload.md](04-Reload.md)); a rotated secret won't be picked up. Use `IOptionsMonitor<T>` for rotatable secrets.

---

## Summary

- **Secrets are configuration but must never be in the repo** — a committed secret leaks and (because git history persists) must be *rotated*, not just deleted.
- **Local dev**: use **User Secrets** (`dotnet user-secrets`) — stored in your user profile, outside the project, auto-loaded in Development (not encrypted, dev-only).
- **Production**: use a **secret manager** like **Azure Key Vault**, integrated as a configuration provider so secrets read as ordinary `IConfiguration` keys (consumers don't change).
- **Managed identity** (`DefaultAzureCredential`) authenticates to the secret store with **no secret in your app** — the gold standard; the same code falls back to your dev login locally.
- Handle **rotation** with reload + `IOptionsMonitor<T>`; never log secrets, and never store the credential that unlocks the vault in config.

→ Next: [Questions.md](Questions.md)
