# Your First ASP.NET Core App

## The web host and the request pipeline

ASP.NET Core is the framework for HTTP applications. At its core: a **web host** (a Generic Host specialized for HTTP — [Ch03](../03-HostingAndDI/README.md)) running **Kestrel** (a fast cross-platform web server), processing each request through a **middleware pipeline** to your endpoint handler.

```csharp
var builder = WebApplication.CreateBuilder(args);   // a Generic Host for HTTP

// Register services (DI — same container as Ch03)
builder.Services.AddSingleton<IClock, SystemClock>();

var app = builder.Build();                           // WebApplication : IHost

// Build the middleware pipeline (order matters!)
app.UseHttpsRedirection();

// Map an endpoint
app.MapGet("/hello/{name}", (string name, IClock clock) =>
    $"Hello {name} at {clock.UtcNow:O}");

app.Run();                                            // start Kestrel, listen, run until shutdown
```

That's a complete web API. `WebApplication.CreateBuilder` gives you DI, configuration, and logging (everything from Chapter 03) plus the HTTP server and pipeline. `MapGet` defines an endpoint; the lambda's parameters are bound from the request and DI automatically (see [07-ModelBinding.md](07-ModelBinding.md)).

---

## `WebApplication` — host + pipeline in one

`WebApplication.CreateBuilder` (.NET 6+) unified the old `Startup.cs` `ConfigureServices`/`Configure` split into a flat, top-level-statements model:

```csharp
var builder = WebApplication.CreateBuilder(args);
//   builder.Services      → IServiceCollection (DI registration)
//   builder.Configuration → IConfiguration
//   builder.Logging       → logging
//   builder.Environment   → IWebHostEnvironment

var app = builder.Build();
//   app.Use*()  → add middleware (the request pipeline)
//   app.Map*()  → define endpoints (routing)

app.Run();
```

The `builder` phase **registers services**; after `Build()`, the `app` phase **composes the pipeline and endpoints**. This two-phase shape (configure services, then configure the pipeline) is the heart of every ASP.NET Core app.

---

## The request pipeline (middleware)

Every request flows through an ordered chain of **middleware** — components that can inspect/modify the request, short-circuit, or pass control to the next:

```
Request →  [HTTPS redirect] → [Routing] → [Auth] → [Custom] → [Endpoint] → Response
                                                                    ↑
              each middleware can act before AND after the next (it's nested)
```

```csharp
app.UseHttpsRedirection();    // 1. redirect http→https
app.UseAuthentication();       // 2. who are you?
app.UseAuthorization();        // 3. are you allowed?
app.MapGet("/", () => "hi");   // 4. the endpoint (terminal)
```

**Order is critical** — auth must come before the endpoint, routing before auth, etc. Middleware is covered in depth in [05-Middleware.md](05-Middleware.md); routing in [06-Routing.md](06-Routing.md). The pipeline is a **decorator chain** over the request ([Ch03 §06](../03-HostingAndDI/06-Decorate.md)).

---

## Kestrel — the web server

**Kestrel** is the default, cross-platform, high-performance HTTP server (built on `System.IO.Pipelines` and sockets — [Ch02 §13](../02-BCL/13-MemoryPrimitives.md)). It handles HTTP/1.1, HTTP/2, and HTTP/3, TLS, and connection management. You usually don't touch it directly:

```csharp
builder.WebHost.ConfigureKestrel(o => {
    o.Limits.MaxRequestBodySize = 10 * 1024 * 1024;   // 10 MB
    o.ListenAnyIP(5000);
});
```

In production, Kestrel often runs **behind a reverse proxy** (nginx, YARP, or a cloud load balancer / Azure App Service) for TLS termination, load balancing, and protection — but it can also be edge-facing. Either way, your app code is the same. (Deployment: [Ch19](../19-Deployment/README.md).)

---

## Minimal APIs vs MVC vs Razor Pages

ASP.NET Core offers three styles for building endpoints — pick per app shape:

