# Azure App Configuration

## Centralized configuration and feature flags

**Azure App Configuration** is a managed service for **centralizing configuration** across multiple apps/services and managing **feature flags**. Instead of each service carrying its own `appsettings.json` (with duplicated/drifting values), shared settings live in one place, are updated without redeploying, and flow into apps via a **configuration provider** ([Ch13 §02](../13-Configuration/02-Providers.md)). It complements Key Vault ([07-KeyVault.md](07-KeyVault.md)) (which handles *secrets*) — App Config holds *non-secret* settings and feature flags, and can reference Key Vault for the secret bits. It's especially valuable for **multi-service** systems where consistent, dynamic configuration matters.

```csharp
builder.Configuration.AddAzureAppConfiguration(options => {
    options.Connect(new Uri("https://myconfig.azconfig.io"), new DefaultAzureCredential())   // keyless
           .Select("MyApp:*", labelFilter: builder.Environment.EnvironmentName)               // env-specific
           .ConfigureKeyVault(kv => kv.SetCredential(new DefaultAzureCredential()))           // resolve KV refs
           .UseFeatureFlags();                                                                 // feature flags
});
```

---

## Why centralize configuration

For a single app, `appsettings.json` is fine ([Ch13](../13-Configuration/README.md)). But across **many services/instances**, problems appear:

- **Duplication/drift** — the same setting copied into many files diverges over time.
- **Redeploy to change config** — a setting change means rebuilding/redeploying each app.
- **No central view** — no single place to see/audit what's configured.

App Configuration solves these: settings live **centrally**, are **changed without redeploy** (apps refresh dynamically), are **versioned/audited**, and can be **labeled per environment** (dev/staging/prod) so one store serves all environments. Apps read them through the normal `IConfiguration` model — no app-specific code beyond adding the provider.

---

## Keys, labels, and environment separation

App Config stores **key-value** pairs, with **labels** to distinguish variants of the same key — the idiomatic way to separate environments:

```csharp
// Same key "MyApp:Timeout", different value per label (environment):
options.Select("MyApp:*", labelFilter: "Production");   // load only Production-labeled values
```

A key like `MyApp:Timeout` can have a `Development` label value and a `Production` label value; the app selects by its environment ([Ch13 §03](../13-Configuration/03-Layering.md)). This gives clean per-environment configuration from a single store, instead of separate files per environment.

---

## Dynamic refresh

A standout feature: apps can **refresh configuration at runtime** without restart, picking up central changes. You configure a **sentinel key** (a key whose change signals "config updated") and a refresh interval:

```csharp
options.ConfigureRefresh(refresh => refresh
    .Register("MyApp:Settings:Sentinel", refreshAll: true)   // watch a sentinel key
    .SetRefreshInterval(TimeSpan.FromSeconds(30)));
builder.Services.AddAzureAppConfiguration();   // middleware that triggers refresh
app.UseAzureAppConfiguration();
```

When you update the sentinel key in App Config, apps refresh their settings (read via `IOptionsMonitor<T>` for the live value — [Ch13 §05](../13-Configuration/05-Options.md)). This lets you **change behavior centrally and have it propagate** to running apps within the interval — powerful for tuning timeouts, toggling behavior, or coordinated config changes across a fleet.

---

## Feature flags / Feature Management

App Configuration integrates with **Feature Management** (`Microsoft.FeatureManagement`) — declarative **feature flags** that let you turn features on/off (or roll them out gradually) **without deploying code**:

```csharp
public class CheckoutController(IFeatureManager features) {
    public async Task<IActionResult> Index() {
        if (await features.IsEnabledAsync("NewCheckout"))   // flag controlled centrally
            return View("NewCheckout");
        return View("Checkout");
    }
}
```

```razor
@* In views: *@
<feature name="NewCheckout"><partial name="_NewCheckout" /></feature>
```

Feature flags support:
- **Simple on/off** toggles, changeable in App Config without redeploy.
- **Percentage rollouts** (enable for X% of users — gradual release).
- **Targeting** (specific users/groups) and time-windows.

This decouples **deployment** from **release** — ship code with a feature off, then enable it (for some/all users) via the flag, and roll back instantly by toggling it off. It's the foundation of progressive delivery / canary-style feature rollout (complementing deployment strategies — [Ch19 §09](../19-Deployment/09-CICD.md)).

---

## Common gotchas

### Putting secrets in App Configuration

App Config is for **non-secret** settings; secrets belong in **Key Vault** ([07-KeyVault.md](07-KeyVault.md)). Store secrets in Key Vault and use **Key Vault references** from App Config (resolved via managed identity) — don't put raw secrets in App Config.

### Expecting refresh without a sentinel/middleware

Dynamic refresh requires configuring a **sentinel key** + refresh interval and the refresh middleware; without it, config is read once at startup. Set up `ConfigureRefresh` and `UseAzureAppConfiguration`.

### Reading refreshed values via `IOptions<T>`

`IOptions<T>` reads once and won't reflect a refresh. Use `IOptionsMonitor<T>` for values that change at runtime ([Ch13 §05](../13-Configuration/05-Options.md)).

### Confusing deployment with release

Feature flags let you **release** independently of **deploy** — but only if you wire flags into the code paths. Use `IFeatureManager`/`<feature>` tags so flags actually gate behavior.

### Connection strings over managed identity

Connect to App Config with **`DefaultAzureCredential` + RBAC**, not an access-key connection string ([01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)) — keyless, no secret to leak.

---

## Summary

- **Azure App Configuration** centralizes **non-secret** configuration and **feature flags** across services — solving duplication/drift, enabling **change-without-redeploy**, central audit, and **per-environment labels** — flowing into apps via a **configuration provider** ([Ch13 §02](../13-Configuration/02-Providers.md)).
- It stores **key-value** pairs with **labels** (the idiomatic per-environment separation) and supports **dynamic refresh** (sentinel key + interval + middleware) so running apps pick up central changes via **`IOptionsMonitor<T>`** without restart.
- **Feature Management** provides declarative **feature flags** (on/off, **percentage rollout**, targeting) that **decouple deployment from release** — ship code with a feature off, enable it centrally, roll back by toggling.
- Use it **with Key Vault** (App Config for settings/flags, **Key Vault for secrets** via references — [07-KeyVault.md](07-KeyVault.md)); connect with **managed identity** ([01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)).

→ Next: [10-AppInsights.md](10-AppInsights.md)
