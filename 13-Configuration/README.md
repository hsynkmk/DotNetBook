# Chapter 13 — Configuration

> Reading settings from JSON, environment variables, command line, Azure Key Vault, user secrets. Validating and reloading. The Options pattern in depth.

**Prerequisites**: Chapter 03 (Hosting & DI). Most of this is introduced there; this chapter goes deep.

**Time to read**: ~3-4 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-IConfiguration.md](01-IConfiguration.md) | The flat key-value model, sections, indexers. |
| [02-Providers.md](02-Providers.md) | JSON, environment variables, command line, user secrets, key vault, custom providers. |
| [03-Layering.md](03-Layering.md) | Override order, environment-specific (`appsettings.Development.json`). |
| [04-Reload.md](04-Reload.md) | Reload on file change, `IChangeToken`, `IOptionsMonitor<T>`. |
| [05-Options.md](05-Options.md) | The Options pattern in detail: `IOptions<T>`, `IOptionsSnapshot<T>`, `IOptionsMonitor<T>`. |
| [06-Validation.md](06-Validation.md) | Validating options at startup: DataAnnotations, FluentValidation, custom. |
| [07-Secrets.md](07-Secrets.md) | User Secrets, Azure Key Vault, Secret Manager, never-in-repo patterns. |
| [Questions.md](Questions.md) | Drilling. |
| [Coding.md](Coding.md) | Layer multiple sources, validate options at startup, hot-reload. |

→ Begin: [01-IConfiguration.md](01-IConfiguration.md)
