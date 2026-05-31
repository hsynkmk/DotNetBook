# Ecosystem Map — The Library Families

## Orienting yourself

Beyond the runtime and BCL, .NET has families of libraries for building real applications. This map names the major ones and what each is for, so the rest of the book has context. Most are open source under the .NET Foundation and ship as NuGet packages (some in the SDK, some you add).

```
                         ┌─────────────────────────────┐
                         │   Your application            │
                         └──────────────┬──────────────┘
        ┌───────────────────────────────┼────────────────────────────────┐
   Web / API            Data            Client UI          Cross-cutting
   ─────────            ────            ─────────          ─────────────
   ASP.NET Core         EF Core         Blazor             DI / Config / Logging
   Minimal APIs         Dapper          MAUI               Polly (resilience)
   gRPC / SignalR       ADO.NET         WPF / WinUI         OpenTelemetry
                        Redis cache     WinForms            Aspire (orchestration)
        └───────────────────────────────┼────────────────────────────────┘
                         ┌──────────────┴──────────────┐
                         │   BCL + Runtime (Ch01–02)     │
                         └─────────────────────────────┘
```

---

## Web & services

### ASP.NET Core — [Chapter 04](../04-AspNetCore/README.md)
The flagship framework for HTTP applications: web APIs, server-rendered UI (MVC, Razor Pages), and the host for Blazor. **Minimal APIs** are the modern, low-ceremony way to build endpoints; **MVC** suits larger apps. Includes middleware, routing, model binding, validation, OpenAPI, auth, rate limiting, and health checks.

### gRPC — [Chapter 09](../09-NetworkingAndHttp/README.md)
High-performance, contract-first RPC over HTTP/2 using Protocol Buffers. Ideal for service-to-service communication where speed and strong contracts matter.

### SignalR — [Chapter 09](../09-NetworkingAndHttp/README.md)
Real-time, bidirectional communication (WebSockets with fallbacks). For live dashboards, chat, notifications, collaborative apps.

### HttpClient / IHttpClientFactory — [Chapter 09](../09-NetworkingAndHttp/README.md)
The client side of HTTP. `IHttpClientFactory` manages handler lifetimes correctly (avoiding socket exhaustion) and composes with resilience.

---

## Data & caching

### Entity Framework Core — [Chapter 05](../05-EFCore/README.md)
The flagship ORM: map C# classes to database tables, query with LINQ, track changes, run migrations. Productive and feature-rich; supports SQL Server, PostgreSQL, SQLite, and more.

### Dapper / ADO.NET — [Chapter 06](../06-DataAndCaching/README.md)
Lightweight data access. **ADO.NET** is the low-level database API (connections, commands, readers). **Dapper** is a thin, fast micro-ORM over it — when you want SQL control with minimal mapping overhead.

### Caching — [Chapter 06](../06-DataAndCaching/README.md)
`IMemoryCache` (in-process), `IDistributedCache` (Redis/SQL for multi-instance), **HybridCache** (.NET 9+, combines both), and ASP.NET Core **output caching**.

---

## Background work & messaging

### Background processing — [Chapter 08](../08-BackgroundProcessing/README.md)
`IHostedService`/`BackgroundService` for in-process background work, Worker Services for headless hosts, and durable job systems (Hangfire, Quartz.NET) for scheduling and retries.

### Messaging — [Chapter 07](../07-Messaging/README.md)
Inter-service communication via brokers: `Channel<T>` (in-process), MassTransit (abstraction), RabbitMQ, Azure Service Bus, Kafka. For decoupling, queues, and event-driven systems.

---

## Client UI

### Blazor — [Chapter 14](../14-Blazor/README.md)
Build interactive web UI in C# (instead of JavaScript). Runs **Server-side** (over SignalR), in the browser via **WebAssembly**, or as **Hybrid** (in MAUI/desktop). Component-based.

### .NET MAUI — [Chapter 15](../15-MAUI/README.md)
**Multi-platform App UI** — one codebase for iOS, Android, macOS, and Windows native apps. The evolution of Xamarin.Forms.

### WPF / WinUI 3 / WinForms — [Chapter 16](../16-Desktop/README.md)
Windows desktop UI frameworks. **WinForms** (classic, simple), **WPF** (rich, XAML, mature), **WinUI 3** (modern Windows look). All Windows-only.

