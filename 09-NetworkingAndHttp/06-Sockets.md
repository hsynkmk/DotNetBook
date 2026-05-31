# Sockets

## Below HTTP — raw TCP and UDP

Most networking uses HTTP (`HttpClient`) or higher-level protocols (gRPC, SignalR). But sometimes you need the raw transport: implementing a custom protocol, a game server, a proxy, a low-latency service, or talking to legacy/binary systems. `System.Net.Sockets` gives you TCP and UDP directly.

> The BCL socket types and connection essentials are introduced in [Ch02 §10](../02-BCL/10-Net.md). This file covers building TCP/UDP services and **when** to drop below HTTP (rarely — and deliberately).

```csharp
using System.Net.Sockets;

// TCP client
using var client = new TcpClient();
await client.ConnectAsync("example.com", 9000, ct);
NetworkStream stream = client.GetStream();    // a Stream — read/write bytes
await stream.WriteAsync(requestBytes, ct);
int read = await stream.ReadAsync(buffer, ct);   // honor the Read contract — may return fewer bytes!
```

---

## TCP — reliable, ordered byte stream

TCP gives a reliable, ordered, connection-oriented **byte stream**. `TcpListener` (server) / `TcpClient` (client) are the friendly wrappers; `Socket` is the low-level type:

```csharp
// TCP server (accept loop)
var listener = new TcpListener(IPAddress.Any, 9000);
listener.Start();
while (!ct.IsCancellationRequested) {
    using TcpClient conn = await listener.AcceptTcpClientAsync(ct);
    _ = HandleClientAsync(conn, ct);          // handle each connection concurrently
}

async Task HandleClientAsync(TcpClient conn, CancellationToken ct) {
    await using NetworkStream stream = conn.GetStream();
    var buffer = new byte[4096];
    int read;
    while ((read = await stream.ReadAsync(buffer, ct)) > 0)   // 0 = client closed
        await ProcessAndRespondAsync(stream, buffer.AsMemory(0, read), ct);
}
```

The crucial detail: **TCP is a byte *stream*, not a message protocol.** `ReadAsync` may return any number of bytes — a single read might contain part of a message, a whole message, or several messages. You must implement **framing** (below) to recover message boundaries. This is the #1 raw-socket mistake (CSharpBook Ch13 §02, [Ch02 §10](../02-BCL/10-Net.md)).

---

## Message framing

Because TCP doesn't preserve message boundaries, you must frame messages yourself — typically **length-prefixed** (write the length, then the payload) or **delimiter-based** (e.g., newline-terminated):

```csharp
// Length-prefixed framing: [4-byte length][payload]
async Task SendMessageAsync(NetworkStream s, ReadOnlyMemory<byte> payload, CancellationToken ct) {
    var lengthPrefix = new byte[4];
    BinaryPrimitives.WriteInt32BigEndian(lengthPrefix, payload.Length);   // explicit endianness ([Ch02 §02])
    await s.WriteAsync(lengthPrefix, ct);
    await s.WriteAsync(payload, ct);
}

async Task<byte[]> ReadMessageAsync(NetworkStream s, CancellationToken ct) {
    var lengthBuf = new byte[4];
    await s.ReadExactlyAsync(lengthBuf, ct);                              // read EXACTLY 4 bytes (loops internally)
    int length = BinaryPrimitives.ReadInt32BigEndian(lengthBuf);
    var payload = new byte[length];
    await s.ReadExactlyAsync(payload, ct);                                // read exactly `length` bytes
    return payload;
}
```

`ReadExactlyAsync` (.NET 7+) reads exactly N bytes (looping internally), which is what framing needs — vs raw `ReadAsync` which may return fewer. Use explicit endianness (`BinaryPrimitives` — [Ch02 §02](../02-BCL/02-Numerics.md)) for the length prefix so the protocol is portable. For high-throughput framing, **`System.IO.Pipelines`** ([10-Pipelines.md](10-Pipelines.md)) handles partial reads and buffering elegantly — it's what Kestrel uses.

---

## UDP — fast, connectionless, unreliable

UDP sends discrete **datagrams** with no connection, no ordering, and no delivery guarantee — fast and low-overhead, for cases where occasional loss is acceptable:

```csharp
using var udp = new UdpClient();
await udp.SendAsync(datagram, "example.com", 9001, ct);     // fire-and-forget; may be lost
UdpReceiveResult result = await udp.ReceiveAsync(ct);        // receive a datagram (whole message, preserved)
```

