# 14–16 — Client-Side: Blazor, MAUI & Desktop

> **Breadth insurance.** For a backend .NET role these rarely open an interview, but not having an
> answer to "what's Blazor Server versus WebAssembly?" reads as a gap. Blazor gets the most space
> because it's the one most likely to come up on a web team.

## ⚡ 30-second answer

**Blazor** builds interactive web UI in **C#** from **components** (`.razor` = markup + `@code`),
using a **render tree + diffing** to apply minimal DOM edits. The central architectural decision
is the **render mode**: **Static SSR** (no interactivity, fastest), **Interactive Server** (C#
runs on the server over a **SignalR circuit** — tiny download, but per-interaction latency and
per-user server state), **Interactive WebAssembly** (C# runs in the browser — offline-capable, no
server state, but a larger download and no direct database access), or **Auto**. **MAUI** builds
native iOS/Android/macOS/Windows apps from one codebase, mapping cross-platform controls to real
native widgets via **handlers**. On desktop, **WPF** is the mature workhorse, **WinUI 3** the
modern Fluent option, **Avalonia** the cross-platform one. **WPF, WinUI, Avalonia and MAUI all
share XAML + data binding + MVVM**, so learning MVVM once covers all four.

---

## Blazor

### Render modes — the decision that matters

| Mode | C# runs | Download | Latency | Offline | Server state |
|---|---|---|---|---|---|
| **Static SSR** (default) | server, once | tiny | n/a | n/a | none |
| **Interactive Server** | **server**, over a SignalR circuit | tiny | **per interaction** | ❌ | **per user** |
| **Interactive WebAssembly** | **browser** (Mono) | **large** (runtime + DLLs) | none after load | ✅ | none |
| **Auto** | Server first, then WASM once cached | — | best of both | ✅ after | transitional |
| **Hybrid** (MAUI Blazor) | native process, WebView UI | n/a | none | ✅ | n/a |

**Interactive Server** keeps a live **circuit** per user: every click is a round-trip, so latency
matters and a dropped connection breaks the UI until it reconnects. It also means **server memory
per connected user**, which is the scaling constraint. And it needs the same scale-out treatment
as SignalR ([09](09-Http-gRPC-SignalR.md)).

**WebAssembly** has no server state and works offline, but ships the runtime plus your assemblies,
and it **cannot touch your database directly** — it calls an API like any other SPA, so everything
in it is public.

### Components

```razor
@page "/products/{Id:int}"

<h3>@_product?.Name</h3>
<button @onclick="() => Delete(_product)">Delete</button>

@code {
    [Parameter] public int Id { get; set; }
    private Product? _product;

    protected override async Task OnParametersSetAsync()      // re-runs when Id changes
        => _product = await Api.GetAsync(Id);
}
```

