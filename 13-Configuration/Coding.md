# Chapter 13 — Configuration — Coding Problems

Hands-on problems on the configuration and options system. Each has a hidden solution — attempt it first.

---

### Problem 1 — Bind a section to a typed options class

Given `appsettings.json` with a `"Smtp"` section (`Host`, `Port`, `FromAddress`), bind it to a `SmtpOptions` POCO and inject it into a service.

<details>
<summary>Solution</summary>

```csharp
public class SmtpOptions {
    public string Host { get; set; } = "";
    public int Port { get; set; } = 25;
    public string FromAddress { get; set; } = "";
}

builder.Services.Configure<SmtpOptions>(builder.Configuration.GetSection("Smtp"));

public class EmailSender(IOptions<SmtpOptions> options) {
    private readonly SmtpOptions _opts = options.Value;
    public void Send() { /* use _opts.Host, _opts.Port, ... */ }
}
```

Key points: mutable properties + parameterless ctor (binding requires them); `IOptions<T>` for fixed config; `GetSection` names must match the JSON key.
</details>

---

### Problem 2 — Override a nested value via environment variable

The deployed app must override `Smtp:Port` to `587` using an environment variable, without changing any file. What env var do you set?

<details>
<summary>Solution</summary>

```bash
# Use __ (double underscore) as the section separator:
Smtp__Port=587
```

The environment variable provider translates `__` → `:`. It's added after the JSON files in the default order, so it wins. (`:` works on some platforms but `__` is portable.)
</details>

---

### Problem 3 — Live config in a singleton

A singleton `RateLimiter` must pick up changes to `appsettings.json` (`reloadOnChange: true`) without a restart. Which accessor, and why not the others?

<details>
<summary>Solution</summary>

```csharp
public class RateLimiter {
    private readonly IOptionsMonitor<LimiterOptions> _monitor;
    public RateLimiter(IOptionsMonitor<LimiterOptions> monitor) {
        _monitor = monitor;
        monitor.OnChange(o => Console.WriteLine($"Reloaded: {o.PermitsPerSecond}"));
    }
    public int Permits => _monitor.CurrentValue.PermitsPerSecond;  // live
}
```

