# Networking (`System.Net.*`)

## The networking stack in the BCL

`System.Net.*` spans from high-level HTTP clients down to raw sockets: `HttpClient` (HTTP/1.1, HTTP/2, HTTP/3), DNS resolution, `IPAddress`/`IPEndPoint`, `Socket`, and lower-level building blocks. This file maps the stack and the must-know correctness rules (especially `HttpClient` lifetime).

> Chapter 09 (Networking & HTTP) covers `HttpClient`/`IHttpClientFactory`, gRPC, SignalR, and WebSockets in depth from the application angle. This file is the **BCL primitives** overview and the foundational gotchas.

```
Application HTTP:   HttpClient / IHttpClientFactory      [Ch09 — usage]
HTTP engine:        SocketsHttpHandler (connection pooling, HTTP/2, HTTP/3)
Names & addresses:  Dns, IPAddress, IPEndPoint, NetworkInterface
Transport:          Socket, TcpClient/TcpListener, UdpClient, NetworkStream
Security:           SslStream, TLS
```

---

## `HttpClient` — and the #1 lifetime trap

`HttpClient` is the standard HTTP client. The single most important rule:

```csharp
// ✗ — creating an HttpClient per request EXHAUSTS SOCKETS
using (var client = new HttpClient()) {      // each disposal leaves a socket in TIME_WAIT
    await client.GetAsync(url);               // thousands of requests → "address in use" failures
}

// ✓ — reuse a single instance (it's thread-safe), or use IHttpClientFactory
private static readonly HttpClient Client = new();
await Client.GetAsync(url);
```

`HttpClient` is **designed to be long-lived and shared** — it pools TCP connections internally. Creating one per request (the classic mistake) leaves sockets in `TIME_WAIT` and exhausts the port range under load. But a single static instance has its own issue: it doesn't pick up **DNS changes**. The robust answer is **`IHttpClientFactory`** (which manages handler lifetimes, rotating them periodically to honor DNS while pooling connections) — covered in [Ch09](../09-NetworkingAndHttp/README.md). The takeaway here: **never `new HttpClient()` per request.**

```csharp
// Modern usage essentials
var resp = await Client.GetAsync(url, ct);
resp.EnsureSuccessStatusCode();
var data = await resp.Content.ReadFromJsonAsync<Model>(ct);   // System.Net.Http.Json
await Client.PostAsJsonAsync(url, payload, ct);
```

---

## `SocketsHttpHandler` — the engine

Under `HttpClient` sits `SocketsHttpHandler` (the cross-platform managed HTTP implementation). It owns connection pooling, HTTP/2 multiplexing, HTTP/3 (QUIC), proxy handling, and timeouts:

```csharp
var handler = new SocketsHttpHandler {
    PooledConnectionLifetime = TimeSpan.FromMinutes(2),      // rotate connections (honor DNS)
    MaxConnectionsPerServer = 50,
    AutomaticDecompression = DecompressionMethods.All,
    ConnectTimeout = TimeSpan.FromSeconds(10),
};
var client = new HttpClient(handler);
```

`PooledConnectionLifetime` is the manual equivalent of what `IHttpClientFactory` automates — cap connection age so DNS changes are picked up while still pooling. **HTTP/3** (QUIC) is supported and increasingly default for capable endpoints (lower latency, better on lossy networks).

---

## DNS, addresses, endpoints

```csharp
using System.Net;
IPAddress[] addrs = await Dns.GetHostAddressesAsync("example.com", ct);
IPHostEntry entry = await Dns.GetHostEntryAsync("example.com");
string host = Dns.GetHostName();

var ip = IPAddress.Parse("192.168.1.1");
var loopback = IPAddress.Loopback;            // 127.0.0.1 / ::1
var endpoint = new IPEndPoint(ip, port: 8080);
bool isV6 = ip.AddressFamily == AddressFamily.InterNetworkV6;
```

`IPAddress` handles both IPv4 and IPv6; write IP-version-agnostic code (`IPAddress.IPv6Any`, dual-stack sockets). `NetworkInterface.GetAllNetworkInterfaces()` enumerates adapters (for diagnostics, picking a bind address).

---

## Sockets — the low level

When you need raw TCP/UDP (custom protocols, proxies, high-performance servers), `Socket` and the friendlier `TcpClient`/`TcpListener`/`UdpClient` wrappers:

