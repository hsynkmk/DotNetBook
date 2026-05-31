# Component Lifecycle

## The sequence of a component's life

Every Blazor component goes through a defined **lifecycle**: it's created, given parameters, initialized, rendered, possibly re-rendered many times, and finally disposed. `ComponentBase` exposes virtual methods at each stage; overriding the right one — and understanding *when* and *how often* each runs — is essential to fetching data correctly, doing JS interop safely, and avoiding bugs like double-fetches and leaks.

```
Created → SetParametersAsync → OnInitialized[Async] → OnParametersSet[Async] → Render
                                      (once)                (each param change)
   ... user interacts / parent re-renders ...
→ OnParametersSet[Async] → Render → OnAfterRender[Async] ...
→ removed from tree → Dispose / DisposeAsync
```

---

## The lifecycle methods

| Method | When | Use for |
|---|---|---|
| `SetParametersAsync` | params arrive (before others) | rarely overridden; custom param handling |
| `OnInitialized` / `OnInitializedAsync` | **once**, after first params | initial data load, subscriptions |
| `OnParametersSet` / `OnParametersSetAsync` | **every** time params change | reload data when inputs change |
| `ShouldRender` | before each re-render | suppress unnecessary renders ([10-Performance.md](10-Performance.md)) |
| `OnAfterRender` / `OnAfterRenderAsync` | after the DOM updates | **JS interop**, focus, third-party JS |
| `Dispose` / `DisposeAsync` | component removed | unsubscribe, dispose resources |

Each has a sync and `async` form; the async forms let Blazor render an intermediate state while awaiting (e.g., a spinner during `OnInitializedAsync`).

---

## `OnInitialized` vs `OnParametersSet` — the data-fetch distinction

This is the most common point of confusion:

- **`OnInitializedAsync`** runs **once** in the component's lifetime (when first created). Load data that doesn't depend on changing parameters here.
- **`OnParametersSetAsync`** runs **every time parameters change** — including the first time *and* on subsequent parent re-renders / route-parameter changes ([06-Routing.md](06-Routing.md)).

```csharp
[Parameter] public int ProductId { get; set; }
private Product? _product;

// ✗ Bug: navigating /products/1 → /products/2 reuses the component;
//   OnInitialized does NOT run again, so the product never reloads.
protected override async Task OnInitializedAsync() => _product = await Load(ProductId);

// ✓ Reload whenever the parameter changes:
protected override async Task OnParametersSetAsync() => _product = await Load(ProductId);
```

Rule of thumb: if a data load **depends on a parameter**, do it in `OnParametersSetAsync` so it re-runs when that parameter changes. Use `OnInitializedAsync` only for one-time setup independent of parameters.

---

## `OnAfterRender` — the only safe place for DOM/JS interop

`OnAfterRenderAsync` runs **after** the component's HTML has been rendered to the DOM. It's the **only** safe place to call JavaScript that touches DOM elements ([09-JSInterop.md](09-JSInterop.md)) — before this, the elements don't exist yet. It receives `firstRender` so you can do one-time JS setup:

```csharp
protected override async Task OnAfterRenderAsync(bool firstRender) {
    if (firstRender) {
        await JS.InvokeVoidAsync("myChart.init", _canvasRef);  // DOM exists now
    }
}
```

Critically, **`OnAfterRender` does not run during static SSR / prerendering** (there's no live DOM), and JS interop is unavailable then — guard interop to run only when interactive. Also: don't change state unconditionally in `OnAfterRender` (it triggers another render → another `OnAfterRender` → infinite loop); gate with `firstRender` or a condition.

---

## The prerendering double-execution trap

With prerendering on (default for interactive modes — [02-RenderModes.md](02-RenderModes.md)), a component initializes **twice**: once on the server during prerender, then again when it becomes interactive in the browser. So `OnInitializedAsync` (and its data fetch) **runs twice** — a wasteful double-fetch, and possibly inconsistent if the data changed between.

The fix is **`PersistentComponentState`**: persist the data computed during prerender on the server and rehydrate it on the interactive pass, skipping the second fetch:

```csharp
[Inject] PersistentComponentState State { get; set; } = default!;
private PersistingComponentStateSubscription _sub;

protected override async Task OnInitializedAsync() {
    _sub = State.RegisterOnPersisting(() => { State.PersistAsJson("data", _data); return Task.CompletedTask; });
    if (!State.TryTakeFromJson<Data>("data", out var restored))
        _data = await _api.LoadAsync();   // prerender pass: actually fetch
    else
        _data = restored!;                // interactive pass: reuse prerendered data
}
```

This is the canonical pattern for prerendered interactive components that fetch data — avoid the double-fetch and the flicker of data changing between passes.

---

## Disposal — preventing leaks

If a component subscribes to events (`NavigationManager.LocationChanged`, an `IOptionsMonitor.OnChange`, a custom service event), holds timers, or owns JS object references, it **must** clean them up by implementing `IDisposable`/`IAsyncDisposable`:

```razor
@implements IDisposable
@code {
    protected override void OnInitialized() => Service.Changed += OnChanged;
    public void Dispose() => Service.Changed -= OnChanged;   // detach to avoid a leak
}
```

This matters acutely in **Interactive Server**, where components live in a server-side circuit — an undisposed subscription keeps the component (and everything it references) alive for the circuit's lifetime, leaking server memory across many users. For async cleanup (disposing a JS module — [09-JSInterop.md](09-JSInterop.md)), implement `IAsyncDisposable`.

---

## Common gotchas

### Data fetch in `OnInitialized` that depends on a parameter

It won't re-run when the parameter changes (same component reused). Fetch parameter-dependent data in `OnParametersSetAsync`.

### Double-fetch from prerendering

`OnInitializedAsync` runs on both the prerender and interactive passes. Use `PersistentComponentState` to fetch once and rehydrate, or disable prerender for that component.

### JS interop in `OnInitialized`

The DOM doesn't exist during init (and not at all during prerender). Do DOM-touching JS interop in `OnAfterRenderAsync` (guarded by `firstRender`), never in `OnInitialized`.

### Infinite render loop from `OnAfterRender`

Changing state every time in `OnAfterRender` triggers another render → another `OnAfterRender`. Gate state changes with `firstRender` or a condition.

### Not disposing subscriptions

Forgetting to unsubscribe (events, timers, JS refs) leaks components — severe in Interactive Server circuits. Implement `IDisposable`/`IAsyncDisposable` and detach.

---

## Summary

- The lifecycle: `SetParametersAsync` → **`OnInitialized[Async]`** (once) → **`OnParametersSet[Async]`** (every param change) → render → **`OnAfterRender[Async]`** (post-DOM) → **`Dispose`/`DisposeAsync`**.
- Fetch **parameter-dependent** data in `OnParametersSetAsync` (re-runs on change); use `OnInitializedAsync` only for one-time, parameter-independent setup.
- **`OnAfterRenderAsync`** is the only safe place for **DOM/JS interop** (the DOM exists; doesn't run during prerender) — guard with `firstRender` and avoid unconditional state changes (render loop).
- **Prerendering runs init twice** — use **`PersistentComponentState`** to fetch once and rehydrate, avoiding double-fetch/flicker.
- **Dispose** subscriptions/timers/JS refs (`IDisposable`/`IAsyncDisposable`) — undisposed subscriptions leak components, badly in Interactive Server circuits.

→ Next: [08-StateManagement.md](08-StateManagement.md)
