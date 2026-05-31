# Components

## The building blocks

A Blazor component is a `.razor` file that compiles to a C# class deriving from `ComponentBase`. It combines **markup** (HTML + Razor `@`-syntax) with a **`@code`** block (fields, methods, parameters). Components are the unit of composition, reuse, and state ([01-Overview.md](01-Overview.md)). Mastering how data flows *into* a component (parameters, cascading values) and *out* via fragments and callbacks is most of Blazor.

```razor
@* Greeting.razor *@
<p class="greeting">Hello, @Name! You have @Count messages.</p>

@code {
    [Parameter] public string Name { get; set; } = "world";
    [Parameter] public int Count { get; set; }
}
```

```razor
@* Usage from a parent *@
<Greeting Name="Ada" Count="3" />
```

---

## Parameters — passing data down

A `[Parameter]` property is set by the parent as an attribute. Rules that matter:

- Parameters should be **public auto-properties** marked `[Parameter]`.
- They are set **before** lifecycle methods like `OnParametersSet` ([07-Lifecycle.md](07-Lifecycle.md)) and may be set again whenever the parent re-renders.
- **Don't mutate your own parameters** — they're owned by the parent; a parent re-render overwrites them. Copy to a private field if you need a mutable working value.

```csharp
[Parameter] public string Title { get; set; } = "";
[Parameter] public IReadOnlyList<Item> Items { get; set; } = [];
```

### `EventCallback` — passing data up

Children communicate *up* to parents via `EventCallback`/`EventCallback<T>` parameters — a Blazor-aware delegate that also triggers the parent's re-render automatically:

```razor
@* child *@
<button @onclick="() => OnSelected.InvokeAsync(item)">Pick</button>
@code { [Parameter] public EventCallback<Item> OnSelected { get; set; } }

@* parent *@
<ItemRow OnSelected="HandlePick" />
@code { void HandlePick(Item i) => _selected = i; }
```

`EventCallback` is preferred over a raw `Action`/`Func` for component callbacks because invoking it dispatches to the right component and requests a render — raw delegates don't.

---

## Two-way binding with `@bind`

`@bind` wires a value + change handler in one step — the idiomatic way to connect form inputs to fields:

```razor
<input @bind="userName" @bind:event="oninput" />   @* update on each keystroke *@
<p>You typed: @userName</p>
@code { private string userName = ""; }
```

Components can expose **two-way bindable parameters** by pairing a `[Parameter] T Value` with `[Parameter] EventCallback<T> ValueChanged` (the `@bind-Value` convention), so a parent can `@bind-Value="..."` against your component.

---

## Cascading values — passing data down many levels

When data must reach deeply nested components without threading it through every parameter (a theme, the current user, a tenant), use a **`CascadingValue`**: an ancestor provides it, any descendant consumes it via `[CascadingParameter]`:

```razor
@* ancestor *@
<CascadingValue Value="theme">
    <App />
</CascadingValue>

@* any descendant, however deep *@
@code { [CascadingParameter] public Theme Theme { get; set; } = default!; }
```

Use **named** cascading values (`<CascadingValue Name="...">`) when several of the same type flow. Cascading values are great for cross-cutting context but overuse hurts clarity and can trigger broad re-renders — prefer explicit parameters for ordinary data, cascading for genuinely ambient context. (`AuthenticationState` is delivered this way — [11-Auth.md](11-Auth.md).)

---

## RenderFragments — passing markup as a parameter

A `RenderFragment` is a chunk of UI you pass *into* a component — the basis of layout/container components. The special `ChildContent` parameter captures the markup placed between a component's tags:

```razor
@* Card.razor *@
<div class="card">
    <div class="card-header">@Header</div>
    <div class="card-body">@ChildContent</div>
</div>
@code {
    [Parameter] public RenderFragment? Header { get; set; }
    [Parameter] public RenderFragment? ChildContent { get; set; }
}
```

```razor
@* Usage — ChildContent is everything between the tags *@
<Card>
    <Header>Title</Header>
    <p>Body content goes here.</p>
</Card>
```

**Templated components** generalize this with `RenderFragment<T>` — a fragment parameterized by data (e.g., a generic `Grid<TItem>` that lets the caller template each row). This is how reusable, content-agnostic UI components (grids, lists, modals) are built.

---

## Component lifecycle (preview)

Components have a lifecycle — initialization, parameter set, render, dispose — covered fully in [07-Lifecycle.md](07-Lifecycle.md). The key methods: `OnInitialized[Async]` (once, when created), `OnParametersSet[Async]` (whenever parameters change), `OnAfterRender[Async]` (after the DOM updates — the only place JS interop with the DOM is safe). Components implementing `IDisposable`/`IAsyncDisposable` get cleaned up when removed.

---

## `@key` — stable identity in lists

When rendering a list, give each item a `@key` so Blazor's diffing matches elements to data correctly across reorders/insertions — preventing state from "leaking" between rows:

```razor
@foreach (var item in items)
{
    <ItemRow @key="item.Id" Item="item" />
}
```

Without `@key`, reordering a list can cause Blazor to reuse the wrong component instances (and their state) for the wrong data. Always key dynamic lists by a stable identifier.

---

## Common gotchas

### Mutating a parameter

Parameters are owned by the parent; mutating one is overwritten on the next parent render. Copy to a private field for mutable working state.

### Using `Action`/`Func` instead of `EventCallback`

Raw delegates as callbacks don't trigger the parent's re-render and don't dispatch to the right component. Use `EventCallback`/`EventCallback<T>` for component events.

### Missing `@key` in dynamic lists

Without `@key`, reordering/inserting list items can attach the wrong component state to the wrong item. Key by a stable id.

### Overusing cascading values

Cascading everything makes data flow opaque and can cause wide re-renders. Use explicit parameters for normal data; reserve cascading for ambient context (theme, user, tenant).

### Expecting parameters to be set in the constructor

Parameters aren't available in the constructor — they're set after construction, before `OnParametersSet`. Read them in lifecycle methods, not the ctor.

---

## Summary

- A **component** (`.razor`) = markup + `@code`, compiling to a `ComponentBase` class; components nest into a tree and are the unit of reuse and state.
- Data flows **down** via `[Parameter]` (don't mutate them — the parent owns them) and **up** via `EventCallback`/`EventCallback<T>` (which also triggers the parent's re-render — prefer over raw delegates).
- **`@bind`** gives two-way binding; expose `Value`/`ValueChanged` for `@bind-Value`-able components.
- **Cascading values** (`CascadingValue`/`[CascadingParameter]`) pass ambient context to deep descendants without threading parameters — use sparingly.
- **`RenderFragment`** passes markup into a component (`ChildContent`); **`RenderFragment<T>`** builds templated/generic components (grids, lists).
- Use **`@key`** on dynamic lists for stable identity; parameters are set after construction, not in the ctor.

→ Next: [04-EventHandling.md](04-EventHandling.md)
