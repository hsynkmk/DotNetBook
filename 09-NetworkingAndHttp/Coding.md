# Chapter 09 — Networking & HTTP — Coding Problems

Build a resilient typed HttpClient with auth, then a gRPC service and a SignalR hub.

---

## Problem 1: A typed client (no socket exhaustion)

Build a typed `HttpClient` for an external API, configured via the factory.

<details><summary>Solution</summary>

```csharp
public class WeatherClient(HttpClient http) {
    public async Task<Forecast?> GetAsync(string city, CancellationToken ct) =>
        await http.GetFromJsonAsync<Forecast>($"forecast?city={Uri.EscapeDataString(city)}", ct);
}

builder.Services.AddHttpClient<WeatherClient>(c => {
    c.BaseAddress = new Uri("https://api.weather.example/");
    c.DefaultRequestHeaders.Add("User-Agent", "MyApp/1.0");
    c.Timeout = TimeSpan.FromSeconds(10);
});

// Inject WeatherClient directly — its HttpClient is factory-managed (pooled, rotated)
public class WeatherController(WeatherClient client) { ... }
```

Typed client = clean API + factory-managed handler (no socket exhaustion, DNS rotation). `Uri.EscapeDataString` for the query value. ([02-IHttpClientFactory.md](02-IHttpClientFactory.md).)

</details>

---

## Problem 2: Add resilience

Make the client survive transient failures with retry, circuit breaker, and timeouts.

<details><summary>Solution</summary>

```csharp
builder.Services.AddHttpClient<WeatherClient>(c => c.BaseAddress = new Uri("https://api.weather.example/"))
    .AddStandardResilienceHandler(o => {
        o.Retry.MaxRetryAttempts = 3;                          // exponential backoff + jitter (defaults)
        o.CircuitBreaker.FailureRatio = 0.5;                    // open if >50% fail
        o.AttemptTimeout.Timeout = TimeSpan.FromSeconds(5);     // per-try
        o.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(20); // whole operation incl. retries
    });
```

One line adds Polly-based retry (only safe failures, with backoff/jitter), a circuit breaker (fail fast when the service is down), and timeouts. Note: GETs are idempotent so retry is safe here. ([04-Resilience.md](04-Resilience.md).)

</details>

---

## Problem 3: Add bearer-token auth via a handler

Attach an OAuth client-credentials token to every request, centralized in a handler.

<details><summary>Solution</summary>

```csharp
public class BearerTokenHandler(ITokenProvider tokens) : DelegatingHandler {
    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken ct) {
        request.Headers.Authorization = new("Bearer", await tokens.GetTokenAsync(ct));   // cached + auto-refreshed
        return await base.SendAsync(request, ct);
    }
}

builder.Services.AddSingleton<ITokenProvider, ClientCredentialsTokenProvider>();  // caches the token
builder.Services.AddTransient<BearerTokenHandler>();
builder.Services.AddHttpClient<ApiClient>(c => c.BaseAddress = new Uri("https://api.example/"))
    .AddHttpMessageHandler<BearerTokenHandler>()
    .AddStandardResilienceHandler();
```