- Data flows **down** via `[Parameter]` (don't mutate them — the parent owns them) and **up** via
  **`EventCallback`** (which also triggers the parent's re-render). `@bind` for two-way.
- **Lifecycle**: `OnInitialized[Async]` (**once**) → `OnParametersSet[Async]` (**every parameter
  change**) → render → `OnAfterRender[Async]`. Parameter-dependent data goes in
  `OnParametersSetAsync`; one-time setup in `OnInitializedAsync`.
- **`OnAfterRenderAsync` is the only safe place for JS/DOM interop** — the DOM doesn't exist
  before, and it doesn't run during prerendering. Guard with `firstRender`.
- **`EditForm`** + `Input*` components + `<DataAnnotationsValidator />` gives you forms with the
  *same* validation attributes as the server ([04](04-AspNetCore.md)); `EditContext` is the engine
  underneath.
- **JS interop**: `IJSRuntime`, always **async**. Prefer **JS isolation** — load an ES module and
  hold an `IJSObjectReference`, then dispose it — over global `<script>` tags.

### State and performance

The idiomatic shared-state pattern is a **DI state container with a `Changed` event**; components
subscribe, call `StateHasChanged`, and **must unsubscribe in `Dispose`** or they leak.

**Lifetime is subtle**: `AddScoped` means **per circuit (per user)** in Interactive Server and
**per session** in WebAssembly. Per-user state must never be a singleton on Server — it would be
shared across every user.

Performance is mostly "rendering too much, too often, or shipping too large a payload":
**`@key`** for stable list identity, **`<Virtualize>`** for long lists (the biggest single win),
and `ShouldRender` carefully — wrong use gives you stale UI.

---

## MAUI

One C# codebase for **iOS, Android, macOS and Windows**, the successor to Xamarin.Forms.
Cross-platform controls map to **real native widgets** through the **handler** architecture — you
program the abstraction, handlers bridge to native at runtime, so you get native look and feel. A
**single project** multi-targets (`net10.0-android`, `-ios`, `-maccatalyst`, `-windows`) with a
`Platforms/` folder for per-OS code, and `MauiProgram.CreateMauiApp()` is the familiar host
builder — same DI, configuration and logging as everywhere else
([03](03-Hosting-DI-Config.md)).

**Blazor Hybrid** runs Blazor components in a WebView inside the native app — one component model
for web and native.

---

## Desktop

| Option | For | Note |
|---|---|---|
| **WPF** | serious Windows business apps | mature, huge ecosystem, the MVVM workhorse |
| **WinUI 3** | new Windows apps wanting modern Fluent | Windows App SDK |
| **WinForms** | simple internal tools, legacy | fastest to throw together |
| **Avalonia** | cross-platform desktop | XAML, community-driven |

**WPF, WinUI 3 and Avalonia share XAML, data binding and MVVM** — which is why MVVM is the
transferable skill.

---

## MVVM (the one pattern to know across all of these)

**View** (XAML, binds) ← **ViewModel** (UI-free C#: bindable properties + commands) →
**Model** (domain, services). The point is that the ViewModel is **testable without a UI**.

Binding updates require **`INotifyPropertyChanged`**. **CommunityToolkit.Mvvm** generates that
plumbing from **`[ObservableProperty]`** and **`[RelayCommand]`**, so you write the property and
the source generator writes the notification.

```csharp
public partial class OrderViewModel(IOrderService orders) : ObservableObject
{
    [ObservableProperty] private string _customerName = "";      // → CustomerName + notification

    [RelayCommand(CanExecute = nameof(CanSubmit))]
    private async Task SubmitAsync() => await orders.SubmitAsync(…);

    private bool CanSubmit() => !string.IsNullOrWhiteSpace(CustomerName);
}
```

---

## 🪤 Traps & gotchas

- **Choosing Interactive Server without thinking about latency** — every click is a network
  round-trip. Fine on a LAN, painful for a global user base.
- **Forgetting Blazor Server holds per-user state on the server** — memory per connected user, and
  it needs a backplane and sticky sessions to scale out, exactly like SignalR.
- **Assuming Blazor WebAssembly code is private** — it's downloaded to the browser. No secrets, no
  connection strings, no authorization logic you rely on. It calls an API like any SPA.
- **Doing JS interop in `OnInitializedAsync`** — the DOM doesn't exist yet, and during
  prerendering there's no browser at all. `OnAfterRenderAsync` with a `firstRender` guard.
- **Fetching parameter-dependent data in `OnInitializedAsync`** — it runs **once**, so navigating
  from `/products/1` to `/products/2` reuses the component and never refetches.
  `OnParametersSetAsync`.
- **Mutating a `[Parameter]`** — the parent owns it and will overwrite your change on its next
  render.
- **Not unsubscribing from a state container's event in `Dispose`** — the classic Blazor memory
  leak, and on Server it's per user.
- **Registering per-user state as a singleton in Blazor Server** — every user shares it. On
  WebAssembly the same code appears to work, because there's one user.
- **`async void` event handlers** — exceptions are unobservable. `async Task`.
- **Rendering a long list without `<Virtualize>` or `@key`** — thousands of DOM nodes and a full
  diff on every change.
- **`ShouldRender` returning false too eagerly** — a stale UI that's very hard to debug, because
  the state is correct and the screen isn't.
- **Blocking the UI thread** in MAUI/WPF/WinUI — the app freezes. `async`/`await` all the way, and
  marshal back to the UI thread for updates.
- **Business logic in code-behind** — the whole reason MVVM exists is so the logic is testable
  without a UI.
- **Forgetting `INotifyPropertyChanged`** — you set the property, the model is right, and the
  screen never changes. `[ObservableProperty]` removes the whole class of bug.

---

## ❓ Likely questions

**Q: What is Blazor?**
A: A framework for building interactive web UI in C# instead of JavaScript, composed of
components that render to a render tree; state changes produce a new tree which is diffed against
the old, and only the differences are applied to the DOM.

**Q: Blazor Server vs Blazor WebAssembly?**
A: Server runs your C# on the server and streams UI diffs over a SignalR circuit — tiny download,
full server and database access, but every interaction is a round-trip, it holds per-user state on
the server, and it doesn't work offline. WebAssembly downloads a .NET runtime and your assemblies
and runs entirely in the browser — no server state, works offline, but a large initial download,
and it must call an API for data like any SPA.

**Q: When would you pick each?**
A: Server for internal or line-of-business apps with low-latency users, where a small download and
direct data access matter. WebAssembly for public apps, offline capability, or when you can't hold
per-user server state. Auto mode gets the fast first load of Server and switches to WebAssembly
once the runtime is cached.

**Q: What's the scaling constraint on Blazor Server?**
A: Each connected user holds an open SignalR circuit and server-side state, so memory and
connections scale with concurrent users — and multiple instances need a backplane plus sticky
sessions, exactly like SignalR.

**Q: Explain the Blazor component lifecycle.**
A: `OnInitialized[Async]` runs **once** when the component is created; `OnParametersSet[Async]`
runs whenever parameters change, including the first time; then it renders; then
`OnAfterRender[Async]` runs with a `firstRender` flag. Parameter-dependent data loading belongs in
`OnParametersSetAsync`, and JS/DOM interop only in `OnAfterRenderAsync`.

**Q: How do components communicate?**
A: Down via `[Parameter]`, up via `EventCallback` — which also triggers the parent's re-render.
For ambient values down a subtree, cascading values. For unrelated components, a DI state
container with a change event that subscribers listen to.

**Q: What's the biggest Blazor memory leak?**
A: Subscribing to a state container or `NavigationManager` event and not unsubscribing in
`Dispose`. The component is gone but the container still references it — and on Blazor Server that
accumulates per user.

**Q: What is MAUI, and how does it render?**
A: One C# codebase producing native apps for iOS, Android, macOS and Windows. Cross-platform
controls map to **real native widgets** through handlers, so the result looks and behaves native
rather than being a drawn approximation.

**Q: WPF, WinUI 3, WinForms or Avalonia?**
A: WPF for serious Windows business apps — mature, huge ecosystem. WinUI 3 for a new Windows app
wanting modern Fluent design. WinForms for simple internal tools and legacy. Avalonia when you need
the same XAML app on Linux and macOS too.

**Q: What is MVVM and why does it matter?**
A: View (XAML) binds to a ViewModel — plain C# with bindable properties and commands — which uses
the Model. The point is testability: the ViewModel has no UI dependency, so you can unit-test the
behavior without a window. Binding needs `INotifyPropertyChanged`, and CommunityToolkit.Mvvm
source-generates it from `[ObservableProperty]` and `[RelayCommand]`.

**Q: How do you test Blazor components?**
A: **bUnit** — renders components in-memory on a normal test runner, with strongly-typed
parameters, CSS-selector queries, event triggering and fake services injected into `Services`. It
tests component logic, not real-browser behavior, so pair it with a small Playwright suite for
critical journeys ([17](17-Testing.md)).

---

## 🎓 Senior Extra

- **Render mode is per component, not per app** — you can leave most of a page as static SSR and
  make only the interactive island interactive, which is the modern Blazor architecture and keeps
  both payload and server state small.
- **Prerendering runs your component twice** — once on the server for the HTML, once when
  interactivity starts. Anything with side effects in `OnInitializedAsync` happens twice, which is
  a genuinely surprising first encounter.
- **Blazor Server's circuit is a stateful connection in a stateless-app world.** It works against
  the usual scaling advice, which is why Auto mode exists and why WebAssembly is often the better
  fit for anything public.
- **The handler architecture is what separates MAUI from Xamarin.Forms' renderers** — handlers are
  decoupled from the control, so customizing platform behavior no longer means subclassing the
  whole renderer.
- **MVVM is the transferable skill across all four UI stacks**, and CommunityToolkit.Mvvm's source
  generators are the reason it's no longer tedious — worth naming, because "MVVM with a lot of
  boilerplate" is the complaint the toolkit answers.
- **Blazor Hybrid is the interesting middle**: the same components run on the web and in a MAUI
  WebView, so a team can share real UI code across web and native without a JavaScript stack.
- **The DI lifetime story differs by hosting model**, and it's the subtlest cross-cutting trap:
  scoped means per-circuit on Server, per-session in WebAssembly, and something else again in a
  Hybrid app. Any per-user state needs the lifetime checked against the render mode
  ([03](03-Hosting-DI-Config.md)).
- **`IDbContextFactory` is the standard answer for EF Core in Blazor Server**, because components
  are long-lived while a `DbContext` must not be ([05](05-EFCore.md)).

→ Deeper: [`../14-Blazor/`](../14-Blazor/README.md) · [`../15-MAUI/`](../15-MAUI/README.md) ·
[`../16-Desktop/`](../16-Desktop/README.md)
