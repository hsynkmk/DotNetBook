# Performance

## What costs what

Blazor performance has two faces: **render efficiency** (how much work each UI update does — relevant to all render modes) and **mode-specific costs** (circuit/latency for Server, download/runtime for WebAssembly — [02-RenderModes.md](02-RenderModes.md)). Most performance problems come from rendering too much, too often, or shipping too large a payload. The fixes are targeted: render less (`ShouldRender`, `@key`), render fewer rows (virtualization), and shrink the WASM payload (trimming, AOT, lazy loading).

---

## Reduce unnecessary renders

Every state change re-runs a component's render and diffs it ([01-Overview.md](01-Overview.md)). Two levers reduce wasted rendering:

### `ShouldRender`

Override `ShouldRender()` to skip a re-render when nothing visible changed:

```csharp
private int _lastRendered = -1;
protected override bool ShouldRender() {
    if (_value == _lastRendered) return false;   // nothing changed → skip
    _lastRendered = _value;
    return true;
}
```

Use sparingly and correctly — a wrong `ShouldRender` causes "stale UI" bugs (the component holds outdated content because it refused to re-render). Reserve it for measured hot spots, not blanket optimization.

### `@key` for stable list identity

In dynamic lists, `@key` lets the diff match elements to data across reorders/inserts, so Blazor moves DOM nodes instead of rebuilding them ([03-Components.md](03-Components.md)):

```razor
@foreach (var row in rows) { <Row @key="row.Id" Data="row" /> }
```

Without `@key`, an insert at the top can cause every subsequent row to re-render. Keying by stable id makes list updates O(changes), not O(list).

---

## Virtualize large lists

Rendering thousands of rows creates thousands of DOM elements — slow to render and memory-heavy. The **`<Virtualize>`** component renders only the items currently in the viewport (plus a small buffer), recycling as you scroll:

```razor
<Virtualize Items="largeList" Context="item" ItemSize="40">
    <ItemRow Data="item" />
</Virtualize>

@* Or load on demand from a backend, never holding the whole set in memory: *@
<Virtualize ItemsProvider="LoadItems" Context="item">
    <ItemRow Data="item" />
</Virtualize>
@code {
    async ValueTask<ItemsProviderResult<Item>> LoadItems(ItemsProviderRequest req) {
        var (items, total) = await _api.PageAsync(req.StartIndex, req.Count);
        return new ItemsProviderResult<Item>(items, total);
    }
}
```

`Virtualize` is the single biggest win for long lists/grids — it turns "render 10,000 rows" into "render ~20 visible rows." The `ItemsProvider` form also avoids loading the entire dataset, paging from the backend as the user scrolls.

---

## Avoid expensive work in render

The render path (the markup expressions and `BuildRenderTree`) runs on every render — keep it cheap:

- **Don't** do LINQ queries, sorting, formatting, or allocations inside `@expressions` that run each render. Compute them once (on data change) and store the result in a field.
- **Don't** allocate lambdas/objects in hot render loops unnecessarily.
- Move data fetching to lifecycle methods ([07-Lifecycle.md](07-Lifecycle.md)), not the render body.

```razor
@* ✗ sorts on every render *@
@foreach (var x in items.OrderBy(i => i.Name)) { ... }

@* ✓ sort once when data changes (e.g., in OnParametersSet), render the cached result *@
@foreach (var x in _sorted) { ... }
```

---

## Interactive Server specifics

- **Latency**: every interaction is a round-trip over the circuit. Minimize chatty interactions; debounce rapid events (e.g., search-as-you-type) so you don't flood the circuit.
- **Circuit memory**: each user holds server state; large component state × many users = memory pressure. Keep per-circuit state lean; dispose subscriptions ([07-Lifecycle.md](07-Lifecycle.md)).
- **Render batching**: Blazor batches DOM updates sent over the wire; avoid forcing excessive `StateHasChanged` calls.

---

## WebAssembly specifics

- **Download size**: the .NET runtime + your assemblies download to the browser. Mitigate with **trimming** ([Ch01 / publishing](../19-Deployment/README.md)) to remove unused IL, **compression** (Brotli, usually automatic), and **lazy loading** assemblies (`BlazorWebAssemblyLazyLoad`) so rarely-used features download on demand.
- **AOT compilation**: compiling to WebAssembly **AOT** speeds up CPU-bound execution (at the cost of a larger download) — opt in for compute-heavy apps; the default (IL interpreter) is smaller but slower for hot loops.
- **Caching**: the runtime/assemblies are cached after first load, so subsequent visits are fast — the first-load cost is one-time. **Auto** mode ([02-RenderModes.md](02-RenderModes.md)) hides the first-load latency by starting on the server.

---

## Common gotchas

### Wrong `ShouldRender` → stale UI

Returning `false` when state *did* change leaves outdated content on screen. Use `ShouldRender` only for measured hot spots, with correct change detection.

### No virtualization on big lists

Rendering thousands of rows is slow and memory-heavy. Use `<Virtualize>` (and `ItemsProvider` to page from the backend).

### Expensive LINQ/formatting in the render body

Sorting/filtering/formatting inside `@expressions` runs every render. Precompute on data change, store in a field, render the cached result.

### Chatty interactions on Interactive Server

Every event is a network round-trip; un-debounced keystroke handlers flood the circuit. Debounce/throttle high-frequency events.

### Shipping a huge WASM payload

No trimming/lazy-loading bloats first load. Enable trimming, compression, lazy assembly loading; consider AOT for compute-heavy apps and Auto mode to mask first-load latency.

---

## Summary

- Performance = **render efficiency** (all modes) + **mode-specific costs**; most issues are rendering too much/often or shipping too large a payload.
- Reduce renders with **`ShouldRender`** (carefully — wrong use causes stale UI) and **`@key`** (stable list identity → minimal DOM diffs).
- **`<Virtualize>`** renders only visible rows (with `ItemsProvider` to page from the backend) — the biggest win for long lists/grids.
- Keep the **render body cheap**: precompute sorting/filtering/formatting on data change, not in `@expressions`; fetch data in lifecycle methods.
- **Interactive Server**: minimize round-trips (debounce), keep circuit state lean. **WebAssembly**: shrink download (**trimming**, compression, **lazy loading**), consider **AOT** for compute, rely on caching + **Auto** mode to mask first-load.

→ Next: [11-Auth.md](11-Auth.md)
