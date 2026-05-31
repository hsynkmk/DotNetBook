# .NET MAUI — Overview

## One codebase, four platforms

.NET MAUI (Multi-platform App UI) is the framework for building **native client apps for iOS, Android, macOS, and Windows from a single C# codebase**. It's the evolution of Xamarin.Forms, rebuilt on the modern .NET runtime and unified project system. You write your UI once (in XAML or C#), and MAUI maps your controls onto each platform's **native** controls — a MAUI `Button` becomes a `UIButton` on iOS, an `Android.Widget.Button` on Android, and so on. The result looks and feels native because it *is* native, not a web view or a custom-drawn imitation.

```
Your code (C#/XAML)
      │
   MAUI control abstraction (Button, Entry, CollectionView, ...)
      │  ── handlers map to ──
   ┌──────────┬──────────┬──────────┬──────────┐
  iOS        Android     macOS      Windows
 (UIKit)   (Android SDK)(AppKit)   (WinUI 3)
```

---

## What MAUI gives you

- **A single project** that targets multiple platforms (`net10.0-android`, `net10.0-ios`, `net10.0-maccatalyst`, `net10.0-windows`) — one `.csproj`, one set of source files, platform-specific code only where needed ([08-PlatformSpecific.md](08-PlatformSpecific.md)).
- **Native UI** via the **handler** architecture — each cross-platform control is realized by a platform handler that creates and manages the real native view (a leaner, more extensible model than Xamarin.Forms' renderers).
- **Cross-platform APIs** ("Essentials") for device features — GPS, sensors, battery, preferences, file picker, connectivity — through one API surface that works on every platform.
- **Shared .NET BCL** — the same C#, the same libraries, the same NuGet ecosystem ([Ch02 BCL](../02-BCL/README.md)) across all targets.
- **The .NET Generic Host** ([Ch03 §01](../03-HostingAndDI/01-GenericHost.md)) — MAUI apps are configured with a `MauiAppBuilder` that wires up **dependency injection**, configuration, and logging just like ASP.NET Core ([07-DependencyInjection.md](07-DependencyInjection.md)).

---

## The project shape

A MAUI app's structure is consistent across platforms with a `Platforms/` folder for per-OS entry points:

```
MyApp/
├── MauiProgram.cs            ← app bootstrap (MauiAppBuilder, DI registration)
├── App.xaml(.cs)             ← application class, resources
├── AppShell.xaml(.cs)        ← Shell navigation root ([05-Navigation.md])
├── MainPage.xaml(.cs)        ← a page (UI + code-behind)
├── ViewModels/               ← MVVM view-models ([06-MVVM.md])
├── Services/                 ← app services (injected via DI)
├── Resources/                ← images, fonts, styles, raw assets
└── Platforms/
    ├── Android/              ← MainActivity, manifest, Android-specific code
    ├── iOS/                  ← AppDelegate, Info.plist
    ├── MacCatalyst/
    └── Windows/              ← App.xaml, packaging
```

`MauiProgram.CreateMauiApp()` is the entry point where you configure the app — register fonts, set up DI services, add configuration. It returns a built `MauiApp`, analogous to ASP.NET Core's `WebApplication`.

```csharp
public static class MauiProgram {
    public static MauiApp CreateMauiApp() {
        var builder = MauiApp.CreateBuilder();
        builder.UseMauiApp<App>()
               .ConfigureFonts(f => f.AddFont("OpenSans-Regular.ttf", "OpenSansRegular"));
        builder.Services.AddSingleton<IDataService, DataService>();   // DI, like ASP.NET Core
        builder.Services.AddTransient<MainPage>();
        return builder.Build();
    }
}
```

---

## How a MAUI app runs

Each platform has a native entry point (Android `MainActivity`, iOS `AppDelegate`, etc.) that calls `MauiProgram.CreateMauiApp()`. From there, MAUI takes over: it creates the `App`, sets up the navigation host (`Shell`), and renders pages. When you display a `ContentPage` with a `Button`, MAUI's handler for `Button` instantiates the *native* button widget and keeps it in sync with your cross-platform control's properties. Events flow back from native → handler → your C# event/command.

The key mental model: **you program against the abstraction; handlers bridge to native at runtime.** This is why MAUI apps perform and feel native — there's a real native control underneath every MAUI control.

---

## MAUI vs Blazor Hybrid vs web

MAUI gives you **native UI**. If you'd rather build the UI with web technologies (Blazor components) inside a native shell, **Blazor Hybrid** ([09-BlazorHybrid.md](09-BlazorHybrid.md)) hosts a `BlazorWebView` in a MAUI app — your Blazor components ([Ch14](../14-Blazor/README.md)) render in an embedded web view with full native-device access. Choose:

| You want | Use |
|---|---|
| Native look/feel, platform-idiomatic UI | MAUI (XAML/C#) |
| Reuse Blazor components / web skills in a native app | MAUI Blazor Hybrid |
| Reach via the browser, no install | Blazor WebAssembly / Server ([Ch14](../14-Blazor/README.md)) |

---

## When MAUI fits (and when it doesn't)

- **Fits**: cross-platform mobile + desktop from one C# codebase, teams already in .NET, apps needing native device features and native UX, line-of-business apps.
- **Less ideal**: a single-platform app where the native SDK's full surface and tooling matter most (a pure SwiftUI/Jetpack Compose app may be better); games (use a game engine); or when a web app suffices (no install, easier reach).

MAUI's sweet spot is sharing the **majority** of code (logic, view-models, services — often 80–95%) across platforms while dropping to platform-specific code only where each OS differs.

---

## Common gotchas

### Expecting "write once, run identically everywhere"

Platforms differ (navigation idioms, permissions, status bars, lifecycle). MAUI shares most code but you'll still handle platform specifics ([08-PlatformSpecific.md](08-PlatformSpecific.md)) and test on each target. It's "write once, *adapt* where needed," not pixel-identical everywhere.

### Treating it like a web app

MAUI is native client development — there's app lifecycle, packaging, store submission, and per-platform permissions. It's closer to Xamarin/native mobile than to ASP.NET Core web.

### Confusing MAUI with Blazor Hybrid

MAUI renders native controls; Blazor Hybrid renders Blazor components in a web view inside MAUI. They're different UI approaches that can even coexist — pick deliberately ([09-BlazorHybrid.md](09-BlazorHybrid.md)).

### Ignoring per-platform setup

Each platform needs SDKs/workloads (Android SDK, Xcode for iOS/macOS, Windows SDK), and iOS/macOS builds require a Mac. Plan the build environment accordingly.

---

## Summary

- **.NET MAUI** builds **native** apps for **iOS, Android, macOS, and Windows** from **one C# codebase** — the successor to Xamarin.Forms, on modern .NET.
- Cross-platform controls map to **real native widgets** via the **handler** architecture; you program the abstraction, handlers bridge to native at runtime (native look, feel, and performance).
- A **single project** multi-targets the platforms (`net10.0-android`/`-ios`/`-maccatalyst`/`-windows`) with a `Platforms/` folder for per-OS code; **`MauiProgram.CreateMauiApp()`** bootstraps the app with **DI/config/logging** via the Generic Host ([Ch03](../03-HostingAndDI/README.md)).
- **Essentials** APIs expose device features cross-platform; share the majority of code (logic, view-models) and drop to platform specifics only where OSes differ.
- For web-tech UI in a native shell, use **Blazor Hybrid** ([09-BlazorHybrid.md](09-BlazorHybrid.md)); choose MAUI for native UX, Blazor web for browser reach.

→ Next: [02-XAML.md](02-XAML.md)
