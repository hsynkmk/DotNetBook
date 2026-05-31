# Blazor Hybrid

## Web UI in a native app

**Blazor Hybrid** hosts Blazor components ([Ch14](../14-Blazor/README.md)) inside a native MAUI app, rendered in an embedded **`BlazorWebView`**. Unlike Blazor WebAssembly (which runs in a browser sandbox) or Blazor Server (which runs over a network), Hybrid runs your Blazor components **on the native .NET runtime, in-process, with full native-device access** — and displays their HTML in a local web view. It's the way to reuse Blazor/web UI skills and components to build native desktop/mobile apps, sharing UI across web and native.

```
MAUI app (native shell, full device access)
   └── BlazorWebView (an embedded web view control)
          └── Blazor components (.razor) running on the native .NET runtime
                 ↕ JS interop to the web view's DOM; direct C# to native/device APIs
```

---

## How it differs from web Blazor

| | Blazor WebAssembly | Blazor Server | **Blazor Hybrid** |
|---|---|---|---|
| Where C# runs | browser (WASM) | server | **native device (in-process)** |
| Device access | sandboxed (APIs only) | server-side | **full native** (files, sensors, DI services) |
| Network needed | after download | always (circuit) | **none** (local) |
| UI rendering | browser | browser | **local web view** |
| Distribution | website | website | **app store / installer** |

The headline: Hybrid components run with **no browser sandbox and no network** — they call native MAUI/Essentials APIs and your DI services directly, while still being authored as ordinary Razor components. You get web UI productivity *and* native capabilities.

---

## Setting it up

A MAUI Blazor app adds a `BlazorWebView` to a page and registers Blazor services in `MauiProgram`:

```csharp
// MauiProgram.cs
builder.Services.AddMauiBlazorWebView();
#if DEBUG
builder.Services.AddBlazorWebViewDeveloperTools();
#endif
builder.Services.AddSingleton<IDataService, DataService>();   // shared with components
```

```xml
<!-- MainPage.xaml -->
<BlazorWebView HostPage="wwwroot/index.html">
    <BlazorWebView.RootComponents>
        <RootComponent Selector="#app" ComponentType="{x:Type local:Routes}" />
    </BlazorWebView.RootComponents>
</BlazorWebView>
```

The `wwwroot/index.html` is the host page; `RootComponents` mounts your Blazor root component into a DOM selector. From there, it's ordinary Blazor — components, routing, DI ([Ch14](../14-Blazor/README.md)) — but running natively.

---

## Sharing UI between web and native

The big win: a **Razor Class Library (RCL)** of components can be shared between a Blazor web app and a MAUI Blazor Hybrid app. The same `.razor` components render in the browser (web) *and* in the native web view (Hybrid), so you write your UI once and ship it both as a website and as native desktop/mobile apps:

```
Shared.RazorComponents (RCL)  ←── used by ──┬── BlazorWebApp (browser)
                                            └── MauiBlazorApp (native, Hybrid)
```

This is the strongest reason to choose Hybrid: maximal UI reuse across web and native, with the native app gaining device access the web version can't have.

---

## Accessing native features from components

Because Hybrid runs in-process on the native runtime, a Blazor component can inject and call MAUI/Essentials services directly ([07-DependencyInjection.md](07-DependencyInjection.md)) — no JS interop bridge needed for device features:

```razor
@inject IGeolocation Geolocation
@inject IConnectivity Connectivity

<button @onclick="GetLocation">Where am I?</button>
@code {
    string? _coords;
    async Task GetLocation() {
        var loc = await Geolocation.GetLocationAsync();   // real GPS, native
        _coords = $"{loc?.Latitude}, {loc?.Longitude}";
    }
}
```

This is something neither Blazor WebAssembly nor Server can do directly — the component reaches native device APIs as plain C#.

---

## When to choose Hybrid (vs MAUI XAML, vs web)

| You want | Choose |
|---|---|
| Native look/feel, platform-idiomatic UI | MAUI XAML ([02-XAML.md](02-XAML.md)) |
| Reuse Blazor/web components in a native app; share UI with a website | **Blazor Hybrid** |
| Browser reach, no install | Blazor web (WASM/Server — [Ch14](../14-Blazor/README.md)) |

Trade-offs of Hybrid: the UI is HTML/CSS in a web view (so it looks like *your* web design, not necessarily the platform's native widgets), and you carry web-rendering overhead. But you gain web-UI velocity, CSS styling, and cross-web/native code sharing. You can even **mix** — native MAUI pages alongside `BlazorWebView` pages in one app.

---

## Common gotchas

### Expecting native-widget look

Hybrid renders HTML/CSS in a web view — it looks like your web styling, not native platform controls. If platform-native UX is the priority, use MAUI XAML instead.

### Treating it like Blazor WebAssembly

Hybrid runs **in-process on the native runtime** with full device access and no sandbox/network — don't architect around WASM limitations (it's not sandboxed) or Server's network circuit (there's none).

### Forgetting `AddMauiBlazorWebView()`

The `BlazorWebView` won't work without registering Blazor services in `MauiProgram`. Add it (and developer tools in DEBUG).

### Not sharing UI via an RCL

Copy-pasting components between web and Hybrid apps defeats the main benefit. Put shared components in a **Razor Class Library** used by both.

### JS interop expectations

DOM/JS interop ([Ch14 §09](../14-Blazor/09-JSInterop.md)) still works for web-view DOM concerns, but for *device* features inject MAUI/Essentials services directly — no JS bridge needed.

---

## Summary

- **Blazor Hybrid** runs Blazor components ([Ch14](../14-Blazor/README.md)) **in-process on the native .NET runtime** inside a MAUI app, rendered in an embedded **`BlazorWebView`** — **full native device access, no browser sandbox, no network**.
- Set up with **`AddMauiBlazorWebView()`** + a `BlazorWebView` hosting a root component; from there it's ordinary Blazor (components/routing/DI) running natively.
- The key win: **share a Razor Class Library** of components between a Blazor **web** app and a MAUI Hybrid app — write UI once, ship as website **and** native apps; Hybrid components inject **MAUI/Essentials** services to use device features as plain C#.
- Choose Hybrid for **web-UI reuse + native capabilities**; choose MAUI XAML for **native-widget look**, and Blazor web for **browser reach**. You can mix native and `BlazorWebView` pages in one app.

→ Next: [10-Performance.md](10-Performance.md)
