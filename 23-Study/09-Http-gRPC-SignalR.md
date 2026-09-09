# 09 — Networking: HttpClient, gRPC & SignalR

## ⚡ 30-second answer

**Never `new HttpClient()` per request** — each one opens its own connection pool and leaves
sockets in `TIME_WAIT`, exhausting ports under load. But a static singleton has the opposite bug:
it **never picks up DNS changes**. **`IHttpClientFactory`** fixes both by handing you a cheap,
short-lived `HttpClient` over a **pooled, rotated handler** — prefer **typed clients**. Note that
`HttpClient` **does not throw on 4xx/5xx**; call `EnsureSuccessStatusCode()`. Add resilience with
one line — **`AddStandardResilienceHandler`** (retry with backoff + jitter, circuit breaker,
timeouts) — and remember to only retry **idempotent** operations. For service-to-service calls,
**gRPC** (contract-first `.proto`, Protobuf over HTTP/2) beats REST on payload size and typing;
for real-time push to browsers, **SignalR** — which **needs a Redis backplane** the moment you run
more than one instance.

---

## Core mechanics

### `HttpClient`

```csharp
var order = await client.GetFromJsonAsync<Order>($"/orders/{id}", ct);   // System.Net.Http.Json
await client.PostAsJsonAsync("/orders", dto, ct);

// Explicit control
using var req = new HttpRequestMessage(HttpMethod.Get, "/big");
using var res = await client.SendAsync(req, HttpCompletionOption.ResponseHeadersRead, ct);
res.EnsureSuccessStatusCode();                       // ← 4xx/5xx do NOT throw on their own
await using var stream = await res.Content.ReadAsStreamAsync(ct);   // stream large responses
```

`SocketsHttpHandler` underneath does pooling, HTTP/2 and HTTP/3, redirects and decompression.
Control timeouts with a per-request **`CancellationToken`** (linked to a timeout CTS) rather than
`HttpClient.Timeout`, which applies to every call on that client.

### `IHttpClientFactory` — the lifetime answer

| Approach | Socket exhaustion | Stale DNS |
|---|---|---|
| `new HttpClient()` per request | ❌ **exhausts sockets** | fine |
| `static HttpClient` singleton | fine | ❌ **never sees DNS changes** |
| **`IHttpClientFactory`** | ✅ pooled | ✅ **handler rotated** (2 min default) |

```csharp
builder.Services.AddHttpClient<CatalogClient>(c => c.BaseAddress = new("https://catalog/"))
    .AddHttpMessageHandler<AuthHandler>()      // outermost first
    .AddStandardResilienceHandler();           // retry + breaker + timeouts
```

`CreateClient()` returns a **cheap, short-lived** `HttpClient` wrapping a pooled handler — create
freely, **don't cache or dispose it**. **Typed clients** (`AddHttpClient<T>`) are the cleanest and
most testable style.

### `DelegatingHandler` — middleware for outbound HTTP

```csharp
public class AuthHandler(ITokenService tokens) : DelegatingHandler
{
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage req, CancellationToken ct)
    {
        req.Headers.Authorization = new("Bearer", await tokens.GetAsync(ct));
        return await base.SendAsync(req, ct);       // → the next handler
    }
}
```

Registered in order — **first registered is outermost** — and the chain ends in the primary
`SocketsHttpHandler`. Use them for **auth**, **correlation/tracing headers**, logging and metrics.
For resilience, use the **standard resilience handler** rather than hand-rolling one.

**The gotcha**: handlers are **pooled and long-lived**, not per-request. A scoped service captured
in a handler field is a captive dependency — reach per-request state through
`IHttpContextAccessor` instead.

### Outbound authentication

Service-to-service uses the OAuth **client-credentials** flow (client id + secret → access token).
**Cache the token and refresh before expiry** — never fetch one per request. Use a library (MSAL,
`Microsoft.Identity.Web`) or, better, **managed identity** (`DefaultAzureCredential`) so there's
no secret at all. Handle expiry with refresh-and-retry-once on a 401 — cloning the request,
because an `HttpRequestMessage` is **single-use**.

### gRPC

