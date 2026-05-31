# gRPC

## High-performance, contract-first RPC

gRPC is a remote-procedure-call framework: you define a **service contract** in a `.proto` file, and gRPC generates strongly-typed client and server code. It serializes with **Protocol Buffers** (compact binary) over **HTTP/2** (multiplexed, streaming). It's the go-to for efficient, typed **service-to-service** communication where you control both ends — faster and more contract-rigorous than JSON-over-HTTP. The .NET implementation is `grpc-dotnet` (`Grpc.AspNetCore`).

```protobuf
// orders.proto — the contract (the source of truth for client AND server)
syntax = "proto3";
service OrderService {
  rpc GetOrder (GetOrderRequest) returns (Order);
  rpc ListOrders (ListRequest) returns (stream Order);   // server streaming
}
message GetOrderRequest { int32 id = 1; }
message Order { int32 id = 1; string customer = 2; double total = 3; }
```

The build generates C# base classes (server) and clients from the `.proto`. You implement the server method; the client calls it like a local async method.

---

## Why gRPC (vs JSON/HTTP)

| | gRPC | JSON over HTTP (REST) |
|---|---|---|
| Contract | **`.proto`** (strict, codegen) | OpenAPI (looser) |
| Serialization | **Protobuf** (compact binary, fast) | JSON (text, larger) |
| Transport | **HTTP/2** (multiplexed, streaming) | HTTP/1.1 or 2 |
| Streaming | **bidirectional** (built-in) | limited (SSE/chunked) |
| Browser support | limited (needs gRPC-Web) | native |
| Best for | **service-to-service**, low latency, polyglot | public APIs, browsers, simple integration |

