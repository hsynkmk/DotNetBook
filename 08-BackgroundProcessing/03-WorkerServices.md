# Worker Services

## Headless apps with no HTTP

A **Worker Service** is a .NET app whose whole purpose is background processing — no web server, no HTTP endpoints, just hosted services running under the Generic Host. It's the project template for daemons: queue processors, scheduled jobs, message consumers, ETL pipelines, IoT gateways. Same DI/config/logging as a web app ([Ch03](../03-HostingAndDI/README.md)), minus ASP.NET Core.

```bash
dotnet new worker -o MyWorker
```

```csharp
// Program.cs — a worker host (no WebApplication, no Kestrel)
var builder = Host.CreateApplicationBuilder(args);
builder.Services.AddHostedService<QueueWorker>();
builder.Services.AddHostedService<ScheduledCleanup>();
var host = builder.Build();
await host.RunAsync();
```

The template gives you `Host.CreateApplicationBuilder` (not `WebApplication.CreateBuilder`) — the headless Generic Host — plus a sample `BackgroundService`. Everything from Chapter 03 applies (DI, `IOptions`, `ILogger`, lifetime); you just don't have HTTP.

---

## Worker Service vs background service in a web app

You can run `BackgroundService`s **inside** a web app *or* in a dedicated Worker Service. Choosing between them:

| | Background service in the web app | Dedicated Worker Service |
|---|---|---|
| Hosting | shares the web app's process | its own process/deployment |
| Scaling | scales with the web app | scales **independently** |
| Failure isolation | a worker bug can affect the web app | isolated from the API |
| Resource use | shares the API's resources | dedicated resources |
| Deployment | one deployable | separate deployable |

- **In the web app** — simplest for light background work tied to the app (warm a cache, a low-volume queue consumer). One deployable, shared lifecycle.
- **Dedicated Worker Service** — when background work needs to **scale independently** of the API (e.g., 2 API instances but 10 queue processors), have **failure isolation** (a crashing worker shouldn't take down the API), or has different resource needs. Separate deployable, separate scaling.

A common architecture: an API for requests + a Worker Service for heavy/async processing, communicating via a queue or broker ([Ch07](../07-Messaging/README.md)).

---

## Running as a Windows Service or systemd daemon

A Worker Service typically runs as a managed OS service so it starts on boot, restarts on failure, and integrates with the platform's service manager:

```csharp
// Windows Service (Microsoft.Extensions.Hosting.WindowsServices)
builder.Services.AddWindowsService(o => o.ServiceName = "MyWorker");

// systemd (Linux — Microsoft.Extensions.Hosting.Systemd)
builder.Services.AddSystemd();
```

- **`AddWindowsService`** integrates with the Windows Service Control Manager (install with `sc create` or via an installer; logs to the Windows Event Log; respects SCM start/stop).
- **`AddSystemd`** integrates with systemd on Linux (notifies readiness, maps log levels to journald, responds to `systemctl stop`).

These adapters make the host behave correctly under the OS service manager — graceful start/stop signals map to the host lifecycle. (In containers/Kubernetes, you usually *don't* use these — the container *is* the process, and the orchestrator manages lifecycle via signals — [08-ReliabilityAndScale.md](08-ReliabilityAndScale.md).)

---

## Hosting models for workers

| Host | Use |
|---|---|
| **Windows Service** | on-prem Windows servers |
| **systemd daemon** | on-prem/VM Linux servers |
| **Docker container** | cloud/Kubernetes (the common modern target) |
| **Azure Functions / cloud jobs** | serverless/event-driven (different model — [Ch20](../20-AzureIntegration/README.md)) |

For cloud-native deployments, a Worker Service runs as a **container** (no Windows-service/systemd adapter needed — the container runtime/orchestrator manages it). For serverless event-driven work, **Azure Functions** (or similar) is often a better fit than a long-running worker. Choose the host for your environment.

---

## Scoped services in workers (recap)

Like any hosted service, a worker's `BackgroundService` is a **singleton** — create a **scope per unit of work** for scoped services ([02-BackgroundService.md](02-BackgroundService.md), [Ch03 §03](../03-HostingAndDI/03-Lifetimes.md)):

```csharp
public class QueueWorker(IServiceProvider services) : BackgroundService {
    protected override async Task ExecuteAsync(CancellationToken ct) {
        await foreach (var msg in _queue.DequeueAllAsync(ct)) {
            await using var scope = services.CreateAsyncScope();          // scope per message
            var handler = scope.ServiceProvider.GetRequiredService<IMessageHandler>();
            await handler.HandleAsync(msg, ct);
        }
    }
}
```

This is universal across hosted services, web or worker. (Detail in [04-QueuedBackgroundWork.md](04-QueuedBackgroundWork.md).)

---

## Configuration & logging in a worker

Workers use the same configuration and logging stack as web apps:

```csharp
var builder = Host.CreateApplicationBuilder(args);
builder.Services.Configure<WorkerOptions>(builder.Configuration.GetSection("Worker"));   // typed options
// appsettings.json + env vars + command line are loaded by default ([Ch03 §07])
// ILogger<T> is available; route to console/Serilog/OTel as usual ([Ch12])
```

No relearning — `IConfiguration`, `IOptions`, and `ILogger` work identically ([Ch03 §07-09](../03-HostingAndDI/07-Configuration.md)). For containerized workers, configure via environment variables (the standard for containers).

---

## Common gotchas

### Putting heavy background work in the web app when it should be isolated

A heavy/independently-scaling worker inside the API couples their scaling and failure. Split it into a dedicated Worker Service communicating via a queue/broker.

### Using Windows-service/systemd adapters in containers

In a container, the container *is* the process and the orchestrator manages lifecycle via signals — you don't need (and shouldn't add) `AddWindowsService`/`AddSystemd`. Use them only for OS-managed services.

### Forgetting scope-per-work-item

A worker's `BackgroundService` is a singleton; inject scoped services via a scope per item, not the constructor.

### No graceful shutdown handling

A worker that ignores the stopping token / shutdown signal drops in-flight work on restart/deploy. Honor cancellation and drain ([08-ReliabilityAndScale.md](08-ReliabilityAndScale.md)).

### Treating a worker as fundamentally different from a web app

It's the same Generic Host minus HTTP — same DI, config, logging, lifetime. Don't relearn the basics.

---

## Summary

- A **Worker Service** (`dotnet new worker`) is a headless .NET app for background processing — the Generic Host without ASP.NET Core/HTTP; same DI, config, logging, and hosted-service model.
- Run background work **in the web app** for light, app-coupled tasks; use a **dedicated Worker Service** when it must scale independently, needs failure isolation, or has different resource needs (a common API + worker + queue architecture).
- Host it as a **Windows Service** (`AddWindowsService`), **systemd daemon** (`AddSystemd`), or — for cloud — a **container** (no OS-service adapter; the orchestrator manages it); serverless event work may suit **Azure Functions**.
- Workers are singletons → **scope per work item** for scoped services; configuration/logging are identical to web apps.

→ Next: [04-QueuedBackgroundWork.md](04-QueuedBackgroundWork.md)
