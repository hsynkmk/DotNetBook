# Render Modes

## The single most important Blazor decision

In modern Blazor (.NET 8+, current in .NET 10), every interactive component runs in a **render mode** that determines *where its C# executes and how interactivity is delivered*. This isn't a cosmetic setting — it dictates latency, offline capability, download size, server load, and what APIs you can call. Choosing render modes well is the central architectural decision in a Blazor app.

```razor
@* Opt a component into a render mode at its usage site or definition *@
<Counter @rendermode="InteractiveServer" />
<Counter @rendermode="InteractiveWebAssembly" />
<Counter @rendermode="InteractiveAuto" />

@* Or declare it in the component file: *@
@rendermode InteractiveServer
```

---

## The modes at a glance

| Mode | Where C# runs | Interactivity | First load | Offline | Server load |
|---|---|---|---|---|---|
| **Static SSR** | server (once) | none (plain HTML) | fastest | n/a | minimal |
| **Interactive Server** | server, live | over SignalR WebSocket | fast | no | per-user circuit |
| **Interactive WebAssembly** | browser | local (no round-trip) | slower (download .NET) | yes | minimal after load |
| **Interactive Auto** | server → then browser | server first, WASM later | fast then local | yes (after) | transitional |

The default is **Static SSR** — the component renders to HTML on the server once and is *not* interactive. You opt specific components into interactivity.

---

## Static Server-Side Rendering (SSR)

The component renders to HTML on the server and is sent to the browser as plain markup — **no interactivity, no client runtime**. It's the fastest first paint and ideal for content (marketing pages, blog posts, read-only views). Modern Blazor adds **streaming SSR** (flush HTML as it's produced, so slow data doesn't block first paint) and **enhanced navigation** (intercept link clicks/form posts and patch the DOM instead of full page loads) — giving SSR an SPA-like feel without any interactive runtime.

```razor
@* No @rendermode → static SSR. Great for SEO and speed. *@
<article>@Post.Body</article>
```

Use static SSR by default; add interactivity only where you actually need it.

---

## Interactive Server

The component runs **on the server**, and interactivity flows over a persistent **SignalR WebSocket** ([Ch09 §08](../09-NetworkingAndHttp/08-SignalR.md)) called a **circuit**. A click in the browser sends the event to the server, your C# runs server-side, the render diff is computed, and the **DOM diff** is sent back over the wire.

- **Pros**: tiny download (no .NET runtime in the browser), fast startup, full server capabilities (direct DB access, secrets, full BCL), code never ships to the client.
- **Cons**: every interaction is a **network round-trip** (latency-sensitive; bad on poor connections), the server holds **per-user state** (a "circuit" — memory cost scales with concurrent users), and it **doesn't work offline** or survive a dropped connection gracefully.

Best for internal/line-of-business apps on reliable networks, where low latency to the server and small payloads matter.

---

## Interactive WebAssembly

The component's C# is compiled to run on the **.NET WebAssembly runtime in the browser**. After an initial download of the runtime + your assemblies, interactions run **entirely client-side** — no server round-trip per click.

- **Pros**: works **offline**, no per-interaction latency, minimal server load after download, true client app (PWA-capable).
- **Cons**: **larger initial download** (the .NET runtime + app DLLs — mitigated by AOT, trimming, and caching), slower first interactivity, runs in the browser sandbox (no direct DB/file/secret access — must call APIs ([Ch09](../09-NetworkingAndHttp/README.md))), and your code ships to the client (don't embed secrets).

Best for public-facing apps needing offline support, or to offload work from the server to clients.

---

## Interactive Auto — the best of both

**Auto** uses Interactive Server for the *first* visit (fast startup, no waiting on a download), while the WebAssembly runtime downloads in the background and is cached. On *subsequent* visits (once WASM is cached), the component runs in the browser — offline-capable, no server round-trips.

```razor
@rendermode InteractiveAuto
```

This gives a fast first experience and a low-server-load steady state. The catch: the component must work in **both** environments — so it can't assume direct server-only access (use HTTP APIs / interfaces that work in both). Auto is often the best default for interactive public apps.

---

## Render mode scope and `prerender`

- A render mode can be set **per component** (`<Comp @rendermode="..." />`) or **per component definition** (`@rendermode InteractiveServer`). Children typically inherit the parent's interactive mode.
- **Prerendering** (on by default for interactive modes) renders the component's *initial* HTML on the server for a fast first paint and SEO, *then* makes it interactive. This means **lifecycle methods can run twice** (once during prerender, once when interactive) — a classic gotcha ([07-Lifecycle.md](07-Lifecycle.md)). Disable with `@rendermode="new InteractiveServerRenderMode(prerender: false)"` if double-execution causes problems (e.g., duplicate data fetches).

---

## Choosing a mode (decision guide)

```
Content / read-only / SEO-critical?            → Static SSR
Internal app, reliable network, low latency?   → Interactive Server
Public app, offline, scale to many clients?    → Interactive WebAssembly
Want fast start AND client-side steady state?  → Interactive Auto
Native desktop/mobile shell?                   → Blazor Hybrid ([Ch15](../15-MAUI/README.md))
```

You can **mix** modes in one app: static SSR for the marketing pages, Server for the admin dashboard, WebAssembly for the customer portal. That per-component flexibility is the whole point of the unified model.

---

## Common gotchas

### Lifecycle runs twice (prerendering)

With prerendering on, `OnInitializedAsync` runs during the server prerender *and* again when the component becomes interactive — duplicating data fetches. Cache state across the two phases (`PersistentComponentState`) or disable prerender for that component ([07-Lifecycle.md](07-Lifecycle.md)).

### Server-only code in a WebAssembly/Auto component

A WASM (or Auto) component runs in the browser sandbox — direct DbContext/file/secret access fails. It must call HTTP APIs. Auto components especially must work in *both* environments.

### Underestimating Interactive Server's per-user cost

Each connected user holds a server-side **circuit** (memory + a live WebSocket). High concurrency multiplies server memory and connection count — and a dropped connection interrupts the user. Plan capacity; consider WASM/Auto to offload.

### Shipping secrets to WebAssembly

WASM code and config download to the browser — anyone can read them. Never put API keys/secrets in a WASM app; keep secrets server-side and expose them via authorized APIs.

### Forgetting to opt into interactivity

A component with `@onclick` that "does nothing" usually lacks an interactive render mode — it rendered as static SSR. Add `@rendermode`.

---

## Summary

- A component's **render mode** determines *where its C# runs* and how interactivity is delivered — the central Blazor architecture decision (latency, offline, payload, server load all follow from it).
- **Static SSR** (default): server-rendered HTML, no interactivity — fastest, SEO-friendly; enhanced by **streaming SSR** + enhanced navigation.
- **Interactive Server**: runs server-side over a SignalR **circuit** — tiny download, full server access, but per-interaction latency, per-user server state, no offline.
- **Interactive WebAssembly**: runs in the browser — offline, no round-trips, low server load, but larger download and sandboxed (call APIs, no secrets).
- **Interactive Auto**: Server first, then WASM once cached — fast start + client-side steady state; the component must work in **both** environments.
- **Prerendering** (on by default) runs lifecycle **twice** — a common gotcha; mix modes per component as needed.

→ Next: [03-Components.md](03-Components.md)
