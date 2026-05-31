# State Management

## Where does state live?

Every interactive UI has state — the current user, a shopping cart, a form's draft, UI toggles. The question is *where it lives and how components share it*. Blazor offers a spectrum from purely local component state to app-wide stores, and the right choice depends on **scope** (one component vs many) and **lifetime** (one interaction vs the session). Getting this wrong leads to either prop-drilling everything or a tangle of global mutable state.

| Approach | Scope | Use for |
|---|---|---|
| Component fields | one component | local UI state (a toggle, an input draft) |
| Parameters + `EventCallback` | parent ↔ child | data flowing between related components |
| Cascading values | a subtree | ambient context (theme, user, tenant) |
| DI service | app/session | shared state across unrelated components |
| State container / library | app | complex shared state with change notification |

---

## Local component state

The simplest state is a private field in a component, mutated by event handlers ([04-EventHandling.md](04-EventHandling.md)). Blazor re-renders the component after the handler:

```razor
@code {
    private bool expanded;
    void Toggle() => expanded = !expanded;   // local, re-renders this component
}
```

Use local state for anything only that component cares about. Don't reach for a global store for a panel's open/closed flag.

---

## Sharing via a DI service (the state container)

To share state across **unrelated** components (not in a parent/child relationship), register a **state container** service in DI and inject it where needed. The container holds the state and raises an event when it changes; components subscribe and call `StateHasChanged`:

```csharp
public class CartState {
    private readonly List<Item> _items = [];
    public IReadOnlyList<Item> Items => _items;
    public event Action? Changed;
    public void Add(Item i) { _items.Add(i); Changed?.Invoke(); }
}
builder.Services.AddScoped<CartState>();   // scope = per circuit (Server) / per app (WASM)
```

```razor
@inject CartState Cart
@implements IDisposable
<p>Cart: @Cart.Items.Count</p>
@code {
    protected override void OnInitialized() => Cart.Changed += StateHasChanged;
    public void Dispose() => Cart.Changed -= StateHasChanged;   // unsubscribe! ([07-Lifecycle.md])
}
```

This is the idiomatic Blazor pattern for shared state: a DI service + a change event + subscribe/unsubscribe. **Unsubscribe in `Dispose`** or you leak the component ([07-Lifecycle.md](07-Lifecycle.md)).

### Lifetime matters

- **Scoped** in **Interactive Server** = **per circuit** (per user connection) — shared across that user's components, isolated between users.
- **Scoped** in **WebAssembly** = effectively **per app instance** (the whole browser session) — there's one user.
- **Singleton** in Server is shared across **all** users (rarely what you want for user state — and a thread-safety concern). Be deliberate: a "per-user cart" must be **scoped**, not singleton.

This Server-vs-WASM scoping difference is a frequent source of bugs — the same `AddScoped` means "per user" on Server and "per session" on WASM.

---

## Cascading state down a subtree

For state an entire subtree needs (theme, current user — [11-Auth.md](11-Auth.md)), a **cascading value** ([03-Components.md](03-Components.md)) avoids threading it through every parameter. Combine with a container for mutable cascading state:

```razor
<CascadingValue Value="appState">
    <Router ... />
</CascadingValue>
```

Use cascading for genuinely ambient context; for everything else, explicit parameters or an injected service keep data flow clearer.

---

## Persisting state across reconnects and reloads

- **`PersistentComponentState`** ([07-Lifecycle.md](07-Lifecycle.md)) carries state from server prerender to the interactive client, avoiding double-fetch.
- **Browser storage** (`localStorage`/`sessionStorage`, via JS interop — [09-JSInterop.md](09-JSInterop.md), or the `ProtectedBrowserStorage` helper) persists across reloads — for drafts, preferences, tokens (be careful with sensitive data).
- **Server-side** (DB, distributed cache — [Ch06](../06-DataAndCaching/README.md)) for durable state. In **Interactive Server**, in-circuit state is **lost if the circuit drops** (connection lost) — don't keep irreplaceable state only in the circuit; persist important things.

---

## State libraries (Flux/Redux-style)

For large apps with complex, widely-shared state and a need for predictable updates, libraries like **Fluxor** or **Blazor-State** bring the Flux/Redux pattern (a single store, actions, reducers, immutable state, dev-tools time-travel). They add ceremony, so reserve them for genuinely complex state graphs — most apps do fine with DI state containers. The newer **signal/observable** approaches (and `ObservableCollection`-style notifications) are lighter alternatives for reactive state.

---

## Common gotchas

### Not unsubscribing from the state container's event

Subscribing to `Changed` without unsubscribing in `Dispose` leaks the component — severe in Server circuits. Always detach.

### Wrong service lifetime (singleton for per-user state)

A singleton state container in Interactive Server is shared across **all users** (and not thread-safe by default). Per-user state must be **scoped** (per circuit). Remember scoped means "per session" in WASM.

### Relying on in-circuit state surviving a disconnect

Interactive Server state lives in the circuit; a dropped connection can lose it. Persist important state (storage, DB) rather than trusting the circuit.

### Over-engineering with a global store

Reaching for Fluxor/global state for simple local UI flags adds needless complexity. Use local fields/parameters first; escalate to a container, then a library, only as sharing/complexity demands.

### Putting secrets/tokens in browser storage on WASM

`localStorage` is readable by any script in the origin (XSS risk). Avoid storing sensitive tokens client-side; prefer secure, server-managed sessions where possible.

---

## Summary

- Choose state by **scope** and **lifetime**: local fields for one component; parameters + `EventCallback` between related components; **cascading values** for ambient subtree context; a **DI service container** for sharing across unrelated components; a **state library** only for complex graphs.
- The idiomatic shared-state pattern: a **DI state container** with a `Changed` event — components subscribe (call `StateHasChanged`) and **must unsubscribe in `Dispose`** to avoid leaks.
- **Lifetime is subtle**: `AddScoped` = **per circuit/user** in Interactive Server, **per session** in WebAssembly; per-user state must be scoped, never singleton (on Server).
- **Persist** important state — `PersistentComponentState` (prerender→client), browser storage (reloads), or server/DB; **Interactive Server in-circuit state is lost on disconnect**.
- Don't over-engineer (local first) and don't store secrets in browser storage.

→ Next: [09-JSInterop.md](09-JSInterop.md)
