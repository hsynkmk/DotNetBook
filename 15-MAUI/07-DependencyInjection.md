# Dependency Injection

## The same container you already know

MAUI is built on the .NET **Generic Host** ([Ch03 §01](../03-HostingAndDI/01-GenericHost.md)), so it uses the **exact same DI container** as ASP.NET Core and worker services — `Microsoft.Extensions.DependencyInjection`. You register services, pages, and view-models in `MauiProgram`, and the framework constructs them with their dependencies injected. Everything you know about DI lifetimes, captive dependencies, and registration ([Ch03 §02–03](../03-HostingAndDI/02-DependencyInjection.md)) applies directly — MAUI just adds the wrinkle that *pages and view-models* are also resolved from the container.

```csharp
public static MauiApp CreateMauiApp() {
    var builder = MauiApp.CreateBuilder();
    builder.UseMauiApp<App>();

    // Services
    builder.Services.AddSingleton<IDataService, DataService>();
    builder.Services.AddSingleton<IConnectivity>(Connectivity.Current);   // Essentials API

    // ViewModels and Pages
    builder.Services.AddSingleton<MainViewModel>();
    builder.Services.AddSingleton<MainPage>();
    builder.Services.AddTransient<DetailViewModel>();
    builder.Services.AddTransient<DetailPage>();

    return builder.Build();
}
```

---

## Constructor injection for pages and view-models

Register a page and its view-model, give the page a constructor taking the view-model, and the container wires it up — no manual `new`, no hardcoded dependencies:

```csharp
public partial class MainPage : ContentPage {
    public MainPage(MainViewModel vm) {       // resolved + injected by DI
        InitializeComponent();
        BindingContext = vm;
    }
}

public partial class MainViewModel : ObservableObject {
    private readonly IDataService _data;
    public MainViewModel(IDataService data) => _data = data;   // service injected
}
```

When the page is resolved (at startup for the main page, or via Shell navigation — [05-Navigation.md](05-Navigation.md)), DI constructs the view-model, injects its services, and injects it into the page. This is the same constructor-injection model as the rest of .NET ([Ch03 §02](../03-HostingAndDI/02-DependencyInjection.md)) — keep services in constructors, not service-located at point of use.

---

## Lifetimes — and what they mean in an app

The three lifetimes ([Ch03 §03](../03-HostingAndDI/03-Lifetimes.md)) apply, but their *meaning* differs from a web app (there's no per-request scope by default in a client app):

| Lifetime | Behavior in MAUI |
|---|---|
| **Singleton** | one instance for the app's lifetime — for stateful services and pages/VMs that should persist (the main/tab pages, app-wide state) |
| **Transient** | a new instance each resolution — for pages/VMs you want fresh each navigation (detail pages with per-item state) |
| **Scoped** | no ambient request scope by default — behaves transient-like unless you create scopes manually; **use sparingly** in MAUI |

Practical guidance:
- **Singleton** for app services (data, settings, connectivity) and for pages that are navigation roots / always present (a tab's main page) so their state survives.
- **Transient** for detail pages/view-models that should start fresh each time you navigate to them (avoid stale state from a previous item).
- Avoid **Scoped** unless you deliberately manage scopes — there's no automatic per-request scope like ASP.NET Core, so a scoped service often acts like a transient or risks captive-dependency confusion.

---

## Essentials / platform services

MAUI's device APIs (Essentials) are available as injectable interfaces, so you depend on an *interface* (testable, mockable) rather than a static:

```csharp
builder.Services.AddSingleton(Connectivity.Current);     // IConnectivity
builder.Services.AddSingleton(Geolocation.Default);      // IGeolocation
builder.Services.AddSingleton(Preferences.Default);      // IPreferences
builder.Services.AddSingleton(FilePicker.Default);       // IFilePicker
```

```csharp
public class SyncService(IConnectivity connectivity) {
    public bool CanSync => connectivity.NetworkAccess == NetworkAccess.Internet;
}
```

Injecting `IConnectivity`/`IGeolocation`/etc. (instead of calling `Connectivity.Current` statically) makes view-models and services **unit-testable** — you substitute a fake in tests ([Ch17 Testing](../17-Testing/README.md)), exactly as you would mock any dependency.

---

## Resolving manually (when you must)

Constructor injection covers most cases, but sometimes you need to resolve from the container directly (e.g., in a place DI doesn't reach). The current `IServiceProvider` is accessible:

```csharp
var vm = Application.Current?.Handler?.MauiContext?.Services.GetService<DetailViewModel>();
```

Prefer constructor injection; reserve manual resolution (service location) for edge cases — over-using it reintroduces the hidden-dependency problems DI exists to solve ([Ch03 §02](../03-HostingAndDI/02-DependencyInjection.md)).

---

## Common gotchas

### Singleton page/VM holding stale state

A singleton detail page reuses the same instance (and its old state) on every navigation. Use **transient** for pages/VMs that should be fresh per navigation; singleton only for persistent roots.

### Captive dependencies

Injecting a transient into a singleton captures it for the singleton's lifetime ([Ch03 §03](../03-HostingAndDI/03-Lifetimes.md)). The same trap applies in MAUI — mind lifetimes when a long-lived service depends on a shorter-lived one.

### Calling Essentials statically

`Connectivity.Current.NetworkAccess` inside a view-model makes it untestable. Inject `IConnectivity` instead and mock it in tests.

### Over-using manual resolution

Reaching into `Services.GetService<T>()` everywhere is service location — it hides dependencies and defeats DI. Prefer constructor injection; resolve manually only where injection genuinely can't reach.

### Scoped lifetime assumptions

There's no automatic per-request scope in a client app. A `Scoped` service won't behave like ASP.NET Core's per-request scope — don't rely on that semantics; use singleton/transient deliberately.

---

## Summary

- MAUI uses the **same `Microsoft.Extensions.DependencyInjection` container** as the rest of .NET (it's built on the **Generic Host** — [Ch03](../03-HostingAndDI/README.md)); register services, **pages, and view-models** in `MauiProgram`.
- Use **constructor injection** — register a page + its view-model, take the VM in the page's ctor, and DI wires everything (services into VM, VM into page) at resolution/navigation time.
- **Lifetimes**: **singleton** for app services and persistent root pages/VMs; **transient** for detail pages/VMs that should be fresh per navigation; **avoid scoped** (no automatic per-request scope in a client app).
- Inject **Essentials** device APIs as interfaces (`IConnectivity`, `IGeolocation`, …) instead of calling statics — that's what makes view-models **unit-testable**.
- Prefer constructor injection over manual `Services.GetService<T>()` (service location) to keep dependencies explicit.

→ Next: [08-PlatformSpecific.md](08-PlatformSpecific.md)