Contract-first: write a `.proto`, generate typed client and server. **Protobuf** (compact binary)
over **HTTP/2** (multiplexed, streaming).

```proto
service Catalog {
  rpc GetProduct (ProductRequest) returns (Product);
  rpc Watch (WatchRequest) returns (stream PriceUpdate);   // server streaming
}
```

Four call types: **unary**, **server-streaming**, **client-streaming**, **bidirectional**. Signal
errors with **`RpcException`** and a status code. First-class **deadlines** (propagated to the
server, which sees cancellation), **metadata** (headers, auth), and **interceptors** for
cross-cutting concerns. Register clients with `AddGrpcClient`, which integrates with
`IHttpClientFactory`.

Needs **end-to-end HTTP/2** — a proxy that downgrades to HTTP/1.1 breaks it. Browsers need
**gRPC-Web**.

### SignalR

Real-time server→client push via **hubs**.

```csharp
public class NotificationHub : Hub<INotificationClient>   // strongly-typed, not magic strings
{
    public Task Subscribe(string topic) => Groups.AddToGroupAsync(Context.ConnectionId, topic);
}
await hub.Clients.Group(topic).PriceChanged(update);
```

Targeting: `All`, `Caller`, `Others`, `Group`, `User`, `Client`. It auto-negotiates **WebSockets**
with SSE and long-polling fallbacks.

**Scaling out requires a backplane** (Redis) or **Azure SignalR Service** — without one, a message
published on instance A never reaches a client connected to instance B. This is the number-one
SignalR production bug. You also need **sticky sessions** for the fallback transports.

Clients should **auto-reconnect and re-establish state** — group membership is per-connection and
is lost on reconnect. Keep hub methods quick; offload long work.

### WebSockets and Pipelines

Raw **WebSockets** are full-duplex and long-lived — the transport beneath SignalR. You handle the
receive/send loop, multi-frame messages (`EndOfMessage`), and the close handshake yourself. Prefer
SignalR unless you need a specific sub-protocol, because reconnection, fallback, targeting and
scale-out are exactly what it adds.

**`System.IO.Pipelines`** is the high-throughput parsing engine inside Kestrel, SignalR and gRPC.
Reach for it only for measured, custom protocol work: it solves partial reads, buffer pooling,
multi-segment buffers and back-pressure — the painful parts of raw socket parsing.

Raw **TCP** is a **byte stream, not messages** — you must implement **framing** (length prefix or
delimiter). Treating one `Read` as one message is the classic socket bug.

---

## Comparison tables

| | REST/JSON | gRPC | SignalR | Raw WebSocket |
|---|---|---|---|---|
| Contract | OpenAPI (after the fact) | **`.proto`, contract-first** | hub methods | none |
| Payload | JSON text | **Protobuf binary** | JSON/MessagePack | yours |
| Transport | HTTP/1.1+ | **HTTP/2 required** | WS + fallbacks | WS |
| Browser | native | needs gRPC-Web | ✅ | ✅ |
| Streaming | limited | **bidirectional** | server push | full duplex |
| Use for | public/browser APIs | **internal service-to-service** | live UI | custom sub-protocols |

| Resilience strategy | Purpose | Caveat |
|---|---|---|
| **Retry** + exponential backoff **+ jitter** | transient faults | **idempotent operations only** |
| **Circuit breaker** | fail fast, let a sick service recover | tune the sampling window |
| **Timeout** (per-attempt + total) | bound resource use | total must exceed attempts × per-attempt |
| **Fallback** | graceful degradation | only where a degraded answer is acceptable |

---

## 🪤 Traps & gotchas

- **`new HttpClient()` per request** — disposal doesn't release the socket immediately; it sits in
  `TIME_WAIT`, and under load you exhaust ephemeral ports. The error looks like a random network
  failure, not a leak.
- **`static HttpClient` forever** — solves sockets, creates stale DNS. After a failover the client
  keeps calling the old IP until the process restarts. `IHttpClientFactory` rotates the handler.
- **Caching or disposing the client from `CreateClient()`** — it's meant to be short-lived and
  cheap; the *handler* is what's pooled.
