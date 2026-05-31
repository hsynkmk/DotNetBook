# Validating Options at Startup

## Fail fast on misconfiguration

A misconfigured app should fail **immediately at startup** with a clear message — not hours later with a cryptic error deep in a request when it finally reads a bad value. Options validation enforces configuration constraints, and `ValidateOnStart()` makes violations a startup failure.

```csharp
builder.Services.AddOptions<SmtpOptions>()
    .Bind(builder.Configuration.GetSection("Smtp"))
    .ValidateDataAnnotations()                                  // [Required], [Range], etc.
    .Validate(o => o.Port is > 0 and < 65536, "Port must be 1–65535")
    .ValidateOnStart();                                          // ← fail at startup, not first use
```

Without `ValidateOnStart()`, validation runs **lazily** on first access to `options.Value` — so a bad config might not surface until a specific code path runs in production. With it, the host **refuses to start** and tells you exactly what's wrong. This is the single most valuable configuration-robustness habit.

---

## Data annotations validation

Decorate the options POCO with validation attributes (`System.ComponentModel.DataAnnotations`):

```csharp
public class SmtpOptions {
    [Required] public string Host { get; set; } = "";
    [Range(1, 65535)] public int Port { get; set; }
    [Required, EmailAddress] public string FromAddress { get; set; } = "";
    [Range(1, 300)] public int TimeoutSeconds { get; set; } = 30;
}

builder.Services.AddOptions<SmtpOptions>()
    .Bind(builder.Configuration.GetSection("Smtp"))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

`ValidateDataAnnotations()` checks the attributes when the options are validated. Common attributes: `[Required]`, `[Range]`, `[StringLength]`, `[EmailAddress]`, `[Url]`, `[RegularExpression]`. Declarative and self-documenting — the constraints live on the type.

---

## Custom validation logic

For rules attributes can't express, use `.Validate(...)` or a full `IValidateOptions<T>`:

```csharp
// Inline predicate (simple cross-property rules)
builder.Services.AddOptions<RetryOptions>()
    .Bind(config.GetSection("Retry"))
    .Validate(o => o.MaxAttempts >= 1, "MaxAttempts must be at least 1")
    .Validate(o => o.BaseDelay < o.MaxDelay, "BaseDelay must be less than MaxDelay")
    .ValidateOnStart();
```

```csharp
// IValidateOptions<T> — for complex validation, reusable, can depend on other services
public class SmtpOptionsValidator : IValidateOptions<SmtpOptions> {
    public ValidateOptionsResult Validate(string? name, SmtpOptions o) {
        var failures = new List<string>();
        if (string.IsNullOrWhiteSpace(o.Host)) failures.Add("Host is required.");
        if (o.UseSsl && o.Port == 25) failures.Add("Port 25 with SSL is unusual; check config.");
        return failures.Count > 0
            ? ValidateOptionsResult.Fail(failures)
            : ValidateOptionsResult.Success;
    }
}
builder.Services.AddSingleton<IValidateOptions<SmtpOptions>, SmtpOptionsValidator>();
```

`.Validate(predicate, message)` handles simple/cross-property rules inline. `IValidateOptions<T>` is for richer validation (multiple failures, reusable, registered in DI). Both run when the options are validated (at startup with `ValidateOnStart`).

---

## FluentValidation (richer rules)

For complex validation with a fluent API and great messages, the **FluentValidation** library integrates with options:

```csharp
public class SmtpOptionsValidator : AbstractValidator<SmtpOptions> {
    public SmtpOptionsValidator() {
        RuleFor(x => x.Host).NotEmpty().WithMessage("SMTP host is required");
        RuleFor(x => x.Port).InclusiveBetween(1, 65535);
        RuleFor(x => x.FromAddress).NotEmpty().EmailAddress();
        RuleFor(x => x.TimeoutSeconds).GreaterThan(0).When(x => x.UseSsl);
    }
}
```

FluentValidation shines for conditional rules (`.When`), collections, nested objects, and readable composite messages. Wire it into the options validation pipeline (via a small `IValidateOptions<T>` adapter or a community integration package). It's also the common choice for **request/model validation** in ASP.NET Core ([Ch04 §08](../04-AspNetCore/README.md)).

---

## Why startup validation matters

```
Without ValidateOnStart:
  app starts "fine" → 3 hours later a request hits the SMTP path →
  cryptic NullReference/format error in production → on-call paged → slow diagnosis.

With ValidateOnStart:
  app refuses to start → log: "SmtpOptions.Host is required" →
  caught in CI / on deploy / immediately → fixed in minutes.
```

`ValidateOnStart()` shifts configuration errors **left** — from a random runtime failure to a deterministic startup failure with a precise message. In containerized/orchestrated environments, a failing-to-start container is caught by health checks and rollout halts, preventing a broken deploy. **Always validate critical options on start.**

---

## Validation vs guard clauses vs request validation

| Concern | Tool |
|---|---|
| **Configuration** is valid (startup) | Options validation + `ValidateOnStart` (this file) |
| **Method arguments** (programmer errors) | Guard clauses (`ArgumentNullException.ThrowIfNull` — CSharpBook Ch17 §08) |
| **User input / request bodies** | Model validation (DataAnnotations/FluentValidation in ASP.NET — [Ch04 §08](../04-AspNetCore/README.md)) |

These operate at different boundaries: config validation ensures the *app is configured correctly* (fail at startup); request validation ensures *incoming data is acceptable* (return 400). Don't conflate them.

---

## Common gotchas

### Forgetting `ValidateOnStart()`

Without it, validation is lazy (first access to `.Value`) — bad config surfaces at runtime, not startup. Add `ValidateOnStart()` for critical options.

### Validating but never reading the options

If nothing ever accesses `IOptions<T>.Value` and you didn't call `ValidateOnStart`, lazy validation never runs. `ValidateOnStart` forces it regardless.

### Data annotations not firing

`[Required]` etc. only run if you call `.ValidateDataAnnotations()`. Attributes alone do nothing.

### Defaults masking missing config

A property with a default (`= 30`) won't trigger `[Required]` and silently uses the default if config is absent. Decide whether absence should be an error (no default + `[Required]`) or acceptable (a default).

### Over-validating non-critical options

Not every option needs strict startup validation. Focus on values whose misconfiguration breaks the app (connection strings, endpoints, credentials).

---

## Summary

- Validate options so misconfiguration **fails fast at startup**, not deep in a runtime path — `AddOptions<T>().Bind(...).ValidateDataAnnotations().Validate(...).ValidateOnStart()`.
- **Data annotations** (`[Required]`, `[Range]`, …) for declarative rules; **`.Validate(predicate, msg)`** or **`IValidateOptions<T>`** for custom/cross-property logic; **FluentValidation** for rich, conditional rules.
- **`ValidateOnStart()`** is the key — it turns a lazy, runtime config error into a deterministic startup failure with a clear message (caught by CI/health checks before serving traffic).
- Distinguish **config validation** (startup) from **argument guards** (programmer errors) and **request/model validation** (user input, [Ch04](../04-AspNetCore/README.md)).
- Validate the options that *matter* (connection strings, endpoints, credentials); allow sensible defaults elsewhere.

→ Next: [Questions.md](Questions.md)
