# HttpClient

## The client for HTTP

`HttpClient` is the .NET type for making HTTP requests — calling REST APIs, downloading files, talking to other services. It's a high-level wrapper over `SocketsHttpHandler` (the managed HTTP engine — [Ch02 §10](../02-BCL/10-Net.md)) that handles connection pooling, HTTP/2 and HTTP/3, redirects, compression, and more.

```csharp
// JSON helpers (System.Net.Http.Json) — the everyday case
var product = await client.GetFromJsonAsync<Product>($"/products/{id}", ct);
await client.PostAsJsonAsync("/products", newProduct, ct);

// Full control
var response = await client.GetAsync("/products", ct);
response.EnsureSuccessStatusCode();
var products = await response.Content.ReadFromJsonAsync<List<Product>>(ct);
```

> The **#1 lifetime rule** (don't `new` one per request) and the modern way to obtain clients are in [02-IHttpClientFactory.md](02-IHttpClientFactory.md). This file covers the request/response model itself.

---

## Request and response model

```csharp
// Build a request explicitly when you need headers/content control
using var request = new HttpRequestMessage(HttpMethod.Post, "/orders") {
    Content = JsonContent.Create(order),
    Headers = { { "X-Idempotency-Key", key } }
};
using HttpResponseMessage response = await client.SendAsync(request, ct);

response.StatusCode;                       // HttpStatusCode (200, 404, ...)
response.IsSuccessStatusCode;              // 2xx?
response.Headers;                          // response headers
var body = await response.Content.ReadAsStringAsync(ct);
```

