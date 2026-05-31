# Chapter 14 — Blazor — Q & A

---

### Q1. What is Blazor, in one sentence?

A .NET framework for building interactive web UIs in **C#** (not JavaScript), composing the UI from **components** (`.razor` = markup + `@code`) that run in multiple environments (server, browser via WebAssembly, native) chosen per-component via render modes.

---

### Q2. What is a component and how does rendering work?

A `.razor` file compiling to a `ComponentBase` class. On a state change it builds a new **render tree**, **diffs** it against the previous one, and applies the **minimal DOM edits** — the virtual-DOM idea. You change C# state; Blazor computes the DOM delta.

---

### Q3. Name the render modes and their trade-offs.

**Static SSR** (default): server HTML, no interactivity — fastest, SEO. **Interactive Server**: runs server-side over a SignalR circuit — tiny download, full server access, but per-interaction latency and per-user server state, no offline. **Interactive WebAssembly**: runs in the browser — offline, no round-trips, but larger download and sandboxed. **Interactive Auto**: Server first, then WASM once cached — fast start + client steady state.

---

### Q4. What changed about hosting models in modern Blazor (.NET 8+)?

The previously separate Blazor Server and Blazor WebAssembly are unified into **one app** with **per-component render modes**. Default is static SSR; you opt specific components/pages into Server, WebAssembly, or Auto interactivity, and can mix modes in one app.

---

### Q5. How does data flow into and out of a component?

