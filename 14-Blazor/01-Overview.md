# Blazor — The Mental Model

## C# in the browser, components everywhere

Blazor is .NET's framework for building **interactive web UIs in C#** instead of JavaScript. You compose your UI from **components** — self-contained units of markup + C# logic — and Blazor handles turning that into HTML, wiring up events, and updating the DOM when state changes. The radical part: the *same component model* runs in several very different environments — on the server over a WebSocket, in the browser via WebAssembly, or inside a native app — and you pick *where* a component runs per-component via **render modes** ([02-RenderModes.md](02-RenderModes.md)).

```razor
@* Counter.razor — a complete, interactive component *@
<h1>Count: @count</h1>
<button @onclick="Increment">+1</button>

@code {
    private int count;
    private void Increment() => count++;   // C#, not JavaScript
}
```

That `@onclick` runs C#. No JavaScript was written. How the click travels from the browser to that method depends on the render mode — but the component code is identical either way.

---

## Components are the unit of everything

A Blazor component is a `.razor` file mixing **Razor markup** (HTML + `@`-expressions) and a **`@code`** block (C# fields, methods, parameters). Components:

- **Nest** — a page is a tree of components, each a tree of more components. The app is one big component tree.
- **Take parameters** — `[Parameter]`-marked properties let a parent pass data down ([03-Components.md](03-Components.md)).
- **Encapsulate state and behavior** — each instance has its own fields, lifecycle ([07-Lifecycle.md](07-Lifecycle.md)), and event handlers ([04-EventHandling.md](04-EventHandling.md)).
- **Re-render on state change** — when a component's state changes, Blazor re-runs its render logic and **diffs** the result against the previous output.

This is the same conceptual model as React/Vue components — but written entirely in C#, sharing types and logic with your backend.

---

## The render tree and diffing

When a component renders, it doesn't produce HTML strings directly. It builds a lightweight **render tree** (a tree of "render tree frames" describing elements, attributes, and child components). On a state change, Blazor builds a *new* render tree and **diffs** it against the previous one, computing the minimal set of DOM changes — then applies just those changes. This is the same virtual-DOM idea behind modern JS frameworks:

```
State changes → component re-renders → new render tree
            → diff against old tree → minimal DOM edits applied
```

The diffing is what makes UI updates efficient: changing one field doesn't rebuild the whole page, only the affected nodes. You rarely touch the render tree directly — you change C# state and call `StateHasChanged` (often implicitly), and Blazor figures out the DOM delta.

---

## How a click becomes a C# call

The flow differs by render mode, but conceptually:

1. The browser fires a DOM event (click).
2. Blazor's JS runtime captures it and routes it to the right component+handler.
3. Your C# handler runs (in the browser for WebAssembly; on the server over a WebSocket for Server).
4. The handler changes state; the component re-renders; the render tree is diffed.
5. The resulting DOM edits are applied in the browser.

For **Blazor Server**, steps 3–4 happen on the server and the *DOM diff* is sent over a SignalR WebSocket ([Ch09 §08](../09-NetworkingAndHttp/08-SignalR.md)). For **WebAssembly**, everything happens in the browser. The component author doesn't write any of this plumbing — it's the framework's job.

---

## Where Blazor fits

| You want | Blazor offers |
|---|---|
| Interactive web UI without writing JS | Component model in C# |
| Share types/validation between client & server | Same C# language and DLLs |
| SEO + fast first paint | Static SSR + streaming, then hydrate |
| Rich offline/client app | WebAssembly (runs .NET in the browser) |
| Native desktop/mobile with web UI | Blazor Hybrid (MAUI/WPF — [Ch15](../15-MAUI/README.md)) |

Blazor isn't always the right choice — a content site needs no interactivity, and a JS-heavy ecosystem app may be better served by a JS framework. But when your team is C#-centric and you want to share logic across client and server, Blazor removes the language split between front and back end.

---

## The unified model (.NET 8+)

Modern Blazor (.NET 8 and later, current in .NET 10) unifies the previously separate "Blazor Server" and "Blazor WebAssembly" hosting models into **one app** that supports **per-component render modes**. A single project can:

- Render most pages as **static server HTML** (fast, SEO-friendly),
- Make specific components **interactive on the server** (low latency to add interactivity),
- Make others **interactive via WebAssembly** (offline-capable, no server round-trip per interaction),
- Or use **Auto** (start on the server, transition to WebAssembly once it's downloaded).

This per-component flexibility ([02-RenderModes.md](02-RenderModes.md)) is the headline change of modern Blazor — you no longer choose one model for the whole app.

---

## Common gotchas

### Assuming Blazor is one thing

"Blazor" is a component model with *multiple* render modes (Server, WebAssembly, Auto, Hybrid, static SSR). Each has very different runtime characteristics (latency, offline, payload size). The render mode — not "Blazor" — determines behavior.

### Expecting WebAssembly to be the default

Modern Blazor renders **static server HTML by default**; interactivity is opt-in per component/page via a render mode. A component that "doesn't respond to clicks" often just lacks an interactive render mode.

### Treating it like classic MVC

Blazor is stateful and component-based, not request/response page rendering. State lives in component instances across interactions (especially Server mode) — a different mental model from MVC/Razor Pages ([Ch04](../04-AspNetCore/README.md)).

### Writing JavaScript out of habit

Most UI logic is C#. Reach for JS interop ([09-JSInterop.md](09-JSInterop.md)) only for browser APIs Blazor doesn't wrap (e.g., a specific JS library) — not for ordinary interactivity.

---

## Summary

- **Blazor** builds interactive web UIs in **C#**, composed from **components** (`.razor` = markup + `@code`) that nest into a tree, take parameters, hold state, and re-render on change.
- Rendering uses a **render tree + diffing**: state changes produce a new tree, diffed against the old, applying minimal DOM edits — efficient updates without manual DOM work.
- The **same component model runs in multiple environments** — server (over a WebSocket), browser (WebAssembly), or native (Hybrid) — chosen per-component via **render modes** ([02-RenderModes.md](02-RenderModes.md)).
- Modern Blazor (.NET 8+) **unifies** the hosting models: static server HTML by default, with **per-component** interactivity (Server / WebAssembly / Auto) opted in where needed.

→ Next: [02-RenderModes.md](02-RenderModes.md)