`IOptionsMonitor<T>` is a singleton with a live `CurrentValue` + `OnChange`. `IOptions<T>` reads once (won't reload). `IOptionsSnapshot<T>` is **scoped** — injecting it into a singleton is a captive dependency.
</details>

---

### Problem 4 — Validate options at startup

`RetryOptions` requires `MaxAttempts >= 1` and `BaseDelay < MaxDelay`. Configure validation so a bad value crashes the app at startup, not at first use.

<details>
<summary>Solution</summary>

```csharp
builder.Services.AddOptions<RetryOptions>()
    .Bind(builder.Configuration.GetSection("Retry"))
    .Validate(o => o.MaxAttempts >= 1, "MaxAttempts must be >= 1.")
    .Validate(o => o.BaseDelay < o.MaxDelay, "BaseDelay must be < MaxDelay.")
    .ValidateOnStart();   // <-- forces eager validation at boot
```

Without `ValidateOnStart()`, validation runs lazily on first `.Value` — the bug surfaces in production. `.Validate` handles the cross-field rule that DataAnnotations can't.
</details>

---

### Problem 5 — DataAnnotations validation

Rewrite Problem 4's presence/range checks using DataAnnotations attributes instead of predicates.

<details>
<summary>Solution</summary>

```csharp
public class RetryOptions {
    [Range(1, 10)] public int MaxAttempts { get; set; }
    [Required] public string Endpoint { get; set; } = "";
}

builder.Services.AddOptions<RetryOptions>()
    .Bind(config.GetSection("Retry"))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

Attributes cover presence/range/format declaratively. Cross-field rules (`BaseDelay < MaxDelay`) still need `.Validate(...)` or `IValidateOptions<T>`.
</details>

---

### Problem 6 — `IValidateOptions<T>` with multiple failures

Write a validator that reports *all* problems with `SmtpOptions` at once (empty host, bad port, SSL-on-port-25), as a DI service.

<details>
<summary>Solution</summary>

```csharp
public sealed class SmtpOptionsValidator : IValidateOptions<SmtpOptions> {
    public ValidateOptionsResult Validate(string? name, SmtpOptions o) {
        var failures = new List<string>();
        if (string.IsNullOrWhiteSpace(o.Host)) failures.Add("Smtp:Host is required.");
        if (o.Port is < 1 or > 65535)          failures.Add($"Smtp:Port {o.Port} out of range.");
        if (o.UseSsl && o.Port == 25)          failures.Add("Port 25 incompatible with SSL.");
        return failures.Count > 0 ? ValidateOptionsResult.Fail(failures) : ValidateOptionsResult.Success;
    }
}
builder.Services.AddSingleton<IValidateOptions<SmtpOptions>, SmtpOptionsValidator>();
builder.Services.AddOptions<SmtpOptions>().Bind(config.GetSection("Smtp")).ValidateOnStart();
```

`IValidateOptions<T>` collects all failures (not just the first), can inject dependencies, and handles named options via `name`.
</details>

---

### Problem 7 — Named options for two endpoints

Configure two `EndpointOptions` instances — `"primary"` and `"replica"` — from different config sections, and resolve both in a service.

<details>
<summary>Solution</summary>

```csharp
builder.Services.Configure<EndpointOptions>("primary", config.GetSection("Endpoints:Primary"));
builder.Services.Configure<EndpointOptions>("replica", config.GetSection("Endpoints:Replica"));

public class Router(IOptionsMonitor<EndpointOptions> monitor) {
    public string Primary => monitor.Get("primary").Url;
    public string Replica => monitor.Get("replica").Url;
}
```

Named options let one type carry several configurations; retrieve with `Get(name)` on `IOptionsMonitor`/`IOptionsSnapshot`.
</details>

---

### Problem 8 — Options that depend on a service

Configure `CacheOptions.Directory` to be `{ContentRoot}/cache`, computed from `IHostEnvironment`.

<details>
<summary>Solution</summary>

```csharp
builder.Services.AddOptions<CacheOptions>()
    .Configure<IHostEnvironment>((o, env) =>
        o.Directory = Path.Combine(env.ContentRootPath, "cache"));
```

`Configure<TDep>((opts, dep) => ...)` injects DI services into the configuration step (up to 5 deps) — for values computed from runtime services rather than static config.
</details>

---

### Problem 9 — Write a custom configuration provider

Implement a minimal in-memory-backed provider that loads key/value pairs from a `Dictionary` (simulating a custom source), wired via `IConfigurationSource`.

<details>
<summary>Solution</summary>

```csharp
public class DictionarySource(IDictionary<string, string?> data) : IConfigurationSource {
    public IConfigurationProvider Build(IConfigurationBuilder builder) => new DictionaryProvider(data);
}
public class DictionaryProvider(IDictionary<string, string?> data) : ConfigurationProvider {
    private readonly IDictionary<string, string?> _data = data;
    public override void Load() => Data = new Dictionary<string, string?>(_data, StringComparer.OrdinalIgnoreCase);
}

// Usage:
builder.Configuration.Add(new DictionarySource(new Dictionary<string, string?> {
    ["Feature:Enabled"] = "true"
}));
```

A provider populates the flat `Data` dictionary; deriving from `ConfigurationProvider` gives you `Data`, change-token support, and reload hooks for free. For a real source (DB, remote service), load in `Load()` and optionally watch for changes.
</details>

---

### Problem 10 — React to reload with `IChangeToken`

Register a callback that runs whenever configuration reloads (e.g., to log it). Use the configuration reload token directly.

<details>
<summary>Solution</summary>

```csharp
ChangeToken.OnChange(
    () => builder.Configuration.GetReloadToken(),   // produce a fresh token each cycle
    () => Console.WriteLine("Configuration reloaded"));
```

`GetReloadToken()` returns the current `IChangeToken`; `ChangeToken.OnChange` re-registers automatically after each fire (tokens are one-shot). This is the same plumbing `IOptionsMonitor` uses to recompute `CurrentValue`.
</details>

---

### Problem 11 — Add User Secrets and read a secret

Set up User Secrets for local dev and read a `Smtp:Password` secret, ensuring it's never committed.

<details>
<summary>Solution</summary>

```bash
dotnet user-secrets init                              # adds UserSecretsId to .csproj (safe to commit)
dotnet user-secrets set "Smtp:Password" "s3cr3t"      # stored in user profile, NOT the repo
```

```csharp
// Auto-loaded in Development; read like any config:
var pwd = builder.Configuration["Smtp:Password"];
```

The `UserSecretsId` (a GUID/folder name) is committed; the actual `secrets.json` lives in your user profile and can't be committed. Dev-only, not encrypted.
</details>

---

### Problem 12 — Wire Azure Key Vault with managed identity

Load production secrets from Key Vault with no secret stored in the app, using the same code that works locally.

<details>
<summary>Solution</summary>

```csharp
var vaultUri = new Uri($"https://{builder.Configuration["KeyVaultName"]}.vault.azure.net/");
builder.Configuration.AddAzureKeyVault(vaultUri, new DefaultAzureCredential());

// Secret "Smtp--Password" in the vault → config key "Smtp:Password":
var pwd = builder.Configuration["Smtp:Password"];
```

`DefaultAzureCredential` uses the app's **managed identity** in production (zero secrets) and falls back to your Azure CLI / Visual Studio login locally — same code both places. Vault secret names use `--` for the `:` separator.
</details>

---

### Problem 13 — Spot the captive-dependency bug

```csharp
public class MetricsReporter {                       // registered as Singleton
    public MetricsReporter(IOptionsSnapshot<ReportOptions> opt) { ... }
}
```

What's wrong, and how do you fix it?

<details>
<summary>Solution</summary>

`IOptionsSnapshot<T>` is **scoped**; injecting it into a **singleton** captures one scope's instance for the singleton's lifetime — a captive dependency (and it won't reflect per-request/reload as intended). Fix: inject `IOptionsMonitor<ReportOptions>` and read `.CurrentValue` (live, singleton-safe). Use `IOptionsSnapshot` only in scoped services.
</details>

---

### Problem 14 — `PostConfigure` for a guaranteed default

Multiple sources may set `WorkerOptions.QueueName`. Ensure that if *none* set it, it defaults to `"default"` — after all other configuration has run.

<details>
<summary>Solution</summary>

```csharp
builder.Services.Configure<WorkerOptions>(config.GetSection("Worker"));   // binds from config
builder.Services.PostConfigure<WorkerOptions>(o => o.QueueName ??= "default");  // runs LAST
```

`Configure` actions run in registration order; `PostConfigure` runs after all of them — the right place for final normalization/defaults that must win regardless of source order.
</details>

---

### Problem 15 — Test a service with `Options.Create`

Unit-test `EmailSender` (from Problem 1) without any configuration files or DI container.

<details>
<summary>Solution</summary>

```csharp
[Fact]
public void Sends_using_configured_host() {
    var opts = Options.Create(new SmtpOptions { Host = "smtp.test", Port = 587 });
    var sender = new EmailSender(opts);   // no config plumbing needed
    // assert behavior...
}
```

`Options.Create(value)` wraps a POCO in `IOptions<T>` for tests — one of the main reasons the Options pattern is preferred over raw `IConfiguration` reads: it's trivially testable.
</details>
