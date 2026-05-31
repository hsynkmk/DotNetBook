# Configuration

## Layered configuration from many sources

`IConfiguration` is the unified, key/value configuration system. Its defining feature is **layering**: multiple sources (JSON files, environment variables, command-line args, secrets, key vaults) are merged into one view, with **later sources overriding earlier ones**. The host sets up sensible defaults; you add more as needed.

```csharp
var builder = Host.CreateApplicationBuilder(args);
// Already loaded (in this order, later wins):
//   1. appsettings.json
//   2. appsettings.{Environment}.json   (e.g., appsettings.Production.json)
//   3. User Secrets (Development only)
//   4. Environment variables
//   5. Command-line arguments

string? cs = builder.Configuration["ConnectionStrings:Default"];
int retries = builder.Configuration.GetValue<int>("Worker:MaxRetries", defaultValue: 3);
```

So an environment variable overrides `appsettings.Production.json`, which overrides `appsettings.json`, which a command-line arg can override again. This precedence is the heart of environment-specific configuration.

---

## The layering model (precedence)

```
appsettings.json                 ← base defaults (committed)
  ↓ overridden by
appsettings.{Environment}.json   ← per-environment overrides (Development/Staging/Production)
  ↓ overridden by
User Secrets                     ← local dev secrets (NOT committed; Development only)
  ↓ overridden by
Environment variables            ← container/host config; secrets in prod
  ↓ overridden by
Command-line arguments           ← highest precedence, ad-hoc overrides
```

Each provider contributes key/value pairs into a flat dictionary; `IConfiguration` presents the merged result. The order is what gives you "defaults in JSON, overrides per environment, secrets from the environment/vault."

---

## Hierarchical keys and binding

JSON nesting maps to `:`-delimited keys:

```json
{
  "Worker": { "MaxRetries": 5, "Interval": "00:00:30" },
  "ConnectionStrings": { "Default": "Server=..." }
}
```

```csharp
config["Worker:MaxRetries"];                       // "5" (string)
config.GetValue<int>("Worker:MaxRetries");          // 5 (typed)
config.GetSection("Worker");                         // a sub-tree
config.GetConnectionString("Default");               // shortcut for ConnectionStrings:Default

// Bind a whole section to a typed object (the basis of the Options pattern):
var workerOptions = config.GetSection("Worker").Get<WorkerOptions>();
```

In **environment variables**, `:` is replaced by `__` (double underscore) for cross-platform compatibility:

```bash
export Worker__MaxRetries=10        # sets Worker:MaxRetries
export ConnectionStrings__Default="Server=prod;..."
```

Binding (`Get<T>`/`Bind`) maps a config section onto a POCO's properties by name — the foundation of the **Options pattern** ([08-Options.md](08-Options.md)).

---

## Configuration providers

`IConfiguration` is provider-based; you can add many:

```csharp
builder.Configuration
    .AddJsonFile("custom.json", optional: true, reloadOnChange: true)
    .AddEnvironmentVariables(prefix: "MYAPP_")
    .AddCommandLine(args)
    .AddIniFile("legacy.ini", optional: true);
```

Built-in / common providers:
- **JSON** (`appsettings.json`), **INI**, **XML** files.
- **Environment variables** (with optional prefix).
- **Command-line** arguments.
- **User Secrets** (dev-only, stored outside the repo).
- **Azure Key Vault**, **AWS Secrets Manager**, **HashiCorp Vault** (via NuGet) — for production secrets.
- **In-memory** (great for tests): `AddInMemoryCollection`.

Each provider adds a layer; precedence follows registration order (later wins).

---

## Secrets — never commit them

```bash
# User Secrets — for LOCAL DEVELOPMENT only (stored in your user profile, not the repo)
dotnet user-secrets init
dotnet user-secrets set "ConnectionStrings:Default" "Server=localhost;..."
```

The cardinal rule: **secrets do not belong in `appsettings.json`** (it's committed to source control). For:
- **Local dev** → User Secrets (`dotnet user-secrets`).
- **Production** → environment variables (injected by the orchestrator) or a **secret store** (Azure Key Vault, AWS Secrets Manager) added as a config provider.

```csharp
// Production: pull secrets from Key Vault as a config layer
builder.Configuration.AddAzureKeyVault(vaultUri, credential);
```

(Security and secret management in depth: [Ch10 Identity & Security](../10-Identity/README.md).)

---

## Reload on change & change tokens

File providers can watch for changes and reload without a restart:

```csharp
builder.Configuration.AddJsonFile("appsettings.json", optional: false, reloadOnChange: true);

// React to changes via change tokens (rare to use directly; IOptionsMonitor wraps this)
ChangeToken.OnChange(
    () => config.GetReloadToken(),
    () => Console.WriteLine("config reloaded"));
```

`reloadOnChange: true` re-reads the file when it changes on disk. You rarely consume `GetReloadToken()` directly — **`IOptionsMonitor<T>`** ([08-Options.md](08-Options.md)) surfaces live-reloaded config to your services. Note: environment variables and command-line args don't reload (they're read once at startup).

---

## Accessing configuration

Three ways, in order of preference:

```csharp
// 1. BEST: bind to typed Options and inject IOptions<T> (see 08-Options.md)
builder.Services.Configure<WorkerOptions>(builder.Configuration.GetSection("Worker"));
public class Worker(IOptions<WorkerOptions> options) { var n = options.Value.MaxRetries; }

// 2. Inject IConfiguration directly (fine for occasional reads)
public class Service(IConfiguration config) { var x = config["Some:Key"]; }

// 3. At startup, read from builder.Configuration (composition root)
var cs = builder.Configuration.GetConnectionString("Default");
```

**Prefer the Options pattern** (typed, validated, testable) over scattering `IConfiguration["string:keys"]` through your code (stringly-typed, no validation, easy to mistype). Inject `IConfiguration` only where typed options don't fit.

---

## Common gotchas

### Secrets in `appsettings.json`

It's committed — secrets leak into source control. Use User Secrets (dev) and env vars / a vault (prod).

### Wrong env-var separator

Environment variables use `__` (double underscore), not `:`, for hierarchy: `Worker__MaxRetries`, not `Worker:MaxRetries`.

### Expecting env vars / args to reload

Only file providers reload (`reloadOnChange`). Env vars and command-line are read once at startup.

### Stringly-typed config everywhere

`config["A:B:C"]` scattered through code is fragile (typos, no validation, no type safety). Bind to typed Options.

### Precedence confusion

Later providers override earlier. If an env var "isn't taking effect," check it's registered after the JSON file (the host's default order already handles the common case).

### Missing key returns null, not an error

`config["Missing:Key"]` returns `null` silently. Use `GetValue<T>(key, default)` or validate options at startup ([10-Validation.md](10-Validation.md)).

---

## Summary

- **`IConfiguration`** merges multiple **layered** sources into one key/value view, with **later sources overriding earlier** (JSON → env-specific JSON → secrets → env vars → command line).
- Hierarchical keys use `:` (and `__` in environment variables); **bind sections to typed POCOs** (the Options pattern) rather than reading string keys everywhere.
- Many **providers** (JSON/INI/XML, env, command line, User Secrets, Key Vault, in-memory for tests); precedence follows registration order.
- **Never commit secrets** — User Secrets for dev, env vars / a secret store for production.
- **`reloadOnChange`** reloads file providers (surfaced via `IOptionsMonitor`); env/args are read once.
- Prefer typed **Options** over scattered `IConfiguration["keys"]`. Options next.

→ Next: [08-Options.md](08-Options.md)
