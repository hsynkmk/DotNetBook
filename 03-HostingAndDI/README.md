# Chapter 03 — Hosting & Dependency Injection

> The shared application model: `IHostBuilder`, `IServiceProvider`, `IConfiguration`, `IOptions`, `ILogger`. Every ASP.NET Core, console worker, and gRPC service uses the same machinery. Master this once.

**Prerequisites**: comfortable with C# generics, interfaces, async.

**Time to read**: ~6-8 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-GenericHost.md](01-GenericHost.md) | `Host.CreateApplicationBuilder`, `IHost`, the startup lifecycle, IHostedService, BackgroundService. |
| [02-DependencyInjection.md](02-DependencyInjection.md) | `IServiceProvider`, registration patterns (Singleton/Scoped/Transient), `IServiceCollection`, factory patterns. |
| [03-Lifetimes.md](03-Lifetimes.md) | Lifetime rules, captive dependencies, scope creation, IServiceScopeFactory, common bugs. |
| [04-OpenGenericsInDI.md](04-OpenGenericsInDI.md) | Registering open generics (`AddSingleton(typeof(IRepository<>), typeof(Repo<>))`), why this matters. |
| [05-Keyed.md](05-Keyed.md) | Keyed services (.NET 8+) — `AddKeyedSingleton`, `[FromKeyedServices]`. |
| [06-Decorate.md](06-Decorate.md) | Decorator patterns via Scrutor, manual decoration, when to decorate. |
| [07-Configuration.md](07-Configuration.md) | `IConfiguration`, providers (JSON, env, user secrets, Azure Key Vault), reload, change tokens. |
| [08-Options.md](08-Options.md) | `IOptions<T>`, `IOptionsSnapshot<T>`, `IOptionsMonitor<T>`, validation. |
| [09-LoggingPrimer.md](09-LoggingPrimer.md) | `ILogger<T>` basics — see Chapter 12 (Observability) for the deep version. |
| [10-Validation.md](10-Validation.md) | Validate options at startup; FluentValidation; data annotations. |
| [Questions.md](Questions.md) | Drilling questions. |
| [Coding.md](Coding.md) | Build a console worker, register services, configure from multiple sources. |

---

## Learning objectives

After this chapter you should be able to:
- Stand up a host with logging, config, and DI in minutes.
- Choose appropriate service lifetimes; identify captive-dependency bugs.
- Use the Options pattern correctly (and validate at startup).
- Run `BackgroundService` for long-running work.
- Layer multiple configuration sources with overrides.

→ Begin: [01-GenericHost.md](01-GenericHost.md)