- **`HttpRequestMessage`** — method, URI, headers, content (body). Disposable (and single-use — you can't resend the same instance).
- **`HttpResponseMessage`** — status, headers, and `Content` (the body stream). Disposable — dispose it (or the helper methods do) to release the connection back to the pool.
- **`HttpContent`** — the body: `JsonContent`, `StringContent`, `StreamContent`, `MultipartFormDataContent` (file uploads), `FormUrlEncodedContent`.

The convenience methods (`GetAsync`/`PostAsJsonAsync`/`GetFromJsonAsync`) build the `HttpRequestMessage` for you — use `SendAsync` with an explicit request only when you need fine control (custom headers per request, specific content types).

---

## `System.Net.Http.Json` — JSON the easy way

```csharp
// GET → deserialize
Product? p = await client.GetFromJsonAsync<Product>("/products/1", ct);

// POST/PUT → serialize body, optionally read the response
HttpResponseMessage r = await client.PostAsJsonAsync("/products", product, ct);
Created? created = await r.Content.ReadFromJsonAsync<Created>(ct);

// Stream large JSON arrays at constant memory
await foreach (var item in client.GetFromJsonAsAsyncEnumerable<Item>("/items", ct))
    Process(item);
```

The JSON extension methods use **System.Text.Json** (configure via `JsonSerializerOptions` — [Ch13 §05](../13-IO/05-SystemTextJson.md)). They handle serialization/deserialization and content types. `GetFromJsonAsync` returns `null`/throws on non-success depending on the overload — check status when it matters. For AOT, pass a source-generated `JsonTypeInfo`.

---

## Status codes & error handling

`HttpClient` does **not** throw on non-success status codes (4xx/5xx) by default — it returns the response. You decide how to react:

```csharp
var response = await client.GetAsync($"/products/{id}", ct);

if (response.StatusCode == HttpStatusCode.NotFound) return null;   // expected — handle it
response.EnsureSuccessStatusCode();                                 // throw HttpRequestException on other non-2xx
var product = await response.Content.ReadFromJsonAsync<Product>(ct);
```

- **`EnsureSuccessStatusCode()`** throws `HttpRequestException` for non-2xx — convenient for "any failure is exceptional."
- For **expected** non-success (404 = not found, 409 = conflict), check `StatusCode` and handle it as control flow rather than catching exceptions ([Ch04 §12](../04-AspNetCore/12-ProblemDetails.md), CSharpBook Ch17 §13).
- `HttpRequestException` carries the status code (`.StatusCode`) in modern .NET.

A transport failure (DNS, connection refused, timeout) throws `HttpRequestException` (or `TaskCanceledException`/`TimeoutException` for timeouts) — distinct from an HTTP error status.

---

## Timeouts & cancellation

```csharp
// Per-client default timeout (applies to the whole request incl. reading the body)
client.Timeout = TimeSpan.FromSeconds(30);

// Per-request cancellation (preferred — forward the caller's token)
var response = await client.GetAsync(url, ct);

// Combine a per-request timeout with the caller's token:
using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
cts.CancelAfter(TimeSpan.FromSeconds(5));
var r = await client.GetAsync(url, cts.Token);
```

`HttpClient.Timeout` is a blunt instrument (whole-request, per-client). For granular control, use a **`CancellationToken`** per request (forward the request's token so a disconnected client cancels the outbound call) and link it with a timeout CTS. A timeout surfaces as `TaskCanceledException`/`TimeoutException`. (Resilience-based timeouts in [04-Resilience.md](04-Resilience.md).)

---

## Streaming responses

For large responses, **stream** rather than buffering the whole body into memory:

```csharp
// Read headers, then stream the body (don't buffer it all)
var response = await client.GetAsync(url, HttpCompletionOption.ResponseHeadersRead, ct);
response.EnsureSuccessStatusCode();
await using var stream = await response.Content.ReadAsStreamAsync(ct);
await stream.CopyToAsync(destinationFile, ct);   // stream to disk, constant memory
```

`HttpCompletionOption.ResponseHeadersRead` returns as soon as headers arrive (default `ResponseContentRead` buffers the whole body first). Combined with `ReadAsStreamAsync`, this streams large downloads/uploads at constant memory — essential for files or large payloads. Pair with `System.IO.Pipelines` ([10-Pipelines.md](10-Pipelines.md)) for high-throughput parsing.

---

## HTTP/2 and HTTP/3

```csharp
var request = new HttpRequestMessage(HttpMethod.Get, url) {
    Version = HttpVersion.Version30,
    VersionPolicy = HttpVersionPolicy.RequestVersionOrLower   // negotiate down if unsupported
};
```

`SocketsHttpHandler` supports HTTP/1.1, **HTTP/2** (multiplexing — many requests over one connection), and **HTTP/3** (QUIC over UDP — lower latency, better on lossy networks; stable in .NET 9+). It negotiates the highest version both ends support. HTTP/2 is the default for capable HTTPS endpoints (and required for gRPC — [07-gRPC.md](07-gRPC.md)). You rarely set the version manually; the handler negotiates.

---

## Common gotchas

### `new HttpClient()` per request

The cardinal HTTP bug — socket exhaustion (`TIME_WAIT` buildup) under load. Reuse a shared instance or use `IHttpClientFactory` ([02-IHttpClientFactory.md](02-IHttpClientFactory.md)). (Covered there in depth.)

### Expecting it to throw on 4xx/5xx

It doesn't — it returns the response. Use `EnsureSuccessStatusCode()` for "failure is exceptional," or check `StatusCode` for expected non-success (404/409).

### Not disposing the response (or its stream)

An undisposed `HttpResponseMessage` (when not using the helper methods) can hold the connection. The JSON/string helpers handle this; with `SendAsync`, dispose the response (`using`).

### Buffering huge responses

Default behavior reads the whole body into memory. For large payloads, use `ResponseHeadersRead` + `ReadAsStreamAsync` to stream.

### `HttpClient.Timeout` for granular control

It's whole-request and per-client. Use per-request `CancellationToken` (+ a linked timeout CTS) for granular, per-call control; forward the caller's token.

### Forgetting to forward the `CancellationToken`

Not passing the request's token means an outbound call continues after the client disconnects (wasted work). Forward it to `GetAsync`/`SendAsync`.

---

## Summary

- **`HttpClient`** makes HTTP requests over `SocketsHttpHandler` (pooling, HTTP/2 & HTTP/3, redirects, compression); use **`System.Net.Http.Json`** helpers (`GetFromJsonAsync`/`PostAsJsonAsync`) for the everyday JSON case.
- The model is **`HttpRequestMessage`** (method/URI/headers/content) → **`HttpResponseMessage`** (status/headers/`Content`); use explicit `SendAsync` for fine control, helpers otherwise.
- It **doesn't throw on 4xx/5xx** — call `EnsureSuccessStatusCode()` for exceptional failures, or check `StatusCode` for expected non-success.
- Control timeouts with a per-request **`CancellationToken`** (+ linked timeout CTS); **stream** large responses with `ResponseHeadersRead` + `ReadAsStreamAsync`.
- The lifetime rule (never `new` per request) and obtaining clients properly are in [02-IHttpClientFactory.md](02-IHttpClientFactory.md).

→ Next: [02-IHttpClientFactory.md](02-IHttpClientFactory.md)
