# Feature Flags

## Decoupling deployment from release

A **feature flag** (feature toggle) is a switch that turns a piece of functionality on or off **at runtime, without deploying code**. Its central value is **decoupling deployment from release**: you can ship code with a feature **off**, deploy it safely (it's dormant), then **enable** it later — for everyone, a percentage of users, or specific groups — and **roll it back instantly** by flipping the switch (no redeploy). This enables progressive/canary rollouts ([Ch19 §09](../19-Deployment/09-CICD.md)), A/B testing, kill-switches for risky features, and trunk-based development (merge incomplete features behind a flag). In .NET, the **`Microsoft.FeatureManagement`** library provides this, and it integrates with **Azure App Configuration** ([Ch20 §09](../20-AzureIntegration/09-AppConfig.md)) for centralized control.

```csharp
builder.Services.AddFeatureManagement();   // reads flags from configuration ([Ch13])

public class CheckoutController(IFeatureManager features) {
    public async Task<IActionResult> Index() =>
        await features.IsEnabledAsync("NewCheckout") ? View("NewCheckout") : View("Checkout");
}
```

```razor
@* In Razor views: *@
<feature name="NewCheckout"><partial name="_NewCheckout" /></feature>
```

---

## Why flags matter

- **Deploy ≠ release** — ship code dark, enable when ready. Deployment becomes low-risk (the code is inert until flagged on); release becomes a config change, not a deploy.
- **Instant rollback** — a misbehaving feature is disabled by flipping the flag, far faster and safer than rolling back a deployment.
- **Progressive rollout** — enable for 5% → 25% → 100%, watching telemetry ([Ch12](../12-Observability/README.md)) at each step to catch problems with limited blast radius (canary — [Ch19 §09](../19-Deployment/09-CICD.md)).
- **Targeting / A-B testing** — enable for specific users/segments to test or gradually onboard.
- **Trunk-based development** — merge incomplete work behind an off flag instead of long-lived branches.

This is why feature flags are a cornerstone of modern continuous delivery.

---

## Feature filters

`Microsoft.FeatureManagement` supports **feature filters** — flags enabled conditionally, not just on/off:

- **Percentage filter** — enable for X% of requests/users (gradual rollout).
- **Targeting filter** — enable for named users/groups, with percentage rollout per group (e.g., "all internal users + 10% of everyone else").
- **Time window filter** — enable between dates (scheduled releases, promotions).
- **Custom filters** — your own logic (per tenant — [07-MultiTenancy.md](07-MultiTenancy.md), per region, etc.).

```json
{ "FeatureManagement": {
    "NewCheckout": {
      "EnabledFor": [
        { "Name": "Percentage", "Parameters": { "Value": 25 } }
      ] } } }
```

Filters turn a binary toggle into a **controlled rollout mechanism** — the same flag drives "off → 25% → everyone" by changing configuration.

---

## Where flags live

Flags are read via the standard configuration system ([Ch13](../13-Configuration/README.md)), so their **source** can be:

- **`appsettings.json`** — simple, but changing a flag means a redeploy/restart (loses the "change without deploy" benefit). Fine for static toggles.
- **Azure App Configuration** ([Ch20 §09](../20-AzureIntegration/09-AppConfig.md)) — **centralized, dynamic**: change a flag in the portal and running apps pick it up (via refresh — [Ch13 §04](../13-Configuration/04-Reload.md)) **without redeploy** — the full benefit. The recommended backing for real feature management.
- A database or third-party flag service (LaunchDarkly, etc.) for advanced targeting/auditing.

For the decouple-deploy-from-release benefit, the flag source must be **changeable at runtime** (App Config or a flag service) — a flag baked into `appsettings.json` only toggles on redeploy.

---

## Operational discipline

Flags add power but also **debt** if unmanaged:

- **Remove stale flags** — a fully-rolled-out feature's flag is dead weight (and a source of confusion/bugs from the now-unused off path). Treat flag removal as part of finishing a feature.
- **Avoid flag explosion / interaction** — many interacting flags create a combinatorial test matrix; keep flags few and short-lived where possible.
- **Test both paths** — code behind a flag means *both* on and off states must work and be tested ([Ch17](../17-Testing/README.md)) until the flag is removed.
- **Audit/permissions** — flag changes alter production behavior; control who can change them and log changes (App Config / a flag service provides this).

Flags are a means to safer delivery, not permanent forks — the discipline is to **roll out, then clean up**.

---

## Common gotchas

### Flags in `appsettings.json` for dynamic control

A flag baked into deployed config only changes on redeploy — losing the "release without deploy" benefit. Use **App Configuration** (or a flag service) for runtime-changeable flags ([Ch20 §09](../20-AzureIntegration/09-AppConfig.md)).

### Stale flags never removed

Long-lived flags accumulate, complicating code and testing and risking bugs from unused branches. Remove a flag once its feature is fully rolled out — finish the cleanup.

### Not testing the off path

Code behind a flag has two states; testing only the on path means the off (or rollback) path may be broken. Test both until the flag is gone.

### Flag explosion / interactions

Many interacting flags create an unmanageable test/behavior matrix. Keep flags few, focused, and short-lived; avoid deep nesting of flag conditions.

### No control/audit over flag changes

A flag change alters production behavior instantly — without access control/auditing, that's risky. Restrict who can toggle flags and log changes.

---

## Summary

- A **feature flag** turns functionality on/off **at runtime without deploying** — its core value is **decoupling deployment from release**: ship code dark, enable when ready (for all / a %/ specific groups), and **roll back instantly** by flipping the switch.
- In .NET, **`Microsoft.FeatureManagement`** (`IFeatureManager` / `<feature>` tags) reads flags from configuration; **feature filters** (percentage, targeting, time window, custom) turn a toggle into a **controlled rollout** (canary, A/B, scheduled).
- For the full benefit the flag source must be **runtime-changeable** — **Azure App Configuration** ([Ch20 §09](../20-AzureIntegration/09-AppConfig.md)) gives centralized, dynamic flags picked up without redeploy; flags in `appsettings.json` only change on deploy.
- Apply **operational discipline**: **remove stale flags**, avoid flag explosion, **test both on/off paths**, and **control/audit** flag changes — flags enable safer continuous delivery ([Ch19 §09](../19-Deployment/09-CICD.md)), not permanent forks.

→ Next: [09-Versioning.md](09-Versioning.md)