| Style | Best for | This book |
|---|---|---|
| **Minimal APIs** | APIs, microservices, low ceremony | [02-MinimalAPIs.md](02-MinimalAPIs.md) (the default here) |
| **MVC controllers** | larger APIs/apps, conventions, many endpoints | [03-MVC.md](03-MVC.md) |
| **Razor Pages** | server-rendered page-based UI | [04-RazorPages.md](04-RazorPages.md) |

This book uses **Minimal APIs** for examples unless MVC is the point, because they're concise and show the framework clearly. They share the same routing, model binding, filters, and DI — the concepts transfer.

---

## A slightly fuller example

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSingleton<IOrderStore, InMemoryOrderStore>();
builder.Services.AddProblemDetails();          // RFC 7807 error responses ([12])
builder.Services.AddOpenApi();                  // OpenAPI document ([10])

var app = builder.Build();

if (app.Environment.IsDevelopment())
    app.MapOpenApi();                           // expose the OpenAPI doc in dev

app.UseHttpsRedirection();
app.UseStatusCodePages();

var orders = app.MapGroup("/orders");           // a route group ([06])
orders.MapGet("/", (IOrderStore store) => store.All());
orders.MapGet("/{id:int}", (int id, IOrderStore store) =>
    store.Get(id) is { } o ? Results.Ok(o) : Results.NotFound());
orders.MapPost("/", (Order order, IOrderStore store) => {
    store.Add(order);
    return Results.Created($"/orders/{order.Id}", order);
});

app.Run();
```

This shows the shape you'll flesh out across the chapter: grouped routes, DI-injected services, typed results (`Results.Ok/NotFound/Created`), environment checks, and cross-cutting setup (ProblemDetails, OpenAPI).

---

## The lifecycle (recap from Ch03)

```
WebApplication.CreateBuilder  → host with DI, config, logging
   ↓ register services
Build()                        → WebApplication (an IHost)
   ↓ compose pipeline + endpoints
Run()                          → start Kestrel, accept requests until SIGTERM/Ctrl+C → graceful shutdown
```

Because `WebApplication` *is* a host, everything from Chapter 03 applies: DI lifetimes (a **scope per request**!), configuration layering, `IOptions`, `ILogger`, hosted services, graceful shutdown. ASP.NET Core adds the HTTP server, routing, and middleware on top.

---

## Common gotchas

### Middleware order

Putting `UseAuthorization` before `UseAuthentication`, or routing after the endpoints, breaks the pipeline. Order is significant — see [05-Middleware.md](05-Middleware.md).

### Forgetting the per-request scope

Each request runs in its own DI scope, so **Scoped** services (like `DbContext`) are per-request. Injecting them into singletons is a captive-dependency bug ([Ch03 §03](../03-HostingAndDI/03-Lifetimes.md)).

### Treating `WebApplication` as unrelated to the host

It's a Generic Host. Don't relearn DI/config/logging — they're identical to Chapter 03.

### Exposing OpenAPI/dev tooling in production

Guard dev-only middleware (`MapOpenApi`, detailed errors) with `app.Environment.IsDevelopment()`.

### Blocking calls in handlers

Blocking (`.Result`/`.Wait()`) on a request thread starves the thread pool under load ([Ch01 §08](../01-Runtime/08-Threading.md)). Handlers should be `async` and `await`.

---

## Summary

- ASP.NET Core = a **Generic Host specialized for HTTP**: `WebApplication.CreateBuilder` gives DI/config/logging plus **Kestrel** and a **middleware pipeline**.
- Two phases: register **services** on `builder`, then compose the **pipeline (`Use*`) and endpoints (`Map*`)** on `app`; `app.Run()` starts the server.
- Requests flow through an **ordered middleware chain** to an endpoint — **order matters**.
- **Kestrel** is the fast cross-platform server (often behind a reverse proxy in production).
- Three endpoint styles — **Minimal APIs** (default here), **MVC**, **Razor Pages** — share routing/binding/filters/DI.
- Everything from Chapter 03 applies, including a **DI scope per request**.

→ Next: [02-MinimalAPIs.md](02-MinimalAPIs.md)
