# JavaScript Interop

## When C# isn't enough

Blazor lets you write UI in C#, but the browser's APIs are JavaScript — and some things have no .NET equivalent: a charting library, a maps SDK, `localStorage`, the clipboard, focus management, a date picker widget. **JS interop** bridges the gap: you call JavaScript from C# (`IJSRuntime`) and call C# from JavaScript. Use it as a targeted escape hatch — most UI logic stays in C# ([01-Overview.md](01-Overview.md)); reach for JS only for browser APIs Blazor doesn't wrap.

```razor
@inject IJSRuntime JS

@code {
    async Task Copy(string text) => await JS.InvokeVoidAsync("navigator.clipboard.writeText", text);
    async Task<int> Width() => await JS.InvokeAsync<int>("getWindowWidth");
}
```

---

## Calling JavaScript from C#

`IJSRuntime` (injected) has two core methods:

- **`InvokeVoidAsync(identifier, args...)`** — call a JS function that returns nothing.
- **`InvokeAsync<T>(identifier, args...)`** — call a JS function and marshal its return value to `T`.

The `identifier` is a path to a function on the JS global scope (`window`), and arguments are JSON-serialized across the boundary:

```csharp
await JS.InvokeVoidAsync("console.log", "from C#", someObject);
var result = await JS.InvokeAsync<string>("localStorage.getItem", "key");
```

**It's async for a reason**: in Interactive Server, the call crosses the network (server → browser over the circuit), so it *must* be awaited. Even in WebAssembly it's async by default. (WASM offers `IJSInProcessRuntime` for rare synchronous calls, but prefer async.)

---

## JS isolation — modules over global scripts

The modern, recommended approach is **JS isolation**: put your JS in an ES module (`.js` file) and load it on demand with `import`, rather than polluting the global `window` scope with `<script>` tags:

```javascript
// wwwroot/js/chart.js
export function init(element, data) { /* draw a chart into element */ }
export function destroy(element) { /* cleanup */ }
```

```csharp
private IJSObjectReference? _module;

protected override async Task OnAfterRenderAsync(bool firstRender) {
    if (firstRender) {
        _module = await JS.InvokeAsync<IJSObjectReference>(
            "import", "./js/chart.js");           // load the module
        await _module.InvokeVoidAsync("init", _ref, _data);
    }
}

public async ValueTask DisposeAsync() {            // dispose the module reference
    if (_module is not null) await _module.DisposeAsync();
}
```

Isolation keeps each component's JS scoped to a module (no global namespace collisions), loads it only when needed, and ties its lifetime to the component via `IJSObjectReference` (dispose it in `DisposeAsync` — [07-Lifecycle.md](07-Lifecycle.md)).

---

## Element references

To hand a specific DOM element to JS (e.g., "draw the chart *here*"), capture an **`ElementReference`** with `@ref` and pass it across:

```razor
<canvas @ref="_canvas"></canvas>
@code {
    private ElementReference _canvas;
    // pass _canvas to JS in OnAfterRenderAsync (the element exists only after render)
}
```

An `ElementReference` is an opaque handle, only valid **after the element has rendered** — so element-targeting interop belongs in `OnAfterRenderAsync` ([07-Lifecycle.md](07-Lifecycle.md)), never in `OnInitialized`.

---

## Calling C# from JavaScript

JS can call back into .NET, enabling JS events/callbacks to invoke your C# code:

- **Instance methods** — wrap a component (or object) in a **`DotNetObjectReference`** and pass it to JS; JS calls `[JSInvokable]`-marked methods on it:

```csharp
[JSInvokable] public void OnResized(int width) { _width = width; StateHasChanged(); }

protected override async Task OnAfterRenderAsync(bool first) {
    if (first) {
        var dotNetRef = DotNetObjectReference.Create(this);
        await JS.InvokeVoidAsync("registerResizeCallback", dotNetRef);
        // dispose dotNetRef in DisposeAsync
    }
}
```

```javascript
export function registerResizeCallback(dotNetRef) {
    window.addEventListener("resize", () => dotNetRef.invokeMethodAsync("OnResized", window.innerWidth));
}
```

- **Static methods** — `[JSInvokable]` static methods are called via `DotNet.invokeMethodAsync("AssemblyName", "MethodName", args)`.

**Dispose the `DotNetObjectReference`** when done, or it (and the component it wraps) leaks.

---

## Prerendering and SSR caveat

JS interop **cannot run during prerendering / static SSR** — there's no live JS runtime/DOM on the server pass ([02-RenderModes.md](02-RenderModes.md), [07-Lifecycle.md](07-Lifecycle.md)). Calling `IJSRuntime` during `OnInitializedAsync` of a prerendered component throws. The rule: do JS interop in **`OnAfterRenderAsync`** (which doesn't run during prerender) or guard it to run only once interactive. This is the single most common JS interop error.

---

## Common gotchas

### JS interop during prerender / `OnInitialized`

There's no JS runtime during prerendering and the DOM doesn't exist during init. Do interop in `OnAfterRenderAsync(firstRender)`.

### Not disposing `IJSObjectReference` / `DotNetObjectReference`

Module references and .NET object references leak (and keep components alive) if not disposed. Implement `IAsyncDisposable`/`IDisposable` and dispose them.

### Using global `<script>` instead of module isolation

Global scripts pollute `window` and risk collisions. Prefer ES module `import` with `IJSObjectReference` (JS isolation).

### Expecting synchronous interop everywhere

In Interactive Server, interop crosses the network — it's inherently async. Always `await`; don't assume `IJSInProcessRuntime` (WASM-only) semantics.

### Passing large/complex objects across the boundary

Everything is JSON-serialized over the interop boundary (and over the wire in Server). Large payloads are slow; pass minimal data, or use byte streams / `IJSStreamReference` for big transfers.

---

## Summary

- **JS interop** is the escape hatch for browser APIs/libraries with no .NET equivalent — call JS from C# via **`IJSRuntime`** (`InvokeVoidAsync`/`InvokeAsync<T>`, always **async**) and C# from JS via `[JSInvokable]`.
- Prefer **JS isolation**: load component JS as an **ES module** (`import` → `IJSObjectReference`) instead of global `<script>` tags; **dispose** the module reference.
- Pass DOM elements with **`ElementReference`** (`@ref`); element-targeting interop must run in **`OnAfterRenderAsync`** (the element exists only after render).
- Call C# from JS using **`DotNetObjectReference`** (instance) or static `[JSInvokable]`; **dispose** the object reference to avoid leaks.
- **No interop during prerender/SSR** — do it in `OnAfterRenderAsync`/once interactive; keep marshaled payloads small (everything is JSON-serialized, and crosses the wire in Server).

→ Next: [10-Performance.md](10-Performance.md)
