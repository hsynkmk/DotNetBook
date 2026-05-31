# Delegating Handlers

## Middleware for outbound HTTP

A `DelegatingHandler` is a component in the **outbound HTTP pipeline** — the client-side equivalent of ASP.NET Core middleware ([Ch04 §05](../04-AspNetCore/05-Middleware.md)). Each handler wraps the next, so it can inspect/modify the request on the way out and the response on the way back. This is where cross-cutting concerns for outbound calls live: authentication, logging, metrics, retry, correlation headers.

```csharp
public class LoggingHandler(ILogger<LoggingHandler> log) : DelegatingHandler {
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct) {
        log.LogInformation("→ {Method} {Uri}", request.Method, request.RequestUri);
        var sw = Stopwatch.StartNew();
        var response = await base.SendAsync(request, ct);    // call the next handler (→ eventually the network)
        log.LogInformation("← {Status} in {Ms}ms", (int)response.StatusCode, sw.ElapsedMilliseconds);
        return response;
    }
}

// Register on a typed/named client
builder.Services.AddTransient<LoggingHandler>();
builder.Services.AddHttpClient<ApiClient>().AddHttpMessageHandler<LoggingHandler>();
```

`base.SendAsync` continues the pipeline; not calling it short-circuits (return a synthetic response). Handlers nest like middleware — the last added is outermost.

---

## The handler pipeline

```
HttpClient.SendAsync
   → DelegatingHandler 1 (e.g., logging)
      → DelegatingHandler 2 (e.g., auth)
         → DelegatingHandler 3 (e.g., retry)
            → SocketsHttpHandler (the primary handler — actually sends over the network)
         ← response flows back out through each handler (in reverse)
```

The pipeline ends in a **primary handler** (`SocketsHttpHandler`) that performs the actual network I/O. Delegating handlers wrap it, each running before and after the inner handlers — exactly the decorator/middleware pattern ([Ch03 §06](../03-HostingAndDI/06-Decorate.md)). Order matters: register them in the order they should wrap (outermost first).

```csharp
builder.Services.AddHttpClient<ApiClient>()
    .AddHttpMessageHandler<LoggingHandler>()      // outermost — logs everything incl. retries
    .AddHttpMessageHandler<AuthHandler>()
    .AddHttpMessageHandler<RetryHandler>();        // innermost (closest to the network)
```

---

## Common handlers

### Authentication (attach a token to every request)

```csharp
public class AuthHandler(ITokenProvider tokens) : DelegatingHandler {
    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken ct) {
        var token = await tokens.GetAccessTokenAsync(ct);                       // fetch/refresh
        request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);
        return await base.SendAsync(request, ct);
    }
}
```

An auth handler centralizes token attachment (and refresh-on-401 — [05-Authentication.md](05-Authentication.md)) so every outbound request to an API carries credentials, without per-call code.

### Correlation / tracing headers

```csharp
public class CorrelationHandler(IHttpContextAccessor accessor) : DelegatingHandler {
    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken ct) {
        var correlationId = accessor.HttpContext?.TraceIdentifier;
        if (correlationId is not null) request.Headers.Add("X-Correlation-Id", correlationId);
        return base.SendAsync(request, ct);   // propagate the correlation/trace id downstream
    }
}
```

Propagating a correlation/trace id to downstream services ties distributed traces together ([Ch12 Observability](../12-Observability/README.md)). (The W3C `traceparent` header is propagated automatically when tracing is enabled — this is for custom correlation.)

### Metrics / instrumentation

```csharp
public class MetricsHandler(Meter meter) : DelegatingHandler {
    private readonly Histogram<double> _duration = meter.CreateHistogram<double>("http.client.duration", "ms");
    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken ct) {
        var sw = Stopwatch.StartNew();
        var response = await base.SendAsync(request, ct);
        _duration.Record(sw.Elapsed.TotalMilliseconds,
            new("host", request.RequestUri?.Host), new("status", (int)response.StatusCode));
        return response;
    }
}
```

