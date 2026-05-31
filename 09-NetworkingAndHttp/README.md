# Chapter 09 — Networking & HTTP

> HttpClient (the right way), IHttpClientFactory, gRPC, SignalR, WebSockets. The protocols and clients that talk between services.

**Prerequisites**: Chapter 03 (Hosting & DI), CSharpBook Chapter 08 (Concurrency).

**Time to read**: ~8-10 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-HttpClient.md](01-HttpClient.md) | The big picture: HttpRequestMessage, HttpResponseMessage, SocketsHttpHandler. |
| [02-IHttpClientFactory.md](02-IHttpClientFactory.md) | Why singletons leak sockets, how the factory pools handlers, named and typed clients. |
| [03-DelegatingHandlers.md](03-DelegatingHandlers.md) | The handler chain: logging, retry, auth, metrics. |
| [04-Resilience.md](04-Resilience.md) | `AddStandardResilienceHandler` (.NET 8+), Polly integration. |
| [05-Authentication.md](05-Authentication.md) | Bearer tokens, OAuth client credentials, certificate auth. |
| [06-Sockets.md](06-Sockets.md) | Raw TCP, UDP, when you need to drop below HTTP. |
| [07-gRPC.md](07-gRPC.md) | grpc-dotnet — unary, server-streaming, client-streaming, bidirectional; interceptors. |
| [08-SignalR.md](08-SignalR.md) | Real-time hubs, client SDKs, scale-out via Redis backplane. |
| [09-WebSockets.md](09-WebSockets.md) | Raw `ClientWebSocket` and server-side WebSocket middleware. |
| [10-Pipelines.md](10-Pipelines.md) | System.IO.Pipelines for high-throughput protocol code. |
| [Questions.md](Questions.md) | Drilling. |
| [Coding.md](Coding.md) | Build a resilient HttpClient, a gRPC service, a SignalR hub. |

→ Begin: [01-HttpClient.md](01-HttpClient.md)