- **`HttpClient` doesn't throw on 404 or 500.** Code that only catches exceptions treats an error
  response as success and deserializes the error body into your DTO as nulls.
- **Reusing an `HttpRequestMessage`** — it's single-use. Retrying with the same instance throws;
  clone it.
- **A scoped service captured in a `DelegatingHandler` field** — handlers are pooled and
  long-lived, so it's a captive dependency ([03](03-Hosting-DI-Config.md)).
- **Retrying non-idempotent requests** — a POST that times out may well have succeeded. Retrying
  creates the order twice. Use an idempotency key, or don't retry.
- **Retry without jitter** — every client retries at the same moment after an outage and
  synchronizes into a thundering herd on a service that's just coming back.
- **Retry inside retry** — a handler retrying 3× wrapped in application code retrying 3× is 9
  requests, and the multiplication is invisible in either place ([11](11-Resilience.md)).
- **`HttpClient.Timeout` used as a per-request timeout** — it applies to every call on that client
  and can't vary. Use a linked `CancellationTokenSource`.
- **Not streaming a large response** — the default buffers the whole body into memory before you
  see a byte. `ResponseHeadersRead` + `ReadAsStreamAsync`.
- **Fetching an OAuth token per request** — an extra round-trip on every call, and you'll get rate
  limited by the IdP. Cache it and refresh before expiry.
- **gRPC through a proxy that downgrades to HTTP/1.1** — it simply doesn't work, and the error
  rarely says why.
- **Ignoring the gRPC deadline server-side** — the deadline propagates as cancellation; not
  honoring it means you keep working for a client that already gave up.
- **Breaking a `.proto` contract** — renumbering or reusing a field tag silently misinterprets
  data. Only add fields with new numbers; never reuse a retired one.
- **SignalR without a backplane behind a load balancer** — messages reach only the clients
  connected to the instance that published them. Users report "notifications work sometimes."
- **Not re-joining groups after reconnect** — group membership is tied to the connection id and is
  gone after a reconnect, so the client silently stops receiving.
- **Long-running work inside a hub method** — it blocks that connection's message processing.
- **Treating a TCP `Read` as a message boundary** — TCP is a byte stream. Without length-prefix or
  delimiter framing, you'll get half a message, or two at once, only under load.
- **Reaching for raw sockets** when HTTP, gRPC or SignalR would do — you're re-implementing
  framing, serialization, auth and resilience by hand.

---

## ❓ Likely questions

**Q: Why shouldn't you `new HttpClient()` per request?**
A: Disposing it doesn't immediately free the underlying socket — it lingers in `TIME_WAIT`. Under
load you exhaust ephemeral ports and calls start failing with what looks like a network error. It
was designed to be reused.

**Q: Then why not just use a static singleton?**
A: Because the handler caches DNS resolution for the life of the connection pool, so after a DNS
change or a failover the client keeps calling the old address until the process restarts.

**Q: How does `IHttpClientFactory` solve both?**
A: It pools handlers and **rotates** them on a timer (two minutes by default), so connections are
reused — no socket exhaustion — but no handler lives long enough to pin stale DNS. `CreateClient`
gives you a cheap short-lived `HttpClient` over the pooled handler.

**Q: Named or typed clients?**
A: Typed (`AddHttpClient<CatalogClient>`) — configuration is bound to a strongly-typed service, the
call sites are testable, and there are no magic strings. Named clients are the fallback when you
need the same class to talk to several endpoints.

**Q: Does `HttpClient` throw on a 500?**
A: No. It only throws for transport-level failures — DNS, connection, timeout. You must call
`EnsureSuccessStatusCode()` or inspect `StatusCode` yourself, which is a very common bug.

**Q: What is a `DelegatingHandler` and what do you use it for?**
A: Middleware for outbound HTTP — it wraps the next handler, running code before and after
`base.SendAsync`. Use it for attaching auth tokens, propagating correlation and trace headers, and
logging. Register in order; first registered is outermost.

**Q: What does `AddStandardResilienceHandler` give you?**
A: A Polly-based pipeline with sensible defaults in one line: rate limiting, total timeout, retry
with exponential backoff and jitter, circuit breaker, and a per-attempt timeout.

