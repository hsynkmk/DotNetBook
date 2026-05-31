# Chapter 09 — Networking & HTTP — Q & A

---

### Q1. Does `HttpClient` throw on a 404 or 500?

No — it returns the `HttpResponseMessage` regardless of status. Call `EnsureSuccessStatusCode()` to throw on non-2xx (when any failure is exceptional), or check `StatusCode` for expected non-success (404 = not found, 409 = conflict) and handle it as control flow.

---

### Q2. What's the #1 `HttpClient` mistake?

Creating `new HttpClient()` per request — it leaks sockets (`TIME_WAIT` buildup), exhausting the ephemeral port range under load. Use `IHttpClientFactory` (or a properly-managed shared instance).

---

### Q3. Why does a single static `HttpClient` have a problem too?

It pools connections forever and never rotates them, so it doesn't pick up **DNS changes** (a problem when service IPs move in cloud environments). `IHttpClientFactory` fixes both socket exhaustion *and* stale DNS by pooling and rotating the underlying handler.

---

### Q4. How does `IHttpClientFactory` separate lifetimes?

`CreateClient()` returns a **cheap, short-lived `HttpClient`** wrapping a **pooled, expensive `HttpMessageHandler`** that's reused across clients and **rotated** (~2 min) to honor DNS. You call `CreateClient` freely (don't cache/dispose the client); the factory manages the handler.

---

### Q5. Named vs typed clients?

**Named** clients (`AddHttpClient("name", ...)`) centralize per-endpoint config (base URL, headers), retrieved by name. **Typed** clients (`AddHttpClient<T>`) bind config to a strongly-typed service class — the cleanest, most testable style. Prefer typed clients.

---

### Q6. What is a `DelegatingHandler`?

Middleware for outbound HTTP — it wraps the next handler, acting before/after `base.SendAsync` (or short-circuiting). The pipeline ends in the primary `SocketsHttpHandler`. Used for auth, correlation/tracing headers, logging, and metrics; registered via the factory in order (first = outermost).

---

### Q7. What's the lifetime gotcha with delegating handlers?

Handlers are **pooled and rotated** (long-lived, not per-request). Injecting a scoped service into a handler field captures a stale scope. Access per-request data via `IHttpContextAccessor` (ambient, `AsyncLocal`-based), not a captured scoped field.

---

### Q8. How do you add resilience to an HttpClient?

`AddStandardResilienceHandler()` (.NET 8+) — a Polly-based handler adding retry (exponential backoff + jitter), circuit breaker, and timeouts with sensible defaults, in one line. Customize the options; build a custom Polly pipeline for special cases.

---

### Q9. Why must retries use backoff and jitter?

**Backoff** (increasing delays) avoids hammering a struggling service; **jitter** (randomized delays) prevents many clients from retrying in lockstep (a retry storm/thundering herd). The standard resilience handler does both by default.

---

### Q10. What's the idempotency caveat for HTTP retries?

Retrying a **non-idempotent** request (e.g., a POST that creates a resource) can **duplicate** it if the first attempt succeeded but the response was lost. Retry only idempotent operations, or use an idempotency key the server dedupes on.

---

### Q11. What does a circuit breaker do?

When a downstream's failure rate exceeds a threshold, it "opens" — failing subsequent calls fast (no network call) for a cooldown — then half-opens to test recovery. It protects the failing service from a flood of retries and makes the caller fail fast instead of waiting through timeouts, preventing cascading failures.

---

### Q12. Why always set timeouts on outbound calls?

An unbounded call ties up a thread/connection indefinitely when a dependency hangs → resource exhaustion under failure. Set per-attempt and total timeouts (the standard resilience handler provides both).

---

### Q13. What is the OAuth client-credentials flow for?

Service-to-service auth (no user): your service authenticates with its client id + secret to a token endpoint, gets an access token (valid ~1 hour), and sends it as a bearer token. **Cache the token** and refresh before expiry — don't fetch one per request.

---

### Q14. How should outbound auth be applied, and where do secrets go?

