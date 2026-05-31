# IConfiguration

## The configuration model

`IConfiguration` is the unified abstraction for reading settings, regardless of where they come from (JSON, environment variables, command line, secret stores). Its model is deceptively simple: a **flat dictionary of string keys to string values**, where hierarchy is expressed by `:`-delimited keys. Every provider contributes key/value pairs into this merged view.

```csharp
string? cs = config["ConnectionStrings:Default"];        // indexer — always a string (or null)
int retries = config.GetValue<int>("Worker:MaxRetries", defaultValue: 3);  // typed with default
IConfigurationSection worker = config.GetSection("Worker");  // a sub-tree
string? db = config.GetConnectionString("Default");      // shortcut for ConnectionStrings:Default
```

> Configuration is introduced in [Ch03 §07](../03-HostingAndDI/07-Configuration.md) (layering, providers, secrets); this chapter goes deeper. This file covers the **model itself** — keys, sections, binding, and the everything-is-a-string reality.

---

## Everything is a flat string dictionary

The core insight: under the hood, **all configuration is `string` key → `string` value**, even from nested JSON. Hierarchy collapses into `:`-delimited keys:

```json
{ "Worker": { "MaxRetries": 5, "Interval": "00:00:30" }, "Features": { "Beta": true } }
```

becomes:

```
"Worker:MaxRetries"   = "5"        (string!)
"Worker:Interval"     = "00:00:30"
"Features:Beta"       = "true"     (string!)
```

So `config["Worker:MaxRetries"]` returns the **string** `"5"`; `GetValue<int>` parses it to `5`. This flat model is why all providers can merge uniformly (each just contributes string key/value pairs — [02-Providers.md](02-Providers.md)) and why the key separator matters: `:` in code/JSON, but **`__`** (double underscore) in environment variables (where `:` isn't always valid):

```bash
export Worker__MaxRetries=10        # sets "Worker:MaxRetries"
```

---

## Sections & navigation

```csharp
IConfigurationSection worker = config.GetSection("Worker");
int retries = worker.GetValue<int>("MaxRetries");
string? interval = worker["Interval"];

foreach (var child in config.GetSection("Endpoints").GetChildren())   // iterate sub-keys
    Console.WriteLine($"{child.Key} = {child["Url"]}");

bool exists = config.GetSection("Optional").Exists();
```

`GetSection` returns a sub-tree you can navigate or bind; `GetChildren` enumerates immediate keys (useful for arrays/dictionaries in config). A missing key returns **null** (indexer) silently — `GetValue<T>(key, default)` provides a fallback, and `Exists()` checks presence.

---

## Arrays and collections in configuration

The flat model represents arrays with **indexed keys**:

```json
{ "AllowedHosts": ["a.com", "b.com"], "Limits": [{ "Name": "x", "Max": 10 }] }
```

```
"AllowedHosts:0" = "a.com"
"AllowedHosts:1" = "b.com"
"Limits:0:Name"  = "x"
"Limits:0:Max"   = "10"
```

```csharp
var hosts = config.GetSection("AllowedHosts").Get<string[]>();   // bind to an array
```

Arrays are just numbered keys (`:0`, `:1`). This matters for **overriding** array elements across providers (you can override `AllowedHosts:0` via an env var) and for binding collections to typed objects.

---

## Binding to typed objects

Rather than reading string keys everywhere, **bind** a section to a POCO — the foundation of the Options pattern ([05-Options.md](05-Options.md)):

```csharp
public class WorkerSettings { public int MaxRetries { get; set; } public TimeSpan Interval { get; set; } }

var settings = config.GetSection("Worker").Get<WorkerSettings>();   // bind once
// or bind into an existing instance:
config.GetSection("Worker").Bind(existingSettings);
```

`Get<T>()`/`Bind()` map config keys onto a POCO's properties **by name** (case-insensitive), converting strings to the property types. This is far better than scattering `config["Worker:MaxRetries"]` — typed, refactorable, validatable. Prefer binding (and the Options pattern) over raw string-key reads ([05-Options.md](05-Options.md)).

---

## Accessing configuration

Three ways, in order of preference:

```csharp
// 1. BEST — bind to typed Options and inject IOptions<T> ([05-Options.md])
builder.Services.Configure<WorkerSettings>(builder.Configuration.GetSection("Worker"));

// 2. Inject IConfiguration directly (for occasional reads)
public class Service(IConfiguration config) { var x = config["Some:Key"]; }

// 3. At startup, from builder.Configuration (composition root)
var cs = builder.Configuration.GetConnectionString("Default");
```

Prefer the **Options pattern** (typed, validated, testable — [05-Options.md](05-Options.md)) over injecting `IConfiguration` and reading string keys throughout your code (stringly-typed, no validation, easy to mistype).

---

## Common gotchas

### Everything is a string

`config["Worker:MaxRetries"]` is the string `"5"`, not an int. Use `GetValue<int>` or bind to a typed object — don't compare/use the raw string as a number.

### Wrong key separator in env vars

JSON/code use `:`; environment variables use `__` (double underscore) for hierarchy: `Worker__MaxRetries`, not `Worker:MaxRetries`.

### Missing key returns null silently

`config["Missing:Key"]` is `null` (no error). Use `GetValue<T>(key, default)`, `Exists()`, or validate bound options at startup ([06-Validation.md](06-Validation.md)).

### Stringly-typed config everywhere

Scattering `config["A:B:C"]` is fragile (typos, no validation, no types). Bind to typed Options instead.

### Case sensitivity assumptions

Configuration keys are **case-insensitive**. Don't rely on exact casing; and binding matches property names case-insensitively.

### Binding fails silently for mismatched names

If a config key doesn't match a property name, binding leaves the default (no error). Validate options at startup to catch missing/misnamed values.

---

## Summary

- **`IConfiguration`** is a unified, **flat string-dictionary** model — nested JSON collapses to `:`-delimited string keys (`"Worker:MaxRetries" = "5"`), and **every value is a string** (parse via `GetValue<T>` or binding).
- Navigate with **`GetSection`/`GetChildren`**; arrays are indexed keys (`:0`, `:1`); the env-var separator is **`__`** (not `:`).
- **Bind sections to typed POCOs** (`Get<T>`/`Bind`, by name, case-insensitive) — the basis of the Options pattern; prefer this over scattering string-key reads.
- A missing key returns **null silently** — use defaults/`Exists()`/startup validation; keys are case-insensitive.
- Access via the **Options pattern** (best — [05](05-Options.md)), injected `IConfiguration` (occasional reads), or `builder.Configuration` (startup). Providers/layering: [02](02-Providers.md)/[03](03-Layering.md).

→ Next: [02-Providers.md](02-Providers.md)
