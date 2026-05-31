# Chapter 15 — .NET MAUI — Q & A

---

### Q1. What is .NET MAUI?

A framework for building **native** client apps for iOS, Android, macOS, and Windows from a **single C# codebase** — the successor to Xamarin.Forms, on modern .NET. Cross-platform controls map to real native widgets via the **handler** architecture.

---

### Q2. How does a MAUI control become a native control?

Through the **handler** architecture: each cross-platform control (e.g., `Button`) has a per-platform handler that creates and manages the underlying native view (`UIButton` on iOS, `Android.Widget.Button` on Android, …) and keeps it in sync with the control's properties. You program the abstraction; handlers bridge to native at runtime.

---

### Q3. What does the project structure look like?

One multi-targeted project (`net10.0-android`/`-ios`/`-maccatalyst`/`-windows`) with shared source plus a `Platforms/` folder for per-OS entry points and code. `MauiProgram.CreateMauiApp()` bootstraps the app (DI, fonts, config) — analogous to ASP.NET Core's `WebApplication`.

---

### Q4. How is MAUI related to the Generic Host / DI?

MAUI is built on the .NET **Generic Host**, so it uses the same `Microsoft.Extensions.DependencyInjection` container, configuration, and logging as ASP.NET Core. You register services, pages, and view-models in `MauiProgram` and get constructor injection throughout.

---

### Q5. What is XAML and how does it relate to code-behind?

XAML is XML markup describing the UI tree declaratively; each element is a .NET object and each attribute a property/event. A `.xaml` file pairs with a `.xaml.cs` **code-behind** partial class (linked by `x:Class`). Anything in XAML can be written in C#; XAML is a convenience for declarative, tree-shaped UI.

---

### Q6. What are attached properties? Give an example.

Properties defined by one type but **set on another** — typically a child carrying layout data its parent interprets. `Grid.Row="0"` is defined by `Grid` but set on a child `Label`; the `Grid` reads it to position the child. Always written `Owner.Property`.

---

### Q7. What are markup extensions? Name the key ones.

`{curly-brace}` expressions that compute an attribute value at parse time. Key ones: `{Binding path}` (data binding — basis of MVVM), `{StaticResource key}` (resource lookup), `{OnPlatform ...}` (per-platform value), `{AppThemeBinding}` (light/dark), `{x:Static}` (static member).

---

### Q8. Compare the main layouts.

**Grid** (rows/columns with `Auto`/`*`/fixed sizing — the responsive workhorse), **VerticalStackLayout/HorizontalStackLayout** (simple one-direction lines), **FlexLayout** (Flexbox-style, wraps/reflows), **AbsoluteLayout** (explicit coordinates for overlays). Use `*` sizing for responsiveness.

---

### Q9. Why can't you put a CollectionView inside a ScrollView?

`CollectionView`/`ListView` scroll and virtualize themselves; a `ScrollView` (or stack) gives them infinite space, which defeats virtualization and breaks scrolling. Put scrolling list controls in a bounded Grid `*` cell instead.

---

### Q10. Why `CollectionView` over `ListView`?

`CollectionView` is the modern, virtualizing list — more flexible (grids, grouping, pull-to-refresh), better performing, with compiled-binding templates. `ListView`/`TableView` are legacy (reserve `TableView` for settings-style forms).

---

### Q11. What's the difference between Shell and NavigationPage?

**Shell** is a high-level, URI-routed navigation host providing flyout/tabs, a back stack, and deep linking from one `AppShell` (recommended). **`NavigationPage`** is the lower-level push/pop stack. Shell is built on stack navigation under the hood; prefer it for multi-section apps.

---

### Q12. How do you pass data between pages in Shell?

Via **query parameters** in the route (`product/details?id=5`) or a passed dictionary for objects, received with `[QueryProperty]` or `IQueryAttributable.ApplyQueryAttributes`. Prefer passing an **id** and reloading over whole objects (robust to deep links/process restarts).

---

### Q13. What is MVVM and why use it?