Unlike TCP, each UDP `Send`/`Receive` is one **datagram** — message boundaries *are* preserved (no framing needed), but delivery isn't guaranteed, datagrams can arrive out of order or be duplicated, and there's a size limit. Use UDP for: real-time data where stale-but-fast beats reliable-but-late (game state, live telemetry, voice/video), discovery/broadcast, and DNS-like request/response. If you need reliability *and* low latency, modern stacks use **QUIC** (the basis of HTTP/3) rather than building reliability on UDP by hand.

---

## Modern async socket APIs

The modern `Socket` API is `ValueTask`-based and takes `Memory<byte>`/`Span<byte>` (low allocation), with cancellation:

```csharp
var socket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
await socket.ConnectAsync(endpoint, ct);
int received = await socket.ReceiveAsync(memory, SocketFlags.None, ct);
await socket.SendAsync(data, SocketFlags.None, ct);
```

Use the `Memory`-based overloads (not the old `byte[], int, int` ones) and forward `CancellationToken`. For TLS over a raw socket, wrap the `NetworkStream` in an `SslStream` ([Ch02 §10](../02-BCL/10-Net.md)).

---

## When to drop below HTTP (and when not to)

Raw sockets are rarely the right choice for application code. Consider them only for:
- **Custom binary protocols** (existing systems, hardware, game networking).
- **Extreme low latency / high throughput** where HTTP overhead matters and you control both ends.
- **Non-request/response patterns** HTTP doesn't fit (continuous streams, multicast).
- **Proxies / network tools** operating at the transport level.

For almost everything else, use a **higher-level protocol**: **HTTP** (`HttpClient`/ASP.NET Core), **gRPC** ([07-gRPC.md](07-gRPC.md)) for efficient typed RPC, **SignalR** ([08-SignalR.md](08-SignalR.md)) for real-time, or **WebSockets** ([09-WebSockets.md](09-WebSockets.md)) for bidirectional. These give you framing, serialization, auth, resilience, and tooling for free. Raw sockets mean you build all of that yourself — only worth it when you have a specific, measured reason.

---

## Common gotchas

### Treating TCP reads as messages

TCP is a byte stream — a read may return partial or multiple messages. Implement **framing** (length-prefix/delimiter) and use `ReadExactlyAsync`; don't assume one read = one message.

### Not looping on `ReadAsync`

A single `ReadAsync` may return fewer bytes than requested (0 = closed). Loop, or use `ReadExactlyAsync` for known lengths.

### Assuming UDP reliability/ordering

UDP datagrams can be lost, reordered, or duplicated. Don't use it where you need guaranteed delivery (use TCP/QUIC); use it where speed beats reliability.

### Endianness assumptions in protocols

Multi-byte fields (length prefixes) must use explicit endianness (`BinaryPrimitives`) for cross-platform interop, not machine-dependent `BitConverter`.

### Building HTTP/reliability on raw sockets

Reinventing HTTP, framing, retry, and TLS on raw sockets is a huge, bug-prone effort. Use a higher-level protocol unless you have a specific reason.

### Leaking sockets / not disposing

Each connection holds OS resources. Dispose `TcpClient`/`Socket`/`NetworkStream` (`using`/`await using`), and bound concurrent connections.

---

## Summary

- **Sockets** (`System.Net.Sockets`) give raw **TCP** (reliable, ordered byte *stream*) and **UDP** (fast, connectionless datagrams, unreliable) — below HTTP.
- TCP is a **byte stream, not messages** — implement **framing** (length-prefix/delimiter) with `ReadExactlyAsync` and explicit endianness; the #1 raw-socket bug is treating reads as messages.
- **UDP** preserves datagram boundaries but not delivery/order — use for real-time/loss-tolerant data; for reliable + low-latency, prefer **QUIC/HTTP3** over hand-rolled UDP reliability.
- Use modern `Memory`-based async socket APIs (+ `CancellationToken`); pair with **Pipelines** ([10](10-Pipelines.md)) for high-throughput framing.
- **Rarely drop below HTTP** — prefer HTTP/gRPC/SignalR/WebSockets (which give framing, serialization, auth, resilience for free); use raw sockets only for custom protocols or measured extreme-performance needs.

→ Next: [07-gRPC.md](07-gRPC.md)
