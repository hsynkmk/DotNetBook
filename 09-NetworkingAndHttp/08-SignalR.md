# SignalR

## Real-time, server-to-client push

SignalR is ASP.NET Core's library for **real-time** communication — the server can **push** messages to connected clients (and clients invoke server methods) without the client polling. It powers live dashboards, chat, notifications, collaborative editing, live sports/trading feeds — anything where the server needs to update clients as things happen. It abstracts the transport (WebSockets, with fallbacks) behind a simple **hub** API.

```csharp
// Server — a Hub
public class ChatHub : Hub {
    public async Task SendMessage(string user, string message) =>
        await Clients.All.SendAsync("ReceiveMessage", user, message);   // PUSH to all connected clients
}

// Program.cs
builder.Services.AddSignalR();
app.MapHub<ChatHub>("/chat");
```

```javascript
// Client (JS) — connect and react to pushes
const conn = new signalR.HubConnectionBuilder().withUrl("/chat").build();
conn.on("ReceiveMessage", (user, msg) => addMessage(user, msg));   // server-pushed
await conn.start();
await conn.invoke("SendMessage", "Alice", "Hello");                 // call a hub method
```

---

## Hubs — the programming model

A **hub** is a server class (deriving from `Hub`) whose public methods clients can invoke, and which can push to clients via the `Clients` property:

```csharp
public class NotificationHub : Hub {
    // Client → server: a method clients call
    public async Task Subscribe(string topic) => await Groups.AddToGroupAsync(Context.ConnectionId, topic);

    // Server → clients: targeting different audiences
    public async Task Broadcast(string msg) {
        await Clients.All.SendAsync("Notify", msg);                        // everyone
        await Clients.Caller.SendAsync("Ack");                              // just the caller
        await Clients.Others.SendAsync("Notify", msg);                     // everyone except caller
        await Clients.Group("topic").SendAsync("Notify", msg);             // a group
        await Clients.User(userId).SendAsync("Notify", msg);               // a specific user (all their connections)
        await Clients.Client(connectionId).SendAsync("Notify", msg);       // one connection
    }

    public override Task OnConnectedAsync() { /* track connection */ return base.OnConnectedAsync(); }
    public override Task OnDisconnectedAsync(Exception? ex) { /* cleanup */ return base.OnDisconnectedAsync(ex); }
}
```

The hub gives you **targeting**: push to all, the caller, others, a **group** (named set of connections — e.g., a chat room or topic), a **user** (all their connections across devices), or a specific connection. `OnConnectedAsync`/`OnDisconnectedAsync` track lifecycle. This high-level model hides the transport entirely.

---

## Transports & fallback

SignalR negotiates the best available transport:

```
WebSockets         — preferred (full-duplex, low overhead)
Server-Sent Events — fallback (server→client only)
Long Polling       — last-resort fallback (works everywhere)
```

It uses **WebSockets** where available (the efficient full-duplex transport — [09-WebSockets.md](09-WebSockets.md)) and falls back to Server-Sent Events or long polling when WebSockets aren't (old proxies, restrictive networks). You write hub code once; SignalR picks the transport. This **automatic fallback** is a key reason to use SignalR over raw WebSockets — it just works across environments.

---

## Strongly-typed hubs

Instead of magic string method names (`SendAsync("ReceiveMessage", ...)`), define a client interface for compile-time safety:

```csharp
public interface IChatClient {
    Task ReceiveMessage(string user, string message);
    Task UserJoined(string user);
}

public class ChatHub : Hub<IChatClient> {                  // strongly-typed hub
    public async Task SendMessage(string user, string msg) =>
        await Clients.All.ReceiveMessage(user, msg);        // no magic strings — compile-checked
}
```

`Hub<TClient>` makes client method calls strongly-typed (the method name and signature are checked at compile time) — preferable to stringly-typed `SendAsync` calls, which are easy to mistype and don't refactor.

---

## Scaling out — the backplane

The challenge: SignalR connections are **stateful** and tied to a **specific server instance**. With multiple instances behind a load balancer, a message sent on instance A won't reach a client connected to instance B — unless the instances coordinate. The solution is a **backplane** that broadcasts messages across all instances:

```csharp
// Redis backplane — all instances publish/subscribe so messages reach clients on ANY instance
builder.Services.AddSignalR().AddStackExchangeRedis(redisConnectionString);

// Or Azure SignalR Service — a fully-managed backplane (offloads connections entirely)
builder.Services.AddSignalR().AddAzureSignalR(connectionString);
```