Model-View-ViewModel separates the **View** (XAML, binds), **ViewModel** (UI-free C#: bindable properties + commands), and **Model** (domain/services). The View binds to the ViewModel instead of using code-behind logic. Benefits: testable logic (no UI needed), reuse, and clean separation. The ViewModel must never reference UI types.

---

### Q14. What does `INotifyPropertyChanged` do?

It's the interface (with the `PropertyChanged` event) that lets data binding update the UI when a ViewModel property changes. Without raising `PropertyChanged`, the UI won't reflect property updates.

---

### Q15. What does CommunityToolkit.Mvvm generate, and how?

Via **source generators** at compile time (no reflection): `[ObservableProperty]` on a (lowercase) field generates a PascalCase property with `PropertyChanged` notification; `[RelayCommand]` on a method generates an `ICommand` (with `CanExecute`/async support). The base class is `ObservableObject`; the class must be `partial`.

---

### Q16. Why bind a `Command` instead of handling `Clicked`?

A `Command` keeps the action in the testable ViewModel and supports `CanExecute` (auto-enable/disable) and async (`IsRunning`). A `Clicked` event handler lives in code-behind, coupling UI to logic and hurting testability.

---

### Q17. Why `ObservableCollection<T>` for lists?

It raises `CollectionChanged`, so a bound `CollectionView` updates automatically on add/remove. A plain `List<T>` doesn't notify, so the UI would appear static after the initial render.

---

### Q18. How do you write platform-specific code, in order of preference?

1) **Essentials** APIs (cover most device features, no branching); 2) **`OnPlatform`/`OnIdiom`** for per-platform *values* in XAML; 3) **partial classes** in `Platforms/<Platform>/` for non-trivial implementations (compiled only for that target); 4) **`#if ANDROID/IOS/...`** for small inline branches. Customize native rendering via **handler mappers**.

---

### Q19. What is Essentials and why inject it as interfaces?

Essentials is MAUI's cross-platform device API (GPS, sensors, battery, preferences, file picker, connectivity, sharing, permissions). Injecting `IConnectivity`/`IGeolocation`/etc. (instead of calling `Connectivity.Current` statically) makes view-models/services **unit-testable** by substituting fakes.

---

### Q20. What is Blazor Hybrid and how does it differ from web Blazor?

Blazor Hybrid hosts Blazor components in a **`BlazorWebView`** inside a MAUI app, running **in-process on the native .NET runtime** with **full device access and no network** (unlike WASM's sandbox or Server's circuit). It lets you reuse Blazor/web UI to build native apps and **share components (via an RCL) between web and native**.

---

### Q21. When choose MAUI XAML vs Blazor Hybrid vs Blazor web?

MAUI XAML for **native look/feel**; Blazor Hybrid to **reuse Blazor/web components in a native app and share UI with a website**; Blazor web (WASM/Server) for **browser reach, no install**. You can mix native and `BlazorWebView` pages in one MAUI app.

---

### Q22. What are MAUI's biggest performance levers?

**Startup** (defer init, small first-page graph, AOT), **lists** (CollectionView virtualization, lightweight templates, compiled bindings, incremental loading), **images** (downsample to display size + cache — top OOM fix), **layout** (Grid over deep nesting), and **build options** (Release + AOT [mandatory on iOS] + trimming, R2R on Windows). Measure on real devices.

---

### Q23. What lifetime should pages/view-models use?

**Transient** for detail pages/VMs that should be fresh each navigation (avoid stale state); **singleton** for app services and persistent root pages (tab mains, app-wide state). Avoid **scoped** — there's no automatic per-request scope in a client app, so it behaves transient-like and invites confusion.

---

### Q24. Why is trimming risky and how do you handle it?

Trimming removes unused IL to shrink the app, but it can remove members only accessed via **reflection**, causing runtime failures. Test trimmed Release builds, and annotate reflection usage (`DynamicDependency`, trim attributes) or make code trim-safe ([Ch01](../01-Runtime/README.md)).