The handler attaches the token to every outbound request; the token provider caches and refreshes it (don't fetch per request). Secret comes from config/Key Vault, not source. ([03-DelegatingHandlers.md](03-DelegatingHandlers.md), [05-Authentication.md](05-Authentication.md).)

</details>

---

## Problem 4: Handle expected vs exceptional status codes

Return null for 404, throw for other failures.

<details><summary>Solution</summary>

```csharp
public async Task<Product?> GetProductAsync(int id, CancellationToken ct) {
    var response = await http.GetAsync($"products/{id}", ct);
    if (response.StatusCode == HttpStatusCode.NotFound) return null;   // EXPECTED — control flow, not exception
    response.EnsureSuccessStatusCode();                                 // other non-2xx → throw
    return await response.Content.ReadFromJsonAsync<Product>(ct);
}
```

404 (not found) is an expected outcome → handle as control flow; other failures (5xx, etc.) are exceptional → `EnsureSuccessStatusCode` throws. `HttpClient` doesn't throw on non-2xx by itself. ([01-HttpClient.md](01-HttpClient.md).)

</details>

---

## Problem 5: Stream a large download

Download a large file without buffering it all in memory.

<details><summary>Solution</summary>

```csharp
public async Task DownloadAsync(string url, string destPath, CancellationToken ct) {
    using var response = await http.GetAsync(url, HttpCompletionOption.ResponseHeadersRead, ct);  // don't buffer body
    response.EnsureSuccessStatusCode();
    await using var source = await response.Content.ReadAsStreamAsync(ct);
    await using var dest = File.Create(destPath);
    await source.CopyToAsync(dest, ct);   // stream to disk, constant memory
}
```

`ResponseHeadersRead` returns once headers arrive (not after buffering the whole body); streaming to disk keeps memory constant regardless of file size. ([01-HttpClient.md](01-HttpClient.md).)

</details>

---

## Problem 6: A gRPC service (unary + server streaming)

Define and implement a gRPC service with a unary and a streaming method.

<details><summary>Solution</summary>

```protobuf
// orders.proto
service OrderService {
  rpc GetOrder (GetOrderRequest) returns (Order);
  rpc StreamOrders (StreamRequest) returns (stream Order);
}
```

```csharp
builder.Services.AddGrpc();
app.MapGrpcService<OrderServiceImpl>();

public class OrderServiceImpl(IOrderRepository repo) : OrderService.OrderServiceBase {
    public override async Task<Order> GetOrder(GetOrderRequest req, ServerCallContext ctx) {
        var o = await repo.GetAsync(req.Id, ctx.CancellationToken)
            ?? throw new RpcException(new Status(StatusCode.NotFound, $"Order {req.Id} not found"));
        return new Order { Id = o.Id, Customer = o.Customer, Total = (double)o.Total };
    }

    public override async Task StreamOrders(StreamRequest req, IServerStreamWriter<Order> stream, ServerCallContext ctx) {
        await foreach (var o in repo.StreamAsync(ctx.CancellationToken))
            await stream.WriteAsync(new Order { Id = o.Id, Customer = o.Customer, Total = (double)o.Total });
    }
}
```

Implement the generated base class; signal not-found with `RpcException` + status code; honor `ctx.CancellationToken` (the client's deadline). Server streaming yields results as produced. ([07-gRPC.md](07-gRPC.md).)

</details>

---

## Problem 7: A gRPC client (typed, factory-integrated)

Call the gRPC service from another service with a deadline.

<details><summary>Solution</summary>

```csharp
builder.Services.AddGrpcClient<OrderService.OrderServiceClient>(o =>
    o.Address = new Uri("https://orders-service:5001"));

public class OrderConsumer(OrderService.OrderServiceClient client) {
    public async Task<Order> GetAsync(int id, CancellationToken ct) =>
        await client.GetOrderAsync(new GetOrderRequest { Id = id },
            deadline: DateTime.UtcNow.AddSeconds(5), cancellationToken: ct);   // deadline propagates to server

    public async Task ConsumeStreamAsync(CancellationToken ct) {
        using var call = client.StreamOrders(new StreamRequest());
        await foreach (var order in call.ResponseStream.ReadAllAsync(ct))
            Process(order);
    }
}
```

`AddGrpcClient` gives a typed, pooled (factory-integrated) client; the deadline propagates so the server can abandon work if the client gives up. ([07-gRPC.md](07-gRPC.md).)

</details>

---

## Problem 8: A SignalR hub with groups

Build a chat hub where clients join rooms and messages broadcast to a room.

<details><summary>Solution</summary>

```csharp
public interface IChatClient {
    Task ReceiveMessage(string user, string message);
    Task UserJoined(string user);
}

public class ChatHub : Hub<IChatClient> {        // strongly-typed
    public async Task JoinRoom(string room) {
        await Groups.AddToGroupAsync(Context.ConnectionId, room);
        await Clients.Group(room).UserJoined(Context.User?.Identity?.Name ?? "anon");
    }
    public async Task SendToRoom(string room, string message) =>
        await Clients.Group(room).ReceiveMessage(Context.User?.Identity?.Name ?? "anon", message);
}

builder.Services.AddSignalR().AddStackExchangeRedis(redisConnectionString);   // backplane for scale-out
var app = builder.Build();
app.MapHub<ChatHub>("/chat");
```

Strongly-typed hub (compile-checked client calls), groups for rooms, Redis backplane so it works across multiple instances. Clients must re-join the room on reconnect (new connection id). ([08-SignalR.md](08-SignalR.md).)

</details>

---

## Problem 9: Choose the right networking tech

For each, pick the technology and justify.
1. A public REST-ish JSON API consumed by browsers and third parties.
2. Two internal microservices exchanging high-volume typed messages with streaming.
3. A live dashboard pushing updates to thousands of browser clients.
4. A custom binary protocol over TCP to a piece of hardware.

<details><summary>Solution</summary>

1. **HTTP + JSON (ASP.NET Core REST)** — native browser support, human-readable, ubiquitous tooling; gRPC's binary contracts don't suit public/browser consumers. ([Ch04](../04-AspNetCore/README.md).)
2. **gRPC** — Protobuf over HTTP/2: compact, fast, strong contracts shared across services, built-in streaming — ideal for service-to-service. ([07-gRPC.md](07-gRPC.md).)
3. **SignalR** (with a Redis backplane or Azure SignalR) — server-to-client push with groups/user targeting, transport fallback, reconnection, and scale-out for thousands of connections. ([08-SignalR.md](08-SignalR.md).)
4. **Raw sockets** (`System.Net.Sockets`) with manual framing (length-prefix), possibly + `System.IO.Pipelines` for throughput — HTTP doesn't fit a custom binary hardware protocol. ([06-Sockets.md](06-Sockets.md)/[10-Pipelines.md](10-Pipelines.md).)

The principle: REST/JSON at the public/browser edge; gRPC for typed service-to-service; SignalR for real-time browser push; raw sockets only for custom protocols.

</details>

---

## Problem 10: Resilient client calling a flaky service (full picture)

Compose a typed client with auth, resilience, and correct error handling for a critical downstream.

<details><summary>Solution</summary>

```csharp
builder.Services.AddTransient<BearerTokenHandler>();
builder.Services.AddHttpClient<PaymentClient>(c => {
        c.BaseAddress = new Uri("https://payments.example/");
        c.DefaultRequestHeaders.Add("User-Agent", "Checkout/1.0");
    })
    .AddHttpMessageHandler<BearerTokenHandler>()       // auth on every call
    .AddStandardResilienceHandler(o => {                // retry + breaker + timeouts
        o.Retry.MaxRetryAttempts = 2;                    // payments: be conservative with retries
        o.AttemptTimeout.Timeout = TimeSpan.FromSeconds(8);
    });

public class PaymentClient(HttpClient http) {
    public async Task<PaymentResult> ChargeAsync(ChargeRequest req, CancellationToken ct) {
        // Idempotency key so a RETRY doesn't double-charge (server dedupes on it)
        using var request = new HttpRequestMessage(HttpMethod.Post, "charge") {
            Content = JsonContent.Create(req),
            Headers = { { "Idempotency-Key", req.IdempotencyKey } }
        };
        var response = await http.SendAsync(request, ct);
        if (response.StatusCode == HttpStatusCode.Conflict) return PaymentResult.Duplicate;  // expected
        response.EnsureSuccessStatusCode();
        return (await response.Content.ReadFromJsonAsync<PaymentResult>(ct))!;
    }
}
```

Auth handler + resilience + an **idempotency key** so retrying the non-idempotent POST can't double-charge (the server dedupes) — the correct way to retry a payment. Conservative retry count; expected 409 handled as control flow. ([03](03-DelegatingHandlers.md)/[04](04-Resilience.md)/[05](05-Authentication.md), [Ch07 §07](../07-Messaging/07-Patterns.md).)

</details>

---

You can now build production HTTP clients (typed, factory-managed, resilient, authenticated, with correct error handling and idempotent retries), gRPC services and clients (unary + streaming, deadlines), and SignalR hubs (groups, scale-out) — and choose the right networking technology per scenario.

→ Back to [Chapter 09 README](README.md) · Next chapter: [Chapter 10 — Identity & Security](../10-Identity/README.md)
