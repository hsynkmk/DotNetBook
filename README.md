# The .NET Book — Runtime, Libraries, Frameworks

> A comprehensive guide to the **.NET platform** as it stands at **.NET 10 (November 2025, LTS)** — companion to `CSharpBook/` which covers the **language**. This book covers everything you need to build, ship, and operate modern .NET applications.

This book is for:
- **Developers learning .NET** alongside C#. Read CSharpBook for the language; read this for the platform.
- **C# developers expanding into web, mobile, cloud, performance, or operations.** Each chapter goes deep.
- **Architects** making framework/library choices. Decision matrices throughout.
- **Senior engineers** preparing for design interviews or modernizing legacy stacks.

Every concept includes:
- *What it is* + mental model
- *Why it exists* (the problem it solves)
- *How it works under the hood* (runtime, IL, metadata, network protocols)
- Common patterns + idioms
- Gotchas and pitfalls
- Performance notes
- Drilling questions and coding problems

---

## 📚 Table of Contents

### Level 0 — Foundation
- [Chapter 00 — Overview](00-Overview/README.md) — What .NET is today, the history, the SDKs and runtimes, what's in the box.

### Level 1 — Runtime & BCL
- [Chapter 01 — Runtime & CLR](01-Runtime/README.md) — CoreCLR internals, JIT, GC deep dive, type system, AssemblyLoadContext.
- [Chapter 02 — Base Class Library](02-BCL/README.md) — System.*, System.IO, System.Text, System.Diagnostics, System.Threading, System.Net.

### Level 2 — Hosting, DI, Config
- [Chapter 03 — Hosting & DI](03-HostingAndDI/README.md) — Generic Host, IServiceProvider, IConfiguration, IOptions, ILogger, IHostedService.

### Level 3 — Building Applications
- [Chapter 04 — ASP.NET Core](04-AspNetCore/README.md) — Middleware, routing, Minimal APIs, MVC, Razor Pages, model binding, OpenAPI.
- [Chapter 05 — Entity Framework Core](05-EFCore/README.md) — DbContext, querying, change tracking, migrations, performance.
- [Chapter 06 — Data Access & Caching](06-DataAndCaching/README.md) — Dapper, ADO.NET, IMemoryCache, IDistributedCache (Redis), output caching.
- [Chapter 07 — Messaging](07-Messaging/README.md) — Channel<T>, MassTransit, RabbitMQ, Azure Service Bus, Kafka.
- [Chapter 08 — Background Processing](08-BackgroundProcessing/README.md) — IHostedService, BackgroundService, Worker Services, queued jobs, scheduling, Hangfire, Quartz.NET.
- [Chapter 09 — Networking & HTTP](09-NetworkingAndHttp/README.md) — HttpClient, IHttpClientFactory, gRPC, SignalR, WebSockets.
- [Chapter 10 — Identity & Security](10-Identity/README.md) — ASP.NET Core Identity, JWT, OAuth/OIDC, Data Protection.

### Level 4 — Cross-Cutting Concerns
- [Chapter 11 — Resilience](11-Resilience/README.md) — Polly, Microsoft.Extensions.Resilience, retry, circuit breaker, fallback, hedging.
- [Chapter 12 — Observability](12-Observability/README.md) — Logging, metrics, traces, ILogger, Serilog, OpenTelemetry.
- [Chapter 13 — Configuration](13-Configuration/README.md) — IConfiguration, environments, secrets, IOptions, validation.

### Level 5 — Client-Side
- [Chapter 14 — Blazor](14-Blazor/README.md) — Server, WebAssembly, Hybrid, components, state management.
- [Chapter 15 — MAUI](15-MAUI/README.md) — Cross-platform mobile + desktop UI.
- [Chapter 16 — Desktop](16-Desktop/README.md) — WPF, WinUI 3, WinForms — when each fits.

### Level 6 — Quality & Ship
- [Chapter 17 — Testing](17-Testing/README.md) — xUnit, integration tests, TestContainers, bUnit, WebApplicationFactory.
- [Chapter 18 — .NET Aspire](18-Aspire/README.md) — Cloud-native orchestration: AppHost app model, service discovery, integrations, the dashboard, deployment.
- [Chapter 19 — Deployment](19-Deployment/README.md) — Docker, Kubernetes, Native AOT, ReadyToRun, single-file, publish modes.
- [Chapter 20 — Azure Integration](20-AzureIntegration/README.md) — Functions, App Service, Service Bus, Storage, Cosmos DB.
- [Chapter 21 — Performance & Tooling](21-Performance/README.md) — BenchmarkDotNet, profilers, dotnet-counters, dotnet-trace, dotnet-dump.