- **Redis backplane** — instances publish messages to Redis; all instances subscribe and forward to their local connections. So `Clients.All.SendAsync(...)` reaches every client across all instances.
- **Azure SignalR Service** — a managed service that **holds the connections** (offloading them from your app servers entirely) and routes messages — scales to massive connection counts without your servers managing the WebSocket sockets.

Without a backplane (or Azure SignalR), a scaled-out SignalR app **breaks** — clients miss messages sent from other instances. This is the #1 SignalR scaling gotcha. (Also requires **sticky sessions** at the load balancer for the initial negotiation, unless using Azure SignalR.)

---

## Connection management & reliability

```javascript
// Client: automatic reconnect with backoff
const conn = new signalR.HubConnectionBuilder()
    .withUrl("/chat")
    .withAutomaticReconnect([0, 2000, 10000, 30000])   // retry delays
    .build();
conn.onreconnecting(() => showReconnecting());
conn.onreconnected(() => resubscribe());                // RE-JOIN groups after reconnect!
```

SignalR connections drop (network blips, server restarts, scale events). Clients should **auto-reconnect** (`withAutomaticReconnect`), and — critically — **re-establish state** on reconnect (re-join groups, re-subscribe), because a reconnect is a *new* connection with a new connection id (group memberships are lost). Server-side, don't store critical per-connection state only in memory (it's lost on disconnect/reconnect/scale) — back it with a store keyed by user/connection.

---

## SignalR vs gRPC vs WebSockets

| | SignalR | gRPC streaming | Raw WebSockets |
|---|---|---|---|
| Use | real-time **server→client push**, broadcast | typed RPC + streaming, service-to-service | low-level bidirectional |
| Transport | WebSockets + fallbacks (auto) | HTTP/2 | WebSockets only |
| Targeting (groups/users) | **built-in** | manual | manual |
| Scale-out | backplane / Azure SignalR | load balancer | manual |
| Client | JS, .NET, Java, etc. | generated, polyglot | any WebSocket client |

Use **SignalR** for browser/app real-time features (chat, notifications, live UI updates) — it gives groups, user targeting, fallbacks, and reconnection out of the box. Use **gRPC** for typed service-to-service streaming. Use **raw WebSockets** ([09-WebSockets.md](09-WebSockets.md)) only when you need low-level control and SignalR's model doesn't fit.

---

## Common gotchas

### No backplane when scaled out

Multiple instances without a Redis backplane / Azure SignalR → clients miss messages from other instances. Add a backplane (and sticky sessions) for scale-out. The #1 SignalR scaling bug.

### Losing group membership on reconnect

A reconnect is a new connection (new id) — group memberships and per-connection state are gone. Re-join groups / re-subscribe in `onreconnected`.

### Storing critical state in hub memory

Per-connection in-memory state is lost on disconnect/reconnect/scale. Back important state with a store (DB/cache) keyed by user, not just connection id.

### Long-running work in a hub method

Hub methods should be quick (they hold the connection). Offload long work to a background queue ([Ch08](../08-BackgroundProcessing/README.md)) and push the result via the hub when done.

### Stringly-typed client calls

`SendAsync("MethodName", ...)` is mistype-prone. Use `Hub<TClient>` for compile-checked client method calls.

### Forgetting auth on hubs

Hubs need authorization like endpoints (`[Authorize]` on the hub/methods). The `Context.User` gives the authenticated principal; secure hubs that expose sensitive operations ([Ch10](../10-Identity/README.md)).

---

## Summary

- **SignalR** provides real-time **server-to-client push** (and client→server calls) via **hubs** — for chat, notifications, live dashboards, collaborative apps — abstracting the transport.
- **Hubs** offer targeting: `Clients.All`/`Caller`/`Others`/`Group`/`User`/`Client`; use **strongly-typed hubs** (`Hub<TClient>`) over magic-string method names.
- It auto-negotiates **WebSockets** (preferred) with **SSE/long-polling fallbacks** — write hub code once, works across environments.
- **Scaling out requires a backplane** (Redis) or **Azure SignalR Service** (managed, holds connections) — without it, multi-instance apps drop cross-instance messages (the #1 scaling gotcha); also needs sticky sessions.
- Clients should **auto-reconnect** and **re-establish state** (re-join groups) on reconnect; keep hub methods quick (offload long work); secure hubs with authorization. Use SignalR for real-time UI; gRPC for typed service streaming; raw WebSockets for low-level control.

→ Next: [09-WebSockets.md](09-WebSockets.md)