Record per-call metrics (duration, status, host) for outbound HTTP ([Ch02 §08](../02-BCL/08-Diagnostics.md)). (Much of this is built into the factory's telemetry, but custom handlers add domain-specific metrics.)

---

## Handler lifetime and scope (the subtle part)

Delegating handlers registered with `AddHttpMessageHandler` follow the **handler's lifetime**, not the request's. Because the factory **pools and rotates handlers** ([02-IHttpClientFactory.md](02-IHttpClientFactory.md)), a handler instance is reused across many requests for the duration of the handler's lifetime (~2 min default). This means:

- A handler that needs **per-request scoped state** (like the current user from `IHttpContextAccessor`) must access it carefully — the handler instance outlives a single request. `IHttpContextAccessor` works (it reads the *current* ambient context via `AsyncLocal`), but **injecting a scoped service directly into a handler can capture a stale scope**.
- For scoped dependencies, the factory supports DI integration so handlers resolve correctly, but be aware of the lifetime mismatch (similar to middleware — [Ch04 §05](../04-AspNetCore/05-Middleware.md)). When in doubt, resolve per-request state via `IHttpContextAccessor` or a scope-aware mechanism, not a captured scoped field.

This lifetime nuance is the main gotcha with handlers — they're long-lived (pooled), so don't assume per-request lifetime.

---

## Handlers vs other approaches

| Concern | DelegatingHandler | Alternative |
|---|---|---|
| Auth on every outbound call | ✓ (centralized) | per-call header (repetitive) |
| Retry/circuit breaker | possible, but... | **`AddStandardResilienceHandler`** (Polly — [04](04-Resilience.md)) — prefer this |
| Logging/metrics | ✓ | built-in factory telemetry + custom handler |
| Correlation propagation | ✓ | automatic `traceparent` (for W3C) |

For **resilience** specifically (retry, circuit breaker, timeout), don't hand-roll a `DelegatingHandler` — use the standard resilience handler ([04-Resilience.md](04-Resilience.md)), which is a well-tested Polly-based handler. Use custom delegating handlers for **auth, correlation, domain-specific logging/metrics, and request/response transformation**.

---

## Common gotchas

### Wrong order

Handlers wrap in registration order (first = outermost). A logging handler outermost logs retries; innermost logs only the final attempt. Order deliberately based on what each should see.

### Forgetting to call `base.SendAsync`

A handler that doesn't call `base.SendAsync` (and isn't deliberately short-circuiting) breaks the pipeline — the request never reaches the network. Always call it unless returning a synthetic response.

### Capturing scoped state in a pooled handler

Handlers are pooled/rotated (long-lived), not per-request. Injecting a scoped service into a handler field captures a stale scope. Use `IHttpContextAccessor` (ambient, `AsyncLocal`-based) for per-request data.

### Hand-rolling retry as a handler

A naive retry handler can mis-handle non-idempotent requests, response disposal, and backoff. Use `AddStandardResilienceHandler` ([04](04-Resilience.md)) instead.

### Not disposing/handling the response on short-circuit

If a handler short-circuits (returns its own response), ensure the response is well-formed; if it reads/buffers the inner response, dispose appropriately.

---

## Summary

- A **`DelegatingHandler`** is **middleware for outbound HTTP** — it wraps the next handler, acting before/after `base.SendAsync` (or short-circuiting); the pipeline ends in the primary `SocketsHttpHandler` that does the network I/O.
- Register via the factory (`AddHttpMessageHandler`) in **order** (first = outermost); used for **auth** (attach/refresh tokens), **correlation/tracing** headers, **logging/metrics**, and request/response transformation.
- Handlers are **pooled and rotated** (long-lived, not per-request) — access per-request state via `IHttpContextAccessor`, not a captured scoped field (the main gotcha).
- For **resilience** (retry/circuit breaker/timeout), prefer the **standard resilience handler** ([04](04-Resilience.md)) over a hand-rolled handler; use custom handlers for auth/correlation/domain concerns.

→ Next: [04-Resilience.md](04-Resilience.md)
