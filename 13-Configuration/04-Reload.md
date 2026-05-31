# Configuration Reload

## Picking up config changes at runtime

Some configuration can change while the app runs — a tuning value, a feature flag, a log level. Configuration **reload** lets the app observe these changes **without a restart**. It's driven by **change tokens** under the hood, and surfaced cleanly through **`IOptionsMonitor<T>`** ([05-Options.md](05-Options.md)). Not all sources reload, and reload has subtleties worth understanding.

```csharp
builder.Configuration.AddJsonFile("appsettings.json", optional: false, reloadOnChange: true);  // watch the file
```

> Reload and `IOptionsMonitor` are introduced in [Ch03 §07–08](../03-HostingAndDI/07-Configuration.md). This file goes deeper on the change-token mechanism and what reloads (vs not).

---

## What reloads, what doesn't

| Source | Reloads at runtime? |
|---|---|
| **JSON files** (`reloadOnChange: true`) | **yes** — re-read when the file changes on disk |
| **Azure App Configuration / Key Vault** | yes — via polling/refresh ([02-Providers.md](02-Providers.md)) |
| **Environment variables** | **no** — read once at startup |
| **Command-line arguments** | **no** — read once at startup |

The key distinction: **file-based (and some cloud) providers can reload; environment variables and command-line args are read once at startup** and never change for the process's lifetime. So a setting you expect to change at runtime must come from a reloadable source (a watched file or a cloud config service), not an env var. This often surprises people — changing an env var on a running process has no effect.

---

## The change-token mechanism

Reload is built on **`IChangeToken`**: a provider that supports reload exposes a token that **signals** when the underlying source changes. The configuration system (and `IOptionsMonitor`) registers callbacks on it:

```csharp
// Low-level (you rarely use this directly — IOptionsMonitor wraps it):
ChangeToken.OnChange(
    () => config.GetReloadToken(),                 // get a fresh token (it fires once, then you re-register)
    () => logger.LogInformation("Config reloaded"));   // callback on change
```

A `reloadOnChange: true` JSON provider watches the file (via a file-system watcher); on change it re-reads the file and **fires its change token**, which triggers re-binding of options and any registered callbacks. You almost never consume `GetReloadToken()` directly — **`IOptionsMonitor<T>`** is the high-level API that surfaces reloaded config to your services (below).

---

## Consuming reloaded config: `IOptionsMonitor<T>`

The clean way to react to config changes is `IOptionsMonitor<T>` ([05-Options.md](05-Options.md)) — it gives the **current** (live-reloaded) value and an `OnChange` callback:

```csharp
public class PollingWorker(IOptionsMonitor<WorkerOptions> monitor) : BackgroundService {
    protected override async Task ExecuteAsync(CancellationToken ct) {
        monitor.OnChange(o => logger.LogInformation("Interval changed to {Interval}", o.Interval));
        while (!ct.IsCancellationRequested) {
            var interval = monitor.CurrentValue.Interval;   // always the LATEST value
            await Task.Delay(interval, ct);
            await PollAsync(ct);
        }
    }
}
```

`IOptionsMonitor<T>.CurrentValue` returns the freshest bound value (re-bound on reload), and `OnChange` fires when config changes — so a singleton/background service ([Ch08](../08-BackgroundProcessing/README.md)) picks up new settings without a restart. This is the correct accessor for **singletons** that need live config; `IOptionsSnapshot<T>` (scoped) reflects reloads per request, and `IOptions<T>` (singleton, read once) **never** reloads ([05-Options.md](05-Options.md)).

---

## When (not) to use reload

Reload is useful but not free — and not always appropriate:

- **Good for**: log-level changes (tune verbosity live), feature flags, non-critical tuning (timeouts, intervals, batch sizes), operational toggles — things you want to change without a deploy.
- **Be careful with**: anything where a mid-flight change could cause inconsistency. A value read at different times within one operation might differ; capture it once per operation if consistency matters.
- **Not for**: things that require re-initialization (a connection string change doesn't re-open existing connections; a DI registration can't change at runtime). Some changes genuinely need a restart.

In container/orchestrated environments, the common practice is **immutable config per deployment** — change config by deploying a new version (new env vars / new image), not by mutating a running instance. Reload matters most for file/cloud-config-driven dynamic values (log levels, feature flags via App Configuration). Decide deliberately whether a setting should be dynamic (reloadable source + `IOptionsMonitor`) or fixed-per-deploy (env var, read once).

---

## Common gotchas

### Expecting env vars / command line to reload

They're read **once at startup** and never change. A setting that must change at runtime must come from a reloadable source (watched file, cloud config), not an env var.

### Using `IOptions<T>` and expecting reload

`IOptions<T>` is computed once and never updates. For reload, use `IOptionsMonitor<T>` (singletons) or `IOptionsSnapshot<T>` (per request) ([05-Options.md](05-Options.md)).

### Injecting `IOptionsSnapshot` into a singleton

`IOptionsSnapshot` is scoped — a captive-dependency bug in a singleton. Use `IOptionsMonitor` in singletons/background services ([Ch03 §08](../03-HostingAndDI/08-Options.md)).

### Inconsistent reads within an operation

With live reload, two reads of `CurrentValue` in one operation could differ across a reload. If consistency matters, capture the value once at the start of the operation.

### Reloading config that needs re-initialization

Changing a connection string in config doesn't re-open existing connections; some changes require a restart. Don't assume reload re-initializes everything.

### Custom provider without change-token support

A custom reloadable source must raise change tokens, or reload won't trigger ([02-Providers.md](02-Providers.md)).

---

## Summary

- Configuration **reload** lets the app pick up changes without a restart, built on **`IChangeToken`** (a reloadable provider signals when its source changes, triggering re-binding).
- **File-based (and some cloud) providers reload** (`reloadOnChange: true`); **environment variables and command-line args are read once at startup** and never change — a setting meant to change at runtime must come from a reloadable source.
- Consume reloaded config via **`IOptionsMonitor<T>`** (`CurrentValue` + `OnChange`) for singletons/background services; `IOptionsSnapshot<T>` per request; **`IOptions<T>` never reloads**.
- Use reload for **log levels, feature flags, non-critical tuning**; be careful with mid-operation consistency and changes needing re-initialization. In containers, prefer **immutable config per deployment** for most settings, reload for genuinely dynamic ones.

→ Next: [05-Options.md](05-Options.md)