**Q: Which requests are safe to retry?**
A: Idempotent ones — GET, PUT, DELETE. A POST that timed out may already have been processed, so
retrying it duplicates the effect. If you need to retry writes, use an idempotency key the server
deduplicates on.

**Q: Why does retry need jitter?**
A: Without it, every client that failed during an outage retries at exactly the same moment,
producing a synchronized thundering herd that knocks over the recovering service. Jitter spreads
them out.

**Q: When would you choose gRPC over REST?**
A: Internal service-to-service calls where you want a strongly-typed contract, compact binary
payloads, HTTP/2 multiplexing, and streaming — especially across languages. REST/JSON stays the
right answer for public and browser-facing APIs, where readability and native browser support
matter more.

**Q: What are gRPC's four call types?**
A: Unary (one request, one response), server-streaming, client-streaming, and bidirectional
streaming. Plus deadlines, which propagate to the server as cancellation, and metadata for
headers and auth.

**Q: What is SignalR and what's the main scaling gotcha?**
A: A real-time messaging abstraction over WebSockets with fallbacks, giving you hubs, group and
user targeting, and reconnection. The gotcha is that with more than one instance you **must** add
a backplane (Redis) or use Azure SignalR Service — otherwise a message published on one instance
never reaches clients connected to another.

**Q: SignalR or raw WebSockets?**
A: SignalR, unless you need a specific sub-protocol or minimal dependencies. It gives you
reconnection, transport fallback, targeting and scale-out — all of which you'd otherwise build
yourself on raw WebSockets.

**Q: What's the classic bug in raw TCP socket code?**
A: Treating a `Read` as returning one complete message. TCP is a byte stream with no message
boundaries, so you must frame — length prefix or delimiter — and loop until you have a whole
message. It works in testing and fails under load.

**Q: What is `System.IO.Pipelines` for?**
A: High-throughput protocol parsing — it handles buffer pooling, partial reads, multi-segment
buffers and back-pressure. It's the engine inside Kestrel, gRPC and SignalR, and you only use it
directly for measured custom-protocol work.

---

## 🎓 Senior Extra

- **`PooledConnectionLifetime` is the knob** behind handler rotation. If you're using a static
  `HttpClient` deliberately (a console tool, say), setting it gives you DNS refresh without the
  factory.
- **HTTP/2 multiplexing changes the failure mode**: many requests share one TCP connection, so one
  stalled connection affects everything on it. `EnableMultipleHttp2Connections` exists for exactly
  that.
- **The ordering of resilience strategies matters**: total timeout outermost, then retry, then
  circuit breaker, then per-attempt timeout. Put the breaker outside the retry and one logical
  call trips it; put it inside and retries are counted as separate failures
  ([11](11-Resilience.md)).
- **A circuit breaker per host, not per client** — otherwise one bad downstream trips calls to a
  healthy one.
- **gRPC deadlines are a distributed-systems feature, not a timeout.** They propagate through the
  call chain, so a 2-second budget at the edge means every downstream hop knows how much time
  remains — which is far better than each service guessing its own timeout.
- **Protobuf field numbers are the contract**, not the names. Adding fields is safe, renaming is
  free, and reusing a retired number silently corrupts data — which is why you mark them
  `reserved`.
- **gRPC-Web loses bidirectional streaming** — browsers can't do it over the wire format. Plan the
  API shape around that if a browser is ever a client.
- **Azure SignalR Service moves connection handling out of your app**, which changes your scaling
  math entirely: your instances stop being connection-bound and you no longer need sticky sessions
  or a backplane ([20](20-Azure.md)).
- **Correlation headers belong in a `DelegatingHandler`**, not at call sites — it's the only way
  to guarantee every outbound request carries them, and it's how the trace survives the hop
  ([12](12-Observability.md)).
- **Socket exhaustion diagnoses as latency, not as an error.** `netstat` showing thousands of
  `TIME_WAIT` entries against one destination is the tell, and it's worth naming in an interview
  as how you'd confirm it.

→ Deeper: [`../09-NetworkingAndHttp/`](../09-NetworkingAndHttp/README.md)
