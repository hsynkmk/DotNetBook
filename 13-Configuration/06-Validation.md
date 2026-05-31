# Validating Options

## Fail at startup, not at 3 a.m.

Configuration is *input* — and input can be wrong: a missing connection string, a negative timeout, a typo'd key that silently leaves a default, a URL that isn't a URL. Without validation, a bad value sits quietly until the exact code path that reads it runs — often in production, under load, hours after deploy. **Options validation** turns those latent failures into a loud, immediate startup crash: the app refuses to start with a clear message naming the bad setting. The rule is *fail fast* — a misconfigured app should never accept traffic.

```csharp
builder.Services.AddOptions<SmtpOptions>()
    .Bind(builder.Configuration.GetSection("Smtp"))
    .ValidateDataAnnotations()   // enforce the [Required]/[Range]/... attributes
    .ValidateOnStart();          // run validation at startup, not on first use
```

> The Options pattern and its validation hooks are introduced in [Ch03 §10](../03-HostingAndDI/10-Validation.md). This file goes deeper: the three validation mechanisms, *when* validation runs, and the `ValidateOnStart` gotcha.

---

## DataAnnotations — declarative validation

The simplest approach: annotate the options class with validation attributes, then call `ValidateDataAnnotations()`:

```csharp
public class SmtpOptions {
    [Required] public string Host { get; set; } = "";
    [Range(1, 65535)] public int Port { get; set; }
    [Required, EmailAddress] public string FromAddress { get; set; } = "";
    [Range(typeof(TimeSpan), "00:00:01", "00:05:00")]
    public TimeSpan Timeout { get; set; }
}

builder.Services.AddOptions<SmtpOptions>()
    .Bind(builder.Configuration.GetSection("Smtp"))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

Attributes (`[Required]`, `[Range]`, `[EmailAddress]`, `[RegularExpression]`, `[MinLength]`, etc.) declare the constraints next to the property. `ValidateDataAnnotations()` wires a validator that enforces them. This covers the common cases (presence, range, format) with zero validation code — and the constraints are self-documenting on the class.

---

## Custom `Validate` — predicate validation

For rules that attributes can't express (cross-field constraints, conditional rules), use the `Validate` overload with a predicate:

```csharp
builder.Services.AddOptions<RetryOptions>()
    .Bind(config.GetSection("Retry"))
    .Validate(o => o.MaxAttempts >= 1, "MaxAttempts must be at least 1.")
    .Validate(o => o.BaseDelay < o.MaxDelay, "BaseDelay must be less than MaxDelay.")
    .ValidateOnStart();
```

Each `Validate(predicate, message)` adds a check; the message is surfaced on failure. This handles *relationships between fields* (`BaseDelay < MaxDelay`) that DataAnnotations can't. The predicate runs against the fully-bound options instance, so all properties are populated.

---

## `IValidateOptions<T>` — full, reusable, injectable validation

The most powerful mechanism: implement `IValidateOptions<T>`. This is a class (so it can take **dependencies via DI**), can produce **multiple failure messages**, and can validate **named options**:

```csharp
public sealed class SmtpOptionsValidator : IValidateOptions<SmtpOptions> {
    public ValidateOptionsResult Validate(string? name, SmtpOptions o) {
        var failures = new List<string>();
        if (string.IsNullOrWhiteSpace(o.Host))
            failures.Add("Smtp:Host is required.");
        if (o.Port is < 1 or > 65535)
            failures.Add($"Smtp:Port {o.Port} is out of range (1-65535).");
        if (o.UseSsl && o.Port == 25)
            failures.Add("Port 25 cannot be used with SSL; use 465 or 587.");
        return failures.Count > 0
            ? ValidateOptionsResult.Fail(failures)
            : ValidateOptionsResult.Success;
    }
}

builder.Services.AddSingleton<IValidateOptions<SmtpOptions>, SmtpOptionsValidator>();
```

Because the validator is a DI service, it can inject other services (e.g., check a value against a database or another option). It reports *all* failures at once (not just the first), and the `name` parameter lets one validator handle named options ([05-Options.md](05-Options.md)). This is the right tool for complex or dependency-aware validation.

---

## `ValidateOnStart()` — the critical piece

Here's the trap that catches everyone: **by default, options validation runs lazily — on first access to `.Value`** (when the options are first resolved). If a misconfigured option is only read on some rarely-hit code path, the app starts "fine" and crashes later, in production.

```csharp
builder.Services.AddOptions<SmtpOptions>()
    .Bind(config.GetSection("Smtp"))
    .ValidateDataAnnotations();          // ✗ validates LAZILY — on first .Value access
