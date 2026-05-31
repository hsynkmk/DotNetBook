# Configuration Providers

## Sources that feed IConfiguration

A **configuration provider** reads settings from a source (a JSON file, environment variables, command line, a secret store) and contributes them as string key/value pairs into the merged `IConfiguration` ([01-IConfiguration.md](01-IConfiguration.md)). The provider model is what lets one `IConfiguration` transparently combine many sources — and it's extensible (you can write your own).

```csharp
var builder = Host.CreateApplicationBuilder(args);
// The host registers these defaults (in this order):
//   1. appsettings.json
//   2. appsettings.{Environment}.json
//   3. User Secrets (Development only)
//   4. Environment variables
//   5. Command-line arguments
builder.Configuration.AddJsonFile("extra.json", optional: true, reloadOnChange: true);  // add more
```

> Providers and the default order are introduced in [Ch03 §07](../03-HostingAndDI/07-Configuration.md). This file catalogs the providers and covers writing a **custom** one.

---

## Built-in providers

| Provider | Source | `Add...` |
|---|---|---|
| **JSON** | `.json` files | `AddJsonFile` |
| **Environment variables** | env (prefix-filterable; `__` for hierarchy) | `AddEnvironmentVariables` |
| **Command line** | `--key value` / `key=value` args | `AddCommandLine` |
| **User Secrets** | dev-only store outside the repo | `AddUserSecrets` |
| **INI / XML** | `.ini` / `.xml` files | `AddIniFile` / `AddXmlFile` |
| **In-memory** | a `Dictionary` (great for tests) | `AddInMemoryCollection` |
| **Azure Key Vault** | Azure secret store | `AddAzureKeyVault` ([07-Secrets.md](07-Secrets.md)) |
| **Azure App Configuration** | centralized cloud config | `AddAzureAppConfiguration` |

```csharp
builder.Configuration
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
    .AddEnvironmentVariables(prefix: "MYAPP_")     // only MYAPP_-prefixed vars, prefix stripped
    .AddCommandLine(args);
```

Each provider adds a layer; precedence follows **registration order** (later wins — [03-Layering.md](03-Layering.md)). The defaults cover most needs; you add Key Vault/App Configuration for production secrets/centralized config.

---

## How providers compose

```
Each provider produces a flat Dictionary<string,string>:
  JSON file        → { "Worker:MaxRetries": "5", "Db:Conn": "local" }
  Env variables    → { "Db:Conn": "prod" }                    (overrides via __ )
  Command line     → { "Worker:MaxRetries": "10" }
        ↓ merged in registration order (later wins)
IConfiguration     → { "Worker:MaxRetries": "10", "Db:Conn": "prod" }
```

Because every provider produces the **same flat string-key shape** ([01-IConfiguration.md](01-IConfiguration.md)), they merge uniformly: `IConfiguration` overlays them in order, and a key set by a later provider overrides an earlier one. This is what makes "defaults in JSON, overrides per environment via env vars" work seamlessly. (Layering precedence is [03-Layering.md](03-Layering.md).)

---

## In-memory provider (for tests)

`AddInMemoryCollection` is invaluable for testing — supply config directly without files:

```csharp
var config = new ConfigurationBuilder()
    .AddInMemoryCollection(new Dictionary<string, string?> {
        ["Worker:MaxRetries"] = "3",
        ["ConnectionStrings:Default"] = "DataSource=:memory:"
    })
    .Build();
```

In unit/integration tests, inject in-memory config so tests don't depend on files or environment — predictable and isolated ([Ch17 Testing](../17-Testing/README.md)). The same `IConfiguration` API works, so code under test behaves identically.

---

## Writing a custom provider

When a built-in source doesn't fit (a database, a remote config service, a custom format), implement a provider. The pattern: a `ConfigurationProvider` (loads data into the flat `Data` dictionary) + a `ConfigurationSource` (the factory) + an extension method:

```csharp
public class DatabaseConfigSource(string connectionString) : IConfigurationSource {
    public IConfigurationProvider Build(IConfigurationBuilder builder) => new DatabaseConfigProvider(connectionString);
}

public class DatabaseConfigProvider(string cs) : ConfigurationProvider {
    public override void Load() {
        // Read key/value rows from the DB into the flat Data dictionary
        Data = LoadSettingsFromDatabase(cs);   // Dictionary<string, string?> with ":"-delimited keys
    }
}

public static class DatabaseConfigExtensions {
    public static IConfigurationBuilder AddDatabase(this IConfigurationBuilder builder, string cs) =>
        builder.Add(new DatabaseConfigSource(cs));
}

// Usage
builder.Configuration.AddDatabase(connectionString);
```

Your provider just populates the flat `Data` dictionary (keys `:`-delimited); it then participates in layering and binding like any built-in provider. For reloadable custom sources, raise change tokens ([04-Reload.md](04-Reload.md)). Custom providers let you source config from anywhere (a database table, a feature-flag service, a remote API) while keeping the uniform `IConfiguration` interface.

---

## Centralized configuration (cloud)

For distributed apps, **Azure App Configuration** (and similar) centralizes settings across services with features file-based config lacks: dynamic refresh, feature flags, labels (per-environment values), and a single source of truth:

```csharp
builder.Configuration.AddAzureAppConfiguration(o => o
    .Connect(endpoint, credential)
    .UseFeatureFlags()
    .ConfigureRefresh(r => r.Register("Sentinel", refreshAll: true)));
```

Centralized config services reduce config drift across many services/instances and enable runtime changes without redeploys (dynamic refresh — [04-Reload.md](04-Reload.md)). They complement (don't replace) the file/env providers — typically App Configuration for shared/dynamic settings + Key Vault for secrets ([07-Secrets.md](07-Secrets.md)).

---

## Common gotchas

### Wrong env-var separator

Environment variables use `__` (double underscore) for hierarchy, not `:`. `Worker__MaxRetries`, not `Worker:MaxRetries`.

### Provider order confusion

Later providers override earlier ones. If an env var "isn't taking effect," ensure it's registered *after* the JSON file (the host's defaults already handle the common case — [03-Layering.md](03-Layering.md)).

### Secrets in JSON files

`appsettings.json` is committed — don't put secrets there. Use User Secrets (dev) / Key Vault / env vars (prod) ([07-Secrets.md](07-Secrets.md)).

### Custom provider not raising reload tokens

A custom provider that changes at runtime but doesn't raise a change token won't trigger reload (`IOptionsMonitor` won't update). Implement reload support if the source is dynamic ([04-Reload.md](04-Reload.md)).

### Forgetting `optional: true` for environment-specific files

`appsettings.Production.json` may not exist in dev. Mark environment/extra files `optional: true` so a missing file doesn't crash startup.

---

## Summary

- A **configuration provider** reads a source (JSON, env vars, command line, User Secrets, Key Vault, in-memory, custom) and contributes flat string key/value pairs to `IConfiguration`.
- All providers produce the **same flat shape**, so they **merge uniformly** in registration order (later wins) — enabling "JSON defaults + env-var/command-line overrides." Env vars use **`__`** for hierarchy.
- **`AddInMemoryCollection`** is ideal for tests (file/env-independent config); built-in providers cover most needs.
- Write a **custom provider** (`ConfigurationProvider` + `ConfigurationSource` + extension) to source config from anywhere (a DB, remote service) — just populate the flat `Data` dictionary; support reload tokens if dynamic.
- **Azure App Configuration** centralizes settings (+ feature flags, dynamic refresh) for distributed apps, complementing file/env providers + Key Vault for secrets.

→ Next: [03-Layering.md](03-Layering.md)