Centralize token attachment in a `DelegatingHandler` so every request carries it (and refresh-and-retry-once on 401). Secrets (client secrets/API keys) go in User Secrets (dev) / env vars or a vault (prod) — never in source; or use **managed identity** (`DefaultAzureCredential`) for Azure-to-Azure (no secret at all). Use MSAL/IdentityModel rather than hand-rolling OAuth.

---

### Q15. Why is TCP described as a "byte stream, not messages"?

TCP delivers an ordered stream of bytes with no message boundaries — a single read may return part of a message, a whole message, or several. You must implement **framing** (length-prefix or delimiter) and use `ReadExactlyAsync` to recover message boundaries. Assuming one read = one message is the #1 raw-socket bug.

---

### Q16. TCP vs UDP?

**TCP** — reliable, ordered, connection-oriented byte stream (needs framing). **UDP** — fast, connectionless datagrams with preserved message boundaries but no delivery/order guarantee. Use UDP for loss-tolerant real-time data (game state, telemetry); for reliable + low-latency prefer QUIC/HTTP3 over hand-rolled UDP reliability.

---

### Q17. What is gRPC and when do you choose it?

Contract-first RPC: define a `.proto` → generate typed client/server; Protobuf (compact binary) over HTTP/2 (multiplexed, streaming). Choose it for **service-to-service** communication (small payloads, fast, polyglot contracts, streaming); choose REST/JSON for public/browser APIs.

---

### Q18. What are gRPC's four call types?

**Unary** (request/response), **server streaming** (one request, many responses), **client streaming** (many requests, one response), and **bidirectional** (many ↔ many, full-duplex over HTTP/2). Streaming over a multiplexed HTTP/2 connection is a key gRPC advantage.

---

### Q19. How do gRPC deadlines work?

A client sets a **deadline** (absolute completion time); it **propagates to the server**, which sees it via `context.Deadline`/`CancellationToken`. The server should honor it (forward the token) to abandon work the client no longer awaits. Errors are signaled with `RpcException` + a status code.

---

### Q20. What does SignalR provide over raw WebSockets?

A high-level **hub** model with server-to-client **push**, **targeting** (all/caller/group/user/connection), automatic **transport fallback** (WebSockets → SSE → long polling), client **auto-reconnect**, and **scale-out** via a backplane — all of which you'd build by hand on raw WebSockets.

---

### Q21. Why does scaled-out SignalR need a backplane?

SignalR connections are stateful and tied to a specific instance — a message sent on instance A won't reach a client connected to instance B without coordination. A **Redis backplane** (or **Azure SignalR Service**) broadcasts messages across instances. Without it, multi-instance apps drop cross-instance messages (the #1 SignalR scaling bug). Also needs sticky sessions.

---

### Q22. What must a SignalR client do on reconnect?

A reconnect is a **new connection** (new connection id) — group memberships and per-connection state are lost. The client must **re-establish state** (re-join groups, re-subscribe) in `onreconnected`. Use `withAutomaticReconnect` for the reconnection itself.

---

### Q23. When use raw WebSockets instead of SignalR?

When you must implement a **specific WebSocket sub-protocol**, want minimal dependencies/full frame-level control, or talk to a non-SignalR WebSocket server. For most real-time app features, SignalR is the default (it adds reconnection, fallback, targeting, scale-out). Raw WebSockets make scale-out and reconnection your problem.

---

### Q24. How do you handle a large WebSocket message?

A large message may arrive in **multiple frames** — loop `ReceiveAsync` until `result.EndOfMessage` is true to assemble the complete message. Handle the `Close` message type with the close handshake (`CloseAsync`), and use keep-alive pings to detect dead connections.

---

### Q25. When should you reach for System.IO.Pipelines in networking?

For **high-throughput custom protocol parsers** over sockets — it solves partial reads (TCP framing), buffer pooling, leftover-copying, and back-pressure (it's what Kestrel uses). Don't use it for ordinary HTTP/gRPC/SignalR (which use it internally) or simple/low-volume socket code. Full mechanics: CSharpBook Ch13 §03.

---

→ Next: [Coding.md](Coding.md)
