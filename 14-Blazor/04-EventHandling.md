# Event Handling

## Wiring DOM events to C#

Blazor lets you attach C# methods to DOM events with `@on{event}` directives — `@onclick`, `@oninput`, `@onchange`, `@onkeydown`, `@onsubmit`, and so on. The handler runs C# (in the browser for WebAssembly, on the server over the circuit for Interactive Server — [02-RenderModes.md](02-RenderModes.md)), and **after the handler returns, the component automatically re-renders** — you usually don't call `StateHasChanged` yourself.

```razor
<button @onclick="Increment">Count: @count</button>

@code {
    private int count;
    private void Increment() => count++;   // re-render is automatic after this returns
}
```

---

## Event arguments

Handlers can accept the strongly-typed event-args object for that event (`MouseEventArgs`, `KeyboardEventArgs`, `ChangeEventArgs`, etc.):

```razor
<input @onkeydown="OnKey" />
<div @onclick="OnClick">Click me</div>

@code {
    void OnKey(KeyboardEventArgs e) { if (e.Key == "Enter") Submit(); }
    void OnClick(MouseEventArgs e) => Console.WriteLine($"({e.ClientX}, {e.ClientY})");
}
```

Blazor maps the DOM event's properties into the args object, so you read `e.Key`, `e.CtrlKey`, `e.ClientX`, etc. in C# — no JavaScript needed.

---

## Async handlers

Event handlers can be `async Task` — the natural shape when the handler does I/O (a DB call, an HTTP request). Blazor awaits it and **re-renders when it completes**; it also re-renders *before* the await if the synchronous part changed state, giving you a chance to show a loading state:

```razor
<button @onclick="LoadAsync" disabled="@busy">Load</button>
@if (busy) { <span>Loading…</span> }

@code {
    bool busy;
    async Task LoadAsync() {
        busy = true;                       // first render: show loading (Blazor re-renders at the await)
        data = await _api.GetDataAsync();  // await I/O
        busy = false;                      // second render after completion
    }
}
```

**Return `Task`, not `void`** — an `async void` handler can't be awaited, so exceptions escape unhandled and Blazor can't track completion. Always use `async Task` for async handlers (the general async rule — [Ch08 / CSharpBook async](../08-BackgroundProcessing/README.md)).

---

## Lambda handlers and passing arguments

To pass an argument (e.g., the item in a loop), use a lambda:

```razor
@foreach (var item in items)
{
    <li @key="item.Id">
        @item.Name
        <button @onclick="() => Delete(item)">Delete</button>
    </li>
}
@code { void Delete(Item i) => items.Remove(i); }
```

Lambdas in loops are convenient but allocate a delegate per render per item; for very hot/large lists this can matter ([10-Performance.md](10-Performance.md)). For most UIs it's fine — favor clarity.

---

## `EventCallback` for component events

DOM events use `@on{event}`; **component-to-parent** events use `EventCallback`/`EventCallback<T>` parameters ([03-Components.md](03-Components.md)). The key difference: invoking an `EventCallback` dispatches to the **parent's** component and triggers the **parent's** re-render automatically — a raw `Action`/`Func` does neither:

```razor
@* child *@
<button @onclick="() => OnDeleted.InvokeAsync(Item.Id)">Delete</button>
@code {
    [Parameter] public Item Item { get; set; } = default!;
    [Parameter] public EventCallback<int> OnDeleted { get; set; }
}
```

Use `EventCallback` for any event a component raises to its parent; use `@onclick` etc. for DOM events on elements.

---

## Preventing default and stopping propagation

DOM event modifiers are set via directive attributes:

```razor
<a href="/x" @onclick="Handle" @onclick:preventDefault>No navigation</a>
<div @onclick="Outer">
    <button @onclick="Inner" @onclick:stopPropagation>Stops here</button>
</div>
```

`@on{event}:preventDefault` and `@on{event}:stopPropagation` map to the JS `preventDefault()`/`stopPropagation()`. There's also `@on{event}:preventDefault="false"` to set it conditionally.

---

## `StateHasChanged` — when you *do* need it

Blazor re-renders automatically after event handlers and `EventCallback` invocations. You only call `StateHasChanged()` manually when state changes **outside** the normal event flow — e.g., from a timer callback, a background event, a subscription, or a non-UI thread:

```csharp
// State changed from a server-side event / timer, not a UI event:
_timer = new Timer(_ => InvokeAsync(() => { _value++; StateHasChanged(); }), null, 0, 1000);
```

Note `InvokeAsync` — when updating state from a thread that isn't the renderer's synchronization context (a `Timer`, a background event), you must marshal back onto it via `InvokeAsync(...)` before calling `StateHasChanged`, or you risk a threading error. Inside ordinary event handlers, you're already on the right context.

---

## Common gotchas

### `async void` handlers

An `async void` handler can't be awaited — exceptions escape and Blazor can't track completion or re-render reliably. Use `async Task`.

### Calling `StateHasChanged` from the wrong thread

Updating UI state from a `Timer`/background callback without `InvokeAsync` throws (you're off the renderer's sync context). Wrap in `InvokeAsync(() => { ...; StateHasChanged(); })`.

### Forgetting that re-render is automatic

After an event handler (or `EventCallback`), Blazor re-renders for you. Calling `StateHasChanged` everywhere is unnecessary noise — reserve it for out-of-band state changes.

### Using `Action` instead of `EventCallback` for component events

A raw delegate doesn't dispatch to the parent component or trigger its re-render. Use `EventCallback`/`EventCallback<T>` for events a component raises.

### Heavy work blocking the UI (Interactive Server)

In Server mode, a long synchronous handler blocks the circuit and freezes that user's UI. Make I/O `async`, and offload CPU-bound work appropriately.

---

## Summary

- Attach C# to DOM events with **`@on{event}`** (`@onclick`, `@oninput`, …); handlers can take strongly-typed **event args** (`MouseEventArgs`, `KeyboardEventArgs`, …), and Blazor **re-renders automatically** after the handler.
- Use **`async Task`** handlers for I/O (never `async void`); Blazor re-renders at the await and on completion — enabling loading states.
- Pass arguments with **lambda handlers** (`() => Delete(item)`); use **`EventCallback`/`EventCallback<T>`** for component→parent events (auto-dispatch + parent re-render — unlike raw delegates).
- Control DOM behavior with **`:preventDefault`/`:stopPropagation`** modifiers.
- Call **`StateHasChanged()`** only for **out-of-band** state changes (timers, background events), and marshal onto the renderer with **`InvokeAsync`** when off its sync context.

→ Next: [05-Forms.md](05-Forms.md)