```

`ValidateOnStart()` forces validation to run **eagerly at host startup** (via an `IHostedService` that resolves and validates every such options instance before the app accepts traffic):

```csharp
builder.Services.AddOptions<SmtpOptions>()
    .Bind(config.GetSection("Smtp"))
    .ValidateDataAnnotations()
    .ValidateOnStart();                  // ✓ validates at startup — bad config = boot failure
```

**Always add `ValidateOnStart()`.** A `OptionsValidationException` thrown during `app.Run()` startup is exactly what you want: the deployment fails, the orchestrator (Kubernetes, App Service) sees an unhealthy boot and doesn't route traffic to it, and you see the error in logs immediately — instead of discovering it when the first email send fails at midnight.

---

## When does each accessor trigger validation?

Validation runs when the options instance is *built*, which differs by accessor ([05-Options.md](05-Options.md)):

| Accessor | Validation timing (without `ValidateOnStart`) |
|---|---|
| `IOptions<T>` | once, on first `.Value` |
| `IOptionsSnapshot<T>` | per scope, on first access in that scope |
| `IOptionsMonitor<T>` | on build / on reload of `CurrentValue` |

With `ValidateOnStart()`, the startup validation forces a build at boot regardless. Note a subtlety: on **reload** ([04-Reload.md](04-Reload.md)), `IOptionsMonitor` re-validates the new value — but a validation failure on reload *throws when `CurrentValue` is accessed*, it doesn't crash the running app. So validation protects startup strongly; for reload, ensure your reload-reading code handles a possible `OptionsValidationException` (or simply don't make hot-reloaded settings safety-critical).

---

## FluentValidation (optional, for rich rules)

For complex validation you may prefer FluentValidation. Bridge it to options via a small `IValidateOptions<T>` adapter:

```csharp
public class FluentValidateOptions<T>(IServiceProvider sp) : IValidateOptions<T> where T : class {
    public ValidateOptionsResult Validate(string? name, T options) {
        using var scope = sp.CreateScope();
        var validator = scope.ServiceProvider.GetRequiredService<IValidator<T>>();
        var result = validator.Validate(options);
        return result.IsValid
            ? ValidateOptionsResult.Success
            : ValidateOptionsResult.Fail(result.Errors.Select(e => e.ErrorMessage));
    }
}
```

This keeps the host's "fail at startup" behavior while letting you express rules in FluentValidation's fluent syntax. Use the built-in mechanisms (DataAnnotations / `Validate` / `IValidateOptions`) unless you already use FluentValidation elsewhere and want consistency.

---

## Common gotchas

### Forgetting `ValidateOnStart()`

The single most common mistake: validation is configured but runs *lazily*, so bad config doesn't surface until the option is first read — often in production. Always chain `.ValidateOnStart()`.

### Expecting binding mismatches to throw

A config key that doesn't match a property name doesn't error — it silently leaves the default ([01-IConfiguration.md](01-IConfiguration.md)). Validation (e.g., `[Required]`) is what turns "silently defaulted" into "loud failure." Validate required settings explicitly.

### Validating in the constructor of the consuming service

Don't scatter `if (options.Host is null) throw ...` across services. Centralize in the options validator so it runs once, at startup, with a clear message.

### Reload bypassing validation expectations

`IOptionsMonitor` re-validates on reload, but the failure throws on access, not at reload time — it won't crash the app. Don't rely on validation to protect hot-reloaded safety-critical values.

### Only the first DataAnnotations error

`ValidateDataAnnotations` reports the failures it finds; for guaranteed "report everything at once" with custom messages, `IValidateOptions<T>` gives you full control over the failure list.

---

## Summary

- **Validate options** so misconfiguration becomes an immediate, clear **startup failure** (fail fast) instead of a latent runtime crash.
- Three mechanisms: **`ValidateDataAnnotations()`** (declarative attributes — `[Required]`/`[Range]`/...), **`.Validate(predicate, msg)`** (cross-field rules), and **`IValidateOptions<T>`** (a DI service: dependency-aware, multiple messages, named-options support).
- **Always add `.ValidateOnStart()`** — without it, validation runs *lazily* on first `.Value` access, so bad config surfaces in production rather than at boot.
- On **reload**, `IOptionsMonitor` re-validates but throws on access (doesn't crash the app) — don't rely on validation for hot-reloaded safety-critical settings.
- Bridge **FluentValidation** via a small `IValidateOptions<T>` adapter if you need richer rules.

→ Next: [07-Secrets.md](07-Secrets.md)
