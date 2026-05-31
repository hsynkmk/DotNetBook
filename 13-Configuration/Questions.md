# Chapter 13 — Configuration — Q & A

---

### Q1. What is `IConfiguration`'s data model?

A **flat dictionary of string key → string value**. Hierarchy is expressed by `:`-delimited keys (`Logging:LogLevel:Default`), arrays by numeric index keys (`Servers:0`, `Servers:1`). Everything is a string at the core; binding/typed access converts on the way out.

---

### Q2. How is hierarchy and arrays represented in the flat model?

Sections use the `:` separator (`Smtp:Host`). Arrays use integer-index segments (`Hosts:0`, `Hosts:1`). `GetSection("Smtp")` returns a sub-view; `GetChildren()` enumerates immediate children — so `Hosts:0/1/2` enumerate as an array when bound to a `List<string>`.

---

### Q3. What is the provider model and the default provider order?

Configuration is assembled from ordered **providers**, each contributing key/value pairs; **later providers override earlier ones** for the same key. The default web host order is: `appsettings.json` → `appsettings.{Environment}.json` → User Secrets (Development) → environment variables → command-line arguments. Command line wins.

---

### Q4. How do you override a nested config value with an environment variable?

Use the key path with `__` (double underscore) as the separator (since `:` isn't valid in env var names on all platforms): `Smtp__Port=587` overrides `Smtp:Port`. The environment variable provider translates `__` to `:`.

---

### Q5. Why does environment-specific layering work the way it does?

`appsettings.{Environment}.json` is added *after* `appsettings.json`, so it overrides only the keys it specifies; unspecified keys fall through to the base file. The environment comes from `ASPNETCORE_ENVIRONMENT`/`DOTNET_ENVIRONMENT`. This lets a base file hold shared config and per-environment files hold just the deltas.

---

### Q6. What's the array-override edge case?

Arrays bind by **index key**, and providers merge by key — they don't replace whole arrays. So if the base file has a 3-element array and an override file supplies only index 0, you get the override's index 0 plus the base's indices 1 and 2 (a merge, not a replacement). To fully replace an array, clear it or use a representation that doesn't merge surprisingly.

---

### Q7. What reloads when `appsettings.json` changes, and what doesn't?

With `reloadOnChange: true`, the JSON file provider watches the file and rebuilds configuration on change, firing `IChangeToken`. `IConfiguration[...]` reads and `IOptionsMonitor<T>.CurrentValue` reflect the new values. `IOptions<T>` does **not** (read once). Things bound only at startup (DI registrations, the listening URL, etc.) also don't change.

---

### Q8. Compare `IOptions<T>`, `IOptionsSnapshot<T>`, `IOptionsMonitor<T>`.

`IOptions<T>`: singleton, value computed **once**, never reloads — for fixed config. `IOptionsSnapshot<T>`: **scoped**, recomputed per request/scope, reflects reload — for request-scoped services. `IOptionsMonitor<T>`: singleton, `CurrentValue` is **live** with an `OnChange` callback — for singletons/background services that need live config.

---

### Q9. Why must you never inject `IOptionsSnapshot<T>` into a singleton?

`IOptionsSnapshot<T>` is **scoped**; injecting a scoped service into a singleton is a captive dependency — the singleton captures one scope's instance forever, breaking per-request semantics (and risking thread-safety issues). Use `IOptionsMonitor<T>` for live config in a singleton.

---

### Q10. What does `OptionsBuilder<T>` (`AddOptions<T>()`) give you?

A fluent chain over an options type: `.Bind(section)`, `.Configure(...)`, `.Configure<TDep>(...)` (dependency-aware), `.ValidateDataAnnotations()`, `.Validate(predicate, msg)`, `.ValidateOnStart()`. It's the modern, fuller setup vs the basic `Configure<T>(section)` — especially when you need validation or dependency-aware configuration.

---

### Q11. What is `Configure<TDep>` and `PostConfigure`?

`Configure<TDep1,...>((opts, dep) => ...)` configures options using other DI services (up to 5 dependencies) — e.g., compute a path from `IHostEnvironment`. `PostConfigure<T>` runs **after** all `Configure` actions — for final defaults/normalization that must win regardless of order (e.g., "if no value set, use a default").

---

### Q12. What are named options?

Multiple configured instances of one options type, retrieved by name via `IOptionsMonitor<T>.Get("name")` / `IOptionsSnapshot<T>.Get("name")`. Registered with `Configure<T>("name", section)`. Useful for per-tenant or per-endpoint settings where the same shape has different values.

---

### Q13. What are the three options-validation mechanisms?

`ValidateDataAnnotations()` (attributes — `[Required]`/`[Range]`/...), `.Validate(predicate, message)` (cross-field rules in code), and `IValidateOptions<T>` (a DI service — dependency-aware, multiple messages, named-options support). Use the simplest that expresses your rules.

---

### Q14. Why is `ValidateOnStart()` essential?

Without it, options validation runs **lazily on first `.Value` access** — so misconfiguration surfaces whenever that code path first runs, often in production. `ValidateOnStart()` forces validation **at host startup**, turning bad config into an immediate boot failure (fail fast) that the deployment system catches.

---

### Q15. Does a binding key/property mismatch throw?

No. Binding is by name (case-insensitive); an unmatched key/property **silently leaves the default**. This is why validation (`[Required]` + `ValidateOnStart`) matters — it converts "silently defaulted" into a loud failure.

---

### Q16. Why must secrets never be in `appsettings.json` or git?

The repo (and its history) can leak; public repos are scraped within minutes. Because git history persists, a committed secret must be **rotated** (invalidated), not merely deleted. Keep secrets out from the start.

---

### Q17. How do you handle secrets in local development?

**User Secrets** (`dotnet user-secrets set ...`): stored in your user profile *outside* the project, auto-loaded by the host in the Development environment, layered to override `appsettings.json`. They're not encrypted (just out-of-repo) and are dev-only.

---

### Q18. How do you handle secrets in production?

A dedicated **secret manager** (e.g., Azure Key Vault) integrated as a configuration provider — secrets read as ordinary `IConfiguration` keys. Authenticate with **managed identity** (`DefaultAzureCredential`) so there's no secret stored in the app to leak.

---

### Q19. What problem does managed identity solve?

The bootstrap problem: reading secrets needs a credential, which is itself a secret. Managed identity gives the app a platform-managed identity that Azure AD authenticates without any credential in code/config — the chain of secrets terminates at the platform. `DefaultAzureCredential` uses it in prod and falls back to your dev login locally.

---

### Q20. How do you pick up a rotated secret without a redeploy?

Configure the secret provider (e.g., Key Vault) to reload on an interval, and read via `IOptionsMonitor<T>` (live) rather than `IOptions<T>` (read-once) so the rotated value is reflected. Don't poll too aggressively (cost/latency).

---

### Q21. Can you write a custom configuration provider? When?

Yes — implement `IConfigurationSource` + `IConfigurationProvider` (or derive from `ConfigurationProvider`). Use it for a source the built-ins don't cover (a database, a remote config service, a bespoke file format). The provider populates the flat key/value `Data` dictionary; the rest of the pipeline treats it like any other source.

---

### Q22. What is `IChangeToken` and how does reload use it?

`IChangeToken` is a one-shot change-notification primitive: it exposes `HasChanged` and `RegisterChangeCallback`. Reloadable providers watch their source and trigger a fresh token on change; the options system registers callbacks to rebuild `IOptionsMonitor` values and fire `OnChange`. It's the plumbing behind reload-on-change.
