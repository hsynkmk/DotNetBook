# WebSockets

## Low-level bidirectional communication

WebSockets provide a **full-duplex** channel over a single long-lived TCP connection — both client and server can send messages anytime, with low overhead (no per-message HTTP headers). It's the transport beneath **SignalR** ([08-SignalR.md](08-SignalR.md)); you use the **raw** WebSocket API directly only when you need low-level control or SignalR's model doesn't fit.

```csharp
// Server — accept a WebSocket on an endpoint
app.UseWebSockets();
app.Map("/ws", async context => {
    if (!context.WebSockets.IsWebSocketRequest) { context.Response.StatusCode = 400; return; }
    using WebSocket socket = await context.WebSockets.AcceptWebSocketAsync();
    await EchoLoopAsync(socket, context.RequestAborted);
});
```

```csharp
// Client — ClientWebSocket
using var ws = new ClientWebSocket();
await ws.ConnectAsync(new Uri("wss://example.com/ws"), ct);
await ws.SendAsync(Encoding.UTF8.GetBytes("hello"), WebSocketMessageType.Text, endOfMessage: true, ct);
```

---

## The send/receive loop

A WebSocket connection is a continuous loop of receiving and sending messages until closed:

```csharp
async Task EchoLoopAsync(WebSocket socket, CancellationToken ct) {
    var buffer = new byte[4096];
    while (socket.State == WebSocketState.Open) {
        WebSocketReceiveResult result = await socket.ReceiveAsync(buffer, ct);

        if (result.MessageType == WebSocketMessageType.Close) {
            await socket.CloseAsync(WebSocketCloseStatus.NormalClosure, "bye", ct);   // handshake close
            break;
        }
        // Echo the received message back
        await socket.SendAsync(buffer.AsMemory(0, result.Count), result.MessageType, result.EndOfMessage, ct);
    }
}
```

Key details:
- **Message framing exists** (unlike raw TCP — [06-Sockets.md](06-Sockets.md)): WebSockets are message-oriented, but a large message may arrive in **multiple frames** — `result.EndOfMessage` tells you when a message is complete (loop receives until `EndOfMessage` for big messages).
- **Message types**: `Text` (UTF-8), `Binary`, and `Close` (the close handshake).
- **Closing** is a handshake — call `CloseAsync` to close cleanly; handle the peer's close frame.

---

## When to use raw WebSockets vs SignalR

| | Raw WebSockets | SignalR |
|---|---|---|
| Level | low (you manage frames, reconnection, scale) | high (hubs, groups, fallbacks) |
| Reconnection | DIY | built-in (`withAutomaticReconnect`) |
| Transport fallback | none (WebSockets only) | SSE/long-polling fallbacks |
| Targeting (groups/users) | DIY | built-in |
| Scale-out | DIY backplane | Redis / Azure SignalR |
| Use | a specific WS protocol, full control, minimal overhead | real-time app features |

**Prefer SignalR** for application real-time features — it adds reconnection, transport fallback, group/user targeting, and scale-out that you'd otherwise build by hand. Use **raw WebSockets** when: you must implement a **specific WebSocket sub-protocol** (a third party's), you want **minimal overhead/dependencies**, you're talking to a non-SignalR WebSocket server, or you need precise frame-level control. For most apps, that's a narrow set of cases — SignalR is the default for real-time.

---

## Server-side considerations

```csharp
builder.Services.Configure(...);
app.UseWebSockets(new WebSocketOptions {
    KeepAliveInterval = TimeSpan.FromSeconds(30)   // ping to keep the connection alive / detect drops
});
```

- **`UseWebSockets`** middleware enables WebSocket support; check `IsWebSocketRequest` before accepting.
- **Keep-alive pings** detect dead connections (a TCP connection can silently die; pings surface it).
- **Connection lifetime** — each accepted socket holds a request thread/connection for its entire (long) lifetime. Manage the number of concurrent connections; a WebSocket server holds many long-lived connections (memory/resource planning).
- **Authentication** — the WebSocket upgrade request carries cookies/headers, so standard auth applies at accept time; but tokens can't be sent in headers from browser WebSocket APIs easily (often passed via query string or a first message) — handle auth at the upgrade ([Ch10](../10-Identity/README.md)).

---

## Scaling raw WebSockets

Like SignalR, raw WebSocket connections are **stateful and instance-bound** — a client connected to instance A can't be reached from instance B without coordination. With raw WebSockets you must build this yourself (a pub/sub backplane via Redis, a message broker, etc.) — which is exactly the machinery SignalR's backplane provides ([08-SignalR.md](08-SignalR.md)). This is a major reason to use SignalR rather than raw WebSockets for anything that scales horizontally: SignalR's scale-out is solved; raw WebSocket scale-out is your problem.

---

## Common gotchas

### Not handling multi-frame messages

A large message arrives in multiple frames; reading one `ReceiveAsync` may not give the whole message. Loop until `result.EndOfMessage` to assemble it.

### Not handling the close handshake

Ignoring `Close` message type / not calling `CloseAsync` leaves connections half-closed. Handle the close handshake on both sides.

### No keep-alive → silent dead connections

A TCP connection can die silently (NAT timeout, network drop). Without keep-alive pings, the server holds a dead connection. Configure `KeepAliveInterval` and detect drops.

### Building SignalR's features by hand

Reconnection, transport fallback, group targeting, and scale-out are substantial to implement on raw WebSockets. If you find yourself building these, use **SignalR**.

### Unbounded connections

Each WebSocket is a long-lived connection holding resources. A server with no connection limit can exhaust memory/sockets under many clients. Bound and monitor connection counts.

### Scaling without a backplane

Raw WebSockets are instance-bound — multi-instance broadcast needs your own backplane. SignalR (with Redis/Azure SignalR) solves this; raw WebSockets don't.

---

## Summary

- **WebSockets** are a low-level, **full-duplex**, long-lived connection — both sides send anytime with low overhead; they're the transport beneath SignalR.
- The model is a **receive/send loop**: handle multi-frame messages (`EndOfMessage`), message types (`Text`/`Binary`/`Close`), and the **close handshake** (`CloseAsync`); enable server support with `UseWebSockets` + keep-alive pings.
- **Prefer SignalR** for real-time app features — it adds reconnection, transport fallback, group/user targeting, and **scale-out** (backplane) that raw WebSockets make you build yourself.
- Use **raw WebSockets** only for specific sub-protocols, minimal-dependency/full-control needs, or non-SignalR servers; mind connection lifetime/limits and that scale-out (instance-bound connections) is your responsibility without a backplane.

→ Next: [10-Pipelines.md](10-Pipelines.md)