### Level 7 — Expert
- [Chapter 22 — Best Practices & Patterns](22-BestPractices/README.md) — Project structure, repository patterns, anti-patterns at scale.

### Reference
- [GLOSSARY](GLOSSARY.md) — Every platform term defined.
- [INDEX](INDEX.md) — Topic → file index.

---

## 🛣️ Learning Paths

### "I know C# but I'm new to .NET ecosystem" (~4 weeks)
1. Chapter 00 (Overview) — 1 day
2. Chapter 03 (Hosting & DI) — 3 days
3. Chapter 04 (ASP.NET Core) — 1 week
4. Chapter 05 (EF Core) — 1 week
5. Chapter 12 (Observability) — 3 days
6. Chapter 17 (Testing) — 3 days
7. Chapter 18 (.NET Aspire) — to tie the services together

### "Senior backend dev moving to .NET" (~2 weeks)
- Skim 00, 03 for foundations.
- Deep dive 04 (ASP.NET Core), 05 (EF Core), 09 (HTTP/gRPC).
- Read 08 (Background Processing), 11 (Resilience), 12 (Observability).
- Read 18 (Aspire) and 19 (Deployment) to compose and ship.
- Apply via the coding problems in each chapter's `Coding.md`.

### "Mobile / desktop developer" (~3 weeks)
- Chapter 14 (Blazor) — for hybrid UI.
- Chapter 15 (MAUI) — for cross-platform mobile.
- Chapter 16 (Desktop) — for Windows.
- Cross-reference back to Chapter 03 (DI works the same everywhere).

### "Performance / runtime nerd" (~2 weeks)
- Chapter 01 (Runtime & CLR) — deeply.
- Chapter 02 (BCL) — selected namespaces (Threading, IO, Net).
- Chapter 21 (Performance & Tooling).
- Cross-link to CSharpBook Chapter 09 (Memory & Performance).

### "Cloud-native / DevOps focus" (~3 weeks)
- Chapter 13 (Configuration).
- Chapter 12 (Observability).
- Chapter 18 (.NET Aspire) — the cloud-native composition model.
- Chapter 19 (Deployment).
- Chapter 20 (Azure Integration).
- Chapter 11 (Resilience).

---

## 📋 Each Chapter Contains

```
NN-ChapterName/
├── README.md             ← chapter overview (you are here)
├── 01-SubTopic1.md       ← deep dive #1
├── 02-SubTopic2.md       ← deep dive #2
├── ...                   ← (typically 5-12 sub-files)
├── Questions.md          ← drilling questions with hidden answers
└── Coding.md             ← 10-15 hands-on coding problems
```

Sub-files use the same structure as CSharpBook:
1. *What it is* + mental model
2. *Why it exists*
3. *Syntax + minimal example*
4. *How it works under the hood*
5. *Common patterns and idioms*
6. *Gotchas and pitfalls*
7. *Performance notes*
8. *When to use / when to avoid*

---

## 🎯 Conventions

- All examples target **.NET 10 / C# 14**.
- ASP.NET Core code uses **Minimal APIs** for brevity unless MVC is the point.
- EF Core examples use **EF Core 10**.
- Containers use **Linux x64** unless Windows is specifically the point.
- "Production-quality" examples include error handling, configuration, observability.

---

## ✍️ Relationship to CSharpBook

| | CSharpBook | DotNetBook |
|---|---|---|
| Focus | C# language | .NET platform |
| Audience | Anyone writing C# | Anyone building apps |
| Prerequisites | None | Comfortable with C# basics (CSharpBook chapters 0-5) |
| Code style | Language constructs front and center | Framework constructs front and center |
| When to use | Reference for language features | Reference for platform/library APIs |

The two books cross-link where natural — e.g., DotNetBook's runtime chapter links to CSharpBook's memory & performance chapter for the language-side complement.

Read CSharpBook first (or in parallel) if you're new to C#. Read DotNetBook to build applications.

→ Begin: [Chapter 00 — Overview](00-Overview/README.md)