---

## Cross-cutting concerns

### Hosting, DI, Configuration — [Chapter 03](../03-HostingAndDI/README.md) & [Chapter 13](../13-Configuration/README.md)
The **Generic Host** wires up dependency injection (`IServiceProvider`), configuration (`IConfiguration`/`IOptions`), logging (`ILogger`), and lifetime — the backbone shared by every modern .NET app, web or not.

### Resilience — [Chapter 11](../11-Resilience/README.md)
**Polly** and **Microsoft.Extensions.Resilience**: retry, circuit breaker, timeout, fallback, hedging — handling transient failures in distributed systems.

### Observability — [Chapter 12](../12-Observability/README.md)
Logging (`ILogger`, Serilog), metrics, and distributed tracing — converging on **OpenTelemetry**, the vendor-neutral industry standard.

### .NET Aspire — [Chapter 18](../18-Aspire/README.md)
Cloud-native **orchestration**: model your multi-service app (API + worker + database + cache) in C#, get service discovery, telemetry, a live dashboard, and consistent deployment. The modern way to compose distributed .NET apps.

---

## Quality & shipping

### Testing — [Chapter 17](../17-Testing/README.md)
xUnit/NUnit/MSTest, integration testing with `WebApplicationFactory`, real dependencies via Testcontainers, Blazor component tests with bUnit. (Language-level testing is in CSharpBook Chapter 16.)

### Deployment — [Chapter 19](../19-Deployment/README.md)
Docker, Kubernetes, Native AOT, ReadyToRun, single-file, and the publish-mode choices.

### Azure Integration — [Chapter 20](../20-AzureIntegration/README.md)
Functions (serverless), App Service, Service Bus, Blob/Queue/Table Storage, Cosmos DB — the most common cloud targets for .NET.

### Performance & Tooling — [Chapter 21](../21-Performance/README.md)
BenchmarkDotNet, profilers, and the `dotnet-counters`/`-trace`/`-dump` diagnostic tools.

---

## Notable libraries you'll meet (not all covered in depth)

| Library | Purpose |
|---|---|
| **MediatR** | In-process messaging / CQRS dispatch ([Ch22](../22-BestPractices/README.md)) |
| **FluentValidation** | Validation rules ([Ch04](../04-AspNetCore/README.md)) |
| **AutoMapper / Mapperly** | Object-to-object mapping (Mapperly is source-gen / AOT-safe) |
| **Serilog** | Structured logging ([Ch12](../12-Observability/README.md)) |
| **xUnit / Moq / FluentAssertions** | Testing (CSharpBook Ch16) |
| **ML.NET / Semantic Kernel** | Machine learning / AI orchestration |
| **Newtonsoft.Json** | JSON (legacy default; STJ is now built-in) |

---

## How to read the rest of this book

- **Foundations** (Ch01–03): runtime, BCL, hosting/DI — read these first; everything builds on them.
- **Building applications** (Ch04–10): pick the chapters matching what you're building (web, data, messaging, background work, HTTP, identity).
- **Cross-cutting** (Ch11–13): resilience, observability, configuration — apply everywhere.
- **Client UI** (Ch14–16): Blazor, MAUI, desktop.
- **Quality & ship** (Ch17–21): testing, Aspire, deployment, Azure, performance.
- **Expert** (Ch22): architecture and patterns at scale.

The README's **learning paths** suggest sequences for common goals (new-to-.NET, senior backend, mobile/desktop, performance, cloud-native).

---

## Summary

- .NET's ecosystem groups into **web/services** (ASP.NET Core, gRPC, SignalR), **data** (EF Core, Dapper, caching), **background/messaging** (hosted services, brokers), **client UI** (Blazor, MAUI, desktop), and **cross-cutting** (DI/config/logging, resilience, observability, Aspire).
- Most are open source NuGet packages under the .NET Foundation; the **Generic Host** (DI + config + logging) is the shared backbone.
- **.NET Aspire** is the modern composition/orchestration layer tying services together.
- Read foundations (Ch01–03) first, then the chapters matching your workload.

→ Next: [Questions.md](Questions.md)
