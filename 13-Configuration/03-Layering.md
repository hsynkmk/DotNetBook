# Configuration Layering

## Overriding settings across sources

Layering is configuration's defining feature: multiple providers ([02-Providers.md](02-Providers.md)) merge into one `IConfiguration`, and **later sources override earlier ones** for the same key. This is what enables "base defaults in JSON, environment-specific overrides, secrets from the environment, ad-hoc overrides from the command line" — each layer refining the one below.

```
appsettings.json                 ← base defaults (committed)
  ↓ overridden by
appsettings.{Environment}.json   ← per-environment (Development/Staging/Production)
  ↓ overridden by
User Secrets                     ← local dev secrets (Development only, not committed)
  ↓ overridden by
Environment variables            ← container/host config; secrets in prod
  ↓ overridden by
Command-line arguments           ← highest precedence, ad-hoc
```

> Introduced in [Ch03 §07](../03-HostingAndDI/07-Configuration.md). This file focuses on the **precedence model**, environment-specific files, and override mechanics.

---

## The precedence rule

The rule is simply **registration order — last wins**. The host registers the default providers in the order above, so each later one overrides keys set by earlier ones:

```csharp
// appsettings.json:               { "Logging": { "LogLevel": { "Default": "Information" } } }
// appsettings.Production.json:     { "Logging": { "LogLevel": { "Default": "Warning" } } }
// env var:  Logging__LogLevel__Default=Error

config["Logging:LogLevel:Default"];   // → "Error" in Production (env var wins over both JSON files)
```

Only the **specified** keys override — unspecified keys fall through to the lower layer. So `appsettings.Production.json` need only contain the keys that *differ* from `appsettings.json`; everything else is inherited. This "override just what changes" model keeps environment files small and the base file authoritative.

---

## Environment-specific files

The host loads `appsettings.{Environment}.json` based on the **environment name** (`ASPNETCORE_ENVIRONMENT` / `DOTNET_ENVIRONMENT`):

```
ASPNETCORE_ENVIRONMENT=Development → loads appsettings.json + appsettings.Development.json
ASPNETCORE_ENVIRONMENT=Production  → loads appsettings.json + appsettings.Production.json
(unset)                            → defaults to Production
```

```csharp
if (builder.Environment.IsDevelopment()) { /* dev-only setup */ }
builder.Environment.EnvironmentName;   // "Development" / "Staging" / "Production"
```

This is the primary mechanism for per-environment configuration: `appsettings.json` holds defaults, `appsettings.Development.json` holds dev overrides (verbose logging, local connection strings), `appsettings.Production.json` holds prod overrides. The environment file layers *on top of* the base, overriding only what differs. Environment-specific files should be `optional: true` (Production's file won't exist locally).

---

## Overriding nested keys and arrays

Layering works at the **key** level, including nested keys and array elements:

```json
// appsettings.json
{ "Smtp": { "Host": "localhost", "Port": 25, "UseSsl": false } }
```
```bash
# Override just two keys via env vars (the rest inherit):
export Smtp__Host=smtp.prod.com
export Smtp__UseSsl=true
# Result: { Host: "smtp.prod.com", Port: 25, UseSsl: true }  (Port inherited)
```

You can override a single deep key (`Smtp:UseSsl`) without redefining the whole section — only that key is replaced. **Array overriding** is by index (`AllowedHosts:0`), which has a sharp edge: layering overrides *element by element*, it doesn't replace the whole array. So if the base has a 3-element array and an override sets only index 0, you get the override's index 0 + the base's indices 1–2 (not a 1-element array). Be deliberate with array overrides.

---

## A typical layering setup

```
Local dev:
  appsettings.json              → base (committed, no secrets)
  appsettings.Development.json   → dev DB, verbose logging (committed)
  User Secrets                   → local secrets (NOT committed)

Production (container):
  appsettings.json              → base (in the image)
  appsettings.Production.json    → prod-tuned non-secret settings (in the image)
  Environment variables          → injected by the orchestrator (incl. secrets via __ )
  (Azure Key Vault)              → secrets, added as a provider ([07-Secrets.md])
```

The pattern: **non-secret defaults in committed JSON** (base + per-environment), **secrets and environment-specific overrides from outside** (User Secrets in dev, env vars / Key Vault in prod). The committed files are safe to read; the sensitive/variable bits layer on top from the environment. This separation is the core of safe, environment-aware configuration.

---

## Common gotchas

### Misunderstanding precedence

Later providers override earlier ones. If an env var isn't taking effect, it's likely registered before the JSON that overrides it (the host defaults handle the common case — env > JSON). Check the order.

### Putting everything in environment files

`appsettings.{Environment}.json` should hold only what **differs** from the base, not a full copy (which causes drift). Override just the changed keys; let the rest inherit.

### Secrets in committed JSON

`appsettings.json`/`.Development.json` are committed — no secrets. Layer secrets from User Secrets (dev) / env vars / Key Vault (prod) ([07-Secrets.md](07-Secrets.md)).

### Array override surprises

Overriding `Array:0` replaces only that element, not the whole array (you get base elements 1..n plus your index 0). To replace an array entirely, you may need to clear or restructure it.

### Forgetting `optional: true`

Environment-specific files (Production's) won't exist in all environments. Mark them optional so a missing file doesn't crash startup.

### Environment not set

If `ASPNETCORE_ENVIRONMENT` is unset, it defaults to **Production** — so dev-only behavior won't activate. Set it explicitly per environment.

---

## Summary

- **Layering** merges providers into one `IConfiguration` with **later sources overriding earlier** (registration order, last wins) — only the **specified keys** override; the rest fall through to lower layers.
- Default precedence: appsettings.json → appsettings.{Environment}.json → User Secrets (dev) → env vars → command line. The host loads the environment file by `ASPNETCORE_ENVIRONMENT` (defaults to **Production** if unset).
- **Environment files hold only what differs** from the base (avoid full copies/drift); override nested keys individually (`Smtp__UseSsl`); arrays override **by index** (a sharp edge — not whole-array replacement).
- Pattern: **non-secret defaults in committed JSON**, **secrets/variable overrides from the environment** (User Secrets dev, env vars / Key Vault prod — [07-Secrets.md](07-Secrets.md)); mark environment/optional files `optional: true`.

→ Next: [04-Reload.md](04-Reload.md)