```csharp
using System.Net.Sockets;

// TCP server
var listener = new TcpListener(IPAddress.Any, 5000);
listener.Start();
using TcpClient client = await listener.AcceptTcpClientAsync(ct);
NetworkStream stream = client.GetStream();           // a Stream — read/write bytes
int n = await stream.ReadAsync(buffer, ct);          // honor the Stream.Read contract!

// Raw socket with modern async + Span
var socket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
await socket.ConnectAsync(endpoint, ct);
int received = await socket.ReceiveAsync(memory, SocketFlags.None, ct);
```

Modern socket APIs are `ValueTask`-based, accept `Memory<byte>`/`Span<byte>` (low allocation), and support cancellation. For high-throughput servers, combine sockets with **`System.IO.Pipelines`** ([04-IO.md](04-IO.md), CSharpBook Ch13 §03) — the pattern Kestrel uses. `NetworkStream` is a `Stream`, so the **`Read` may return fewer bytes** rule applies — always loop.

---

## TLS / `SslStream`

```csharp
using var ssl = new SslStream(networkStream, leaveInnerStreamOpen: false);
await ssl.AuthenticateAsClientAsync(new SslClientAuthenticationOptions {
    TargetHost = "example.com",
    EnabledSslProtocols = SslProtocols.Tls13 | SslProtocols.Tls12,
});
// now read/write encrypted over `ssl`
```

`SslStream` wraps a transport stream with TLS. `HttpClient` does this for you; you only touch `SslStream` for custom protocols. Use TLS 1.2/1.3, validate certificates (don't disable validation in production), and prefer the OS trust store. (Cert handling: [Ch10 Identity](../10-Identity/README.md).)

---

## `Uri` — parsing and building URLs

```csharp
var uri = new Uri("https://api.example.com:443/v1/orders?id=42#section");
uri.Scheme;        // "https"
uri.Host;          // "api.example.com"
uri.AbsolutePath;  // "/v1/orders"
uri.Query;         // "?id=42"

var built = new UriBuilder("https", "example.com", 443, "/orders") { Query = "id=42" }.Uri;
// Encode query values:
string q = Uri.EscapeDataString("a&b=c");        // a%26b%3Dc
```

Use `Uri`/`UriBuilder` rather than string concatenation for URLs — they handle encoding, normalization, and validation. Always `Uri.EscapeDataString` user-supplied query/path values to prevent injection and breakage.

---

## Common gotchas

### `new HttpClient()` per request

Exhausts sockets under load (`TIME_WAIT` buildup). Reuse a shared instance or use `IHttpClientFactory` ([Ch09](../09-NetworkingAndHttp/README.md)).

### Static `HttpClient` and stale DNS

A single never-rotated `HttpClient` won't notice DNS changes. Set `PooledConnectionLifetime`, or use `IHttpClientFactory`.

### Not looping on `NetworkStream.Read`

A socket read may return fewer bytes than requested (or 0 at close). Loop or use `ReadExactly`/Pipelines (CSharpBook Ch13 §02).

### IPv4-only assumptions

Hardcoding IPv4 breaks on IPv6-only/dual-stack hosts. Use IP-version-agnostic APIs.

### Disabling cert validation

`ServerCertificateCustomValidationCallback = (_,_,_,_) => true` in production is a serious vulnerability. Validate certificates.

### Building URLs by string concat

Breaks on encoding/special chars and invites injection. Use `Uri`/`UriBuilder` + `EscapeDataString`.

---

## Summary

- `System.Net.*` ranges from **`HttpClient`** (high-level HTTP, HTTP/2 & HTTP/3 via `SocketsHttpHandler`) down to raw **`Socket`/`TcpClient`/`UdpClient`** and **`SslStream`** (TLS).
- **Never `new HttpClient()` per request** (socket exhaustion); reuse a shared instance or use **`IHttpClientFactory`** (handles pooling + DNS rotation).
- Use **`IPAddress`/`IPEndPoint`** IP-version-agnostically; **`Uri`/`UriBuilder`** + `EscapeDataString` for URLs.
- Sockets are `ValueTask`/`Memory`-based; `NetworkStream` honors the **`Read` contract** (loop); pair with **Pipelines** for high-throughput servers.
- Use TLS 1.2/1.3 and **validate certificates**.
- Application-level HTTP, gRPC, SignalR, WebSockets: [Chapter 09](../09-NetworkingAndHttp/README.md).

→ Next: [11-Cryptography.md](11-Cryptography.md)
