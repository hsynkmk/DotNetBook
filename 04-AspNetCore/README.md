# Chapter 04 — ASP.NET Core

> The framework for HTTP-based applications: Minimal APIs, MVC, Razor Pages, middleware, routing, model binding. .NET's flagship web framework.

**Prerequisites**: Chapter 03 (Hosting & DI). Comfortable with HTTP basics.

**Time to read**: ~12-16 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-FirstApp.md](01-FirstApp.md) | `WebApplication.CreateBuilder`, the request pipeline, the very first endpoint. |
| [02-MinimalAPIs.md](02-MinimalAPIs.md) | Route handlers, parameter binding, results, filters (`AddEndpointFilter`), groups, OpenAPI. |
| [03-MVC.md](03-MVC.md) | Controllers, attribute routing, model binding, action filters, when MVC beats Minimal APIs. |
| [04-RazorPages.md](04-RazorPages.md) | Page-based UI: routing, handler methods, model binding for forms. |
| [05-Middleware.md](05-Middleware.md) | The request pipeline: `Use`, `Run`, `Map`, custom middleware classes, ordering. |
| [06-Routing.md](06-Routing.md) | Endpoint routing, attribute vs convention, route constraints, fallback, sub-domain routing. |
| [07-ModelBinding.md](07-ModelBinding.md) | Binding from route/query/header/body/form; custom binders; `IBindAsync`. |
| [08-Validation.md](08-Validation.md) | DataAnnotations, FluentValidation, returning ProblemDetails, validation in Minimal APIs. |
| [09-Filters.md](09-Filters.md) | Authorization, action, result, exception filters; endpoint filters for Minimal APIs. |
| [10-OpenAPI.md](10-OpenAPI.md) | `Microsoft.AspNetCore.OpenApi` (.NET 9+), Swashbuckle, NSwag, documentation patterns. |
| [11-StaticFiles.md](11-StaticFiles.md) | Serving static files, fallback files for SPAs, caching headers. |
| [12-ProblemDetails.md](12-ProblemDetails.md) | RFC 7807, `ProblemDetailsService`, consistent error responses. |
| [13-ContentNegotiation.md](13-ContentNegotiation.md) | JSON / XML / formatters; `[Produces]`, accept headers. |
| [14-RateLimiting.md](14-RateLimiting.md) | `Microsoft.AspNetCore.RateLimiting` — token bucket, sliding window, concurrency. |
| [15-OutputCaching.md](15-OutputCaching.md) | Output caching (response caching's modern replacement, .NET 7+). |
| [16-HealthChecks.md](16-HealthChecks.md) | `AddHealthChecks`, probes, custom checks, integration with K8s. |
| [Questions.md](Questions.md) | ~35 drilling questions. |
| [Coding.md](Coding.md) | Build a Minimal API, add validation, add OpenAPI, add filters. |

---

## Learning objectives

After this chapter you should be able to:
- Build a production-quality HTTP API with Minimal APIs.
- Choose Minimal APIs vs MVC vs Razor Pages for a given app shape.
- Write custom middleware and order it correctly.
- Bind, validate, and respond consistently.
- Document the API and version it sanely.

→ Begin: [01-FirstApp.md](01-FirstApp.md)