**In**: `[Parameter]` properties (set by the parent; don't mutate them) and `[CascadingParameter]` for ambient context. **Out**: `EventCallback`/`EventCallback<T>` — which dispatch to the parent and trigger its re-render (unlike raw `Action`/`Func`).

---

### Q6. Why `EventCallback` instead of `Action` for component events?

Invoking an `EventCallback` dispatches to the correct (parent) component and automatically requests its re-render. A raw delegate does neither, so the parent's UI won't update.

---

### Q7. What's `@key` for?

Stable identity in dynamic lists. It lets the render-tree diff match elements to data across reorders/inserts, so Blazor moves DOM nodes instead of rebuilding them — preventing state from attaching to the wrong item and making list updates O(changes).

---

### Q8. What is a `RenderFragment` / `RenderFragment<T>`?

`RenderFragment` is UI passed *into* a component (the `ChildContent` between its tags) — the basis of layout/container components. `RenderFragment<T>` is a fragment parameterized by data — the basis of **templated/generic** components (grids, lists).

---

### Q9. Distinguish `OnInitializedAsync` from `OnParametersSetAsync`.

`OnInitializedAsync` runs **once** (component created). `OnParametersSetAsync` runs **every time parameters change** (including first). Fetch **parameter-dependent** data in `OnParametersSetAsync` so it reloads on change — a classic bug is fetching in `OnInitialized` and the data never refreshing when a route parameter changes.

---

### Q10. Where must DOM/JS interop happen, and why?

In **`OnAfterRenderAsync`** — only then does the rendered DOM exist. It also doesn't run during prerender/SSR (no live DOM/JS runtime), so it's the safe place. Doing interop in `OnInitialized` fails.

---

### Q11. What is the prerendering double-execution trap and its fix?

With prerendering on, a component initializes **twice** (server prerender + interactive), so `OnInitializedAsync` data fetches run twice. Fix with **`PersistentComponentState`**: persist the prerendered data and rehydrate on the interactive pass, skipping the second fetch.

---

### Q12. Why must components dispose subscriptions?

Subscriptions (events, timers, JS refs) keep the component alive if not detached. In **Interactive Server**, the component lives in a server-side circuit, so an undisposed subscription leaks server memory across users. Implement `IDisposable`/`IAsyncDisposable` and unsubscribe.

---

### Q13. How do you share state across unrelated components?

A **DI state container** service holding the state with a `Changed` event; components inject it, subscribe (call `StateHasChanged`), and **unsubscribe in `Dispose`**. For ambient context use cascading values; for complex graphs, a state library (Fluxor).

---

### Q14. Why is service lifetime subtle in Blazor?

`AddScoped` means **per circuit (per user)** in Interactive Server but **per app/session** in WebAssembly. Per-user state must be **scoped**, never singleton on Server (a singleton is shared across all users and not thread-safe). The same registration behaves differently per render mode.

---

### Q15. What happens to Interactive Server state on a dropped connection?

In-circuit state lives on the server; a dropped connection can lose it. Don't keep irreplaceable state only in the circuit — persist important state (browser storage, DB, distributed cache).

---

### Q16. How does JS interop work both directions?

C#→JS: inject `IJSRuntime`, call `InvokeVoidAsync`/`InvokeAsync<T>` (always async; crosses the wire in Server). JS→C#: `[JSInvokable]` methods, invoked via a `DotNetObjectReference` (instance) or `DotNet.invokeMethodAsync` (static). Prefer **JS isolation** (ES modules via `import` → `IJSObjectReference`); dispose references.

---

### Q17. What is `EditForm` and how does validation work?

A model-bound form component that tracks edit state and raises `OnValidSubmit`/`OnInvalidSubmit`. Validation: add `<DataAnnotationsValidator />` to enforce model attributes, plus `<ValidationMessage>`/`<ValidationSummary>`. Built on an `EditContext`; reuse the same model class server-side. Client validation is UX — always re-validate on the server.

---

### Q18. How do route parameters and constraints work?

`@page "/products/{Id:int}"` binds `{Id}` to a `[Parameter] int Id`, with a type **constraint** (`:int`) so non-matching URLs fall through. Supports optional (`{Id:int?}`) and catch-all (`{*slug}`) forms; query strings bind via `[SupplyParameterFromQuery]`.

---

### Q19. What does `NavigationManager` provide?

Programmatic navigation (`NavigateTo`, with `forceLoad`/`replace`), the current/base URI, the `LocationChanged` event (unsubscribe!), and navigation interception (`RegisterLocationChangingHandler`/`NavigationLock`) for unsaved-changes guards. Use `NavLink` for active-aware links.

---

### Q20. How is the current user surfaced to components?

As a cascading **`Task<AuthenticationState>`** (wrapping a `ClaimsPrincipal`) from `AuthenticationStateProvider`, enabled by `CascadingAuthenticationState`. Use `AuthorizeView` (Roles/Policy) for conditional UI and `[Authorize]` + `AuthorizeRouteView` for pages.

---

### Q21. Is `AuthorizeView` a security boundary?

No — it only shows/hides UI. A user can still call the underlying API. Real authorization must be enforced **server-side** where the operation runs ([Ch10](../10-Identity/README.md)); component gating is UX.

---

### Q22. How do you unit-test a component?

**bUnit**: `TestContext.RenderComponent<T>(...)` renders in-memory; query with CSS selectors, assert with `MarkupMatches` (semantic), trigger events (`Click`/`Change`), register fake services in `Services`, use `WaitForState`/`WaitForAssertion` for async lifecycle, and mock `IJSRuntime` via bUnit's `JSInterop`. Use Playwright for real-browser E2E.

---

### Q23. When should you NOT use Blazor (or a given mode)?

A static content site needs no interactive runtime (plain SSR/HTML suffices). A JS-ecosystem-heavy app may fit a JS framework better. For modes: avoid Interactive Server for high-latency/offline scenarios or huge concurrency (circuit memory); avoid WebAssembly when first-load size is critical or you need server-only resources directly.

---

### Q24. Why is `StateHasChanged` usually unnecessary, and when is it needed?

Blazor re-renders automatically after event handlers and `EventCallback` invocations. You call `StateHasChanged()` only for **out-of-band** state changes (timers, background/service events), wrapping in `InvokeAsync` when off the renderer's synchronization context.