gRPC wins for **internal microservice communication**: smaller payloads, faster serialization, strong contracts shared across languages (a `.proto` generates clients for C#, Go, Java, etc.), and first-class streaming. REST/JSON wins for **public/browser-facing** APIs (native browser support, human-readable, ubiquitous tooling). Many systems use both — gRPC between services, REST at the edge.

---

## Server implementation

```csharp
// Program.cs
builder.Services.AddGrpc();
var app = builder.Build();
app.MapGrpcService<OrderServiceImpl>();

// Implement the generated base class
public class OrderServiceImpl(IOrderRepository repo) : OrderService.OrderServiceBase {
    public override async Task<Order> GetOrder(GetOrderRequest request, ServerCallContext context) {
        var order = await repo.GetAsync(request.Id, context.CancellationToken);
        if (order is null)
            throw new RpcException(new Status(StatusCode.NotFound, $"Order {request.Id} not found"));
        return new Order { Id = order.Id, Customer = order.Customer, Total = (double)order.Total };
    }
}
```

You override the generated `OrderServiceBase` methods. `ServerCallContext` gives the cancellation token, deadline, headers (metadata), and peer info. Errors are signaled with **`RpcException`** carrying a gRPC **status code** (`NotFound`, `InvalidArgument`, `PermissionDenied`, etc.) — the gRPC analog of HTTP status codes.

---

## Client usage

```csharp
// Register a typed gRPC client (integrates with IHttpClientFactory — pooling, resilience)
builder.Services.AddGrpcClient<OrderService.OrderServiceClient>(o =>
    o.Address = new Uri("https://orders-service:5001"));

// Inject and call — looks like a local async method
public class OrdersController(OrderService.OrderServiceClient client) {
    public async Task<Order> Get(int id, CancellationToken ct) =>
        await client.GetOrderAsync(new GetOrderRequest { Id = id }, cancellationToken: ct);
}
```

`AddGrpcClient` registers a strongly-typed client that integrates with `IHttpClientFactory` ([02-IHttpClientFactory.md](02-IHttpClientFactory.md)) — so it gets handler pooling, delegating handlers (auth — [05-Authentication.md](05-Authentication.md)), and resilience ([04-Resilience.md](04-Resilience.md)). The generated client method is a normal `async` call that handles serialization and the HTTP/2 transport.

---

## The four call types

gRPC supports four streaming patterns (its big advantage over request/response REST):

```protobuf
rpc GetOrder(Req) returns (Order);                 // 1. Unary — one request, one response (like REST)
rpc ListOrders(Req) returns (stream Order);         // 2. Server streaming — one request, many responses
rpc UploadOrders(stream Order) returns (Summary);   // 3. Client streaming — many requests, one response
rpc Chat(stream Msg) returns (stream Msg);          // 4. Bidirectional — many ↔ many
```

```csharp
// Server streaming — yield results as they're produced
public override async Task ListOrders(ListRequest request, IServerStreamWriter<Order> responseStream, ServerCallContext context) {
    await foreach (var order in repo.StreamAsync(context.CancellationToken))
        await responseStream.WriteAsync(MapToProto(order));   // client receives each as it's written
}

// Client consumes the stream
using var call = client.ListOrders(new ListRequest());
await foreach (var order in call.ResponseStream.ReadAllAsync(ct))
    Process(order);
```

- **Unary** — the common case (request/response).
- **Server streaming** — push many results (live updates, large result sets) over one call.
- **Client streaming** — upload a stream (batch ingestion).
- **Bidirectional** — full-duplex (real-time chat, live sync) — both sides stream concurrently over the multiplexed HTTP/2 connection.

Streaming over HTTP/2 (no new connection per message) is efficient and a major reason to choose gRPC for streaming workloads.

---

## Deadlines, cancellation, and metadata

```csharp
// Client sets a DEADLINE (absolute time by which the call must complete) — propagates to the server
var order = await client.GetOrderAsync(request,
    deadline: DateTime.UtcNow.AddSeconds(5), cancellationToken: ct);

// Pass metadata (headers), e.g., auth
var headers = new Metadata { { "authorization", $"Bearer {token}" } };
await client.GetOrderAsync(request, headers);
```

gRPC has first-class **deadlines** — a deadline propagates to the server (which sees it via `context.Deadline`/`CancellationToken`), so the server can abandon work the client no longer awaits. **Metadata** (key-value headers) carries auth tokens, correlation ids, etc. Honor the deadline/cancellation on the server (forward `context.CancellationToken` to your async calls).

---

## Interceptors

gRPC **interceptors** are the cross-cutting pipeline (like delegating handlers / middleware) for gRPC calls — logging, auth, metrics, error mapping, on both client and server:

```csharp
public class LoggingInterceptor(ILogger<LoggingInterceptor> log) : Interceptor {
    public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(
        TRequest request, ServerCallContext context, UnaryServerMethod<TRequest, TResponse> continuation) {
        log.LogInformation("gRPC {Method}", context.Method);
        try { return await continuation(request, context); }
        catch (Exception ex) { log.LogError(ex, "gRPC call failed"); throw; }
    }
}
builder.Services.AddGrpc(o => o.Interceptors.Add<LoggingInterceptor>());
```

Use interceptors for cross-cutting concerns across all gRPC methods, analogous to ASP.NET Core filters/middleware ([Ch04 §09](../04-AspNetCore/09-Filters.md)).

---

## gRPC-Web and browsers

Browsers can't use raw gRPC (no access to the required HTTP/2 framing). **gRPC-Web** is a variant that works from browsers (via a proxy or built-in ASP.NET Core support):

```csharp
app.UseGrpcWeb();
app.MapGrpcService<OrderServiceImpl>().EnableGrpcWeb();
```

gRPC-Web enables Blazor WebAssembly and JS clients to call gRPC services (with some streaming limitations). For purely browser-facing APIs, REST/JSON is often simpler; use gRPC-Web when you want gRPC's contracts/efficiency from a browser client. (Blazor: [Ch14](../14-Blazor/README.md).)

---

## Common gotchas

### Using gRPC for public/browser APIs

Browsers need gRPC-Web (with limitations), and gRPC's binary contracts are less convenient for public consumers than REST/JSON. Use gRPC for service-to-service; REST at the public/browser edge.

### Forgetting HTTP/2 requirements

gRPC requires HTTP/2 (and usually TLS). Misconfigured servers/proxies (HTTP/1.1, TLS termination dropping HTTP/2) break gRPC. Ensure end-to-end HTTP/2.

### Not honoring deadlines/cancellation on the server

A client deadline propagates to `context.CancellationToken` — forward it to your async work so the server stops when the client gives up. Ignoring it wastes server resources.

### Throwing arbitrary exceptions instead of `RpcException`

Server errors should be `RpcException` with a meaningful `StatusCode` (NotFound, InvalidArgument). A raw exception becomes a generic `Unknown` status — less useful to clients.

### Breaking the `.proto` contract

Changing field numbers/types breaks wire compatibility. Follow protobuf evolution rules (add new fields with new numbers; don't reuse/renumber) — like API versioning ([Ch04 §06](../04-AspNetCore/06-Routing.md)).

### Not reusing the client/channel

Create the gRPC client/channel via DI (`AddGrpcClient`) so the HTTP/2 connection is pooled and reused — don't create a new channel per call.

---

## Summary

- **gRPC** is contract-first RPC: define a **`.proto`** → generate typed client/server; serializes with **Protobuf** (compact binary) over **HTTP/2** (multiplexed, streaming) — efficient, strongly-typed **service-to-service** communication.
- Choose gRPC for **internal microservices** (small payloads, fast, polyglot contracts, streaming); REST/JSON for **public/browser** APIs (native browser support, human-readable).
- Implement the generated base class (signal errors with **`RpcException`** + status codes); register typed clients with **`AddGrpcClient`** (integrates with `IHttpClientFactory`).
- Four call types — **unary, server-streaming, client-streaming, bidirectional** — with first-class **deadlines** (propagate to the server), **metadata** (headers/auth), and **interceptors** (cross-cutting pipeline).
- Needs end-to-end **HTTP/2**; use **gRPC-Web** for browsers; honor server-side cancellation; evolve `.proto` contracts compatibly.

→ Next: [08-SignalR.md](08-SignalR.md)
