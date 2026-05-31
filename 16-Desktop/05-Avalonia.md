# Avalonia

## Cross-platform XAML

**Avalonia** is a mature, open-source UI framework that brings the **XAML + data binding + MVVM** model to **multiple platforms** — Windows, macOS, Linux, and (increasingly) mobile and WebAssembly — from a single codebase. It's not a Microsoft product, but it's conceptually close to WPF/WinUI ([02-WPF.md](02-WPF.md), [03-WinUI3.md](03-WinUI3.md)), so the concepts and much of the architecture transfer directly. For developers who want a *desktop* app that runs beyond Windows and prefer the WPF-style XAML/MVVM approach (rather than MAUI's native-control model — [Ch15](../15-MAUI/README.md)), Avalonia is the leading choice.

```xml
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        x:Class="MyApp.MainWindow" Title="Avalonia Demo">
    <StackPanel Margin="16" Spacing="8">
        <TextBlock Text="{Binding Greeting}" FontSize="20" />
        <Button Content="Click" Command="{Binding ClickCommand}" />
    </StackPanel>
</Window>
```

---

## Why Avalonia (vs WPF/WinUI/MAUI)

| | WPF / WinUI 3 | **Avalonia** | MAUI |
|---|---|---|---|
| Platforms | Windows only | **Win/macOS/Linux** (+ mobile/web) | mobile + desktop |
| UI model | XAML + MVVM | **XAML + MVVM** (WPF-like) | XAML + MVVM (native controls) |
| Rendering | native/DirectX | **own renderer (Skia)** — consistent across OSes | native widgets per platform |
| Vendor | Microsoft | **open source / community** | Microsoft |

Avalonia **draws its own controls** (via Skia) rather than wrapping each OS's native widgets — so the app looks **identical across platforms** (like Flutter's approach), which is a feature (consistency) or a trade-off (not pixel-native per OS) depending on your goal. This contrasts with MAUI, which maps to **native** controls per platform ([Ch15 §01](../15-MAUI/01-Overview.md)). For a consistent, themeable desktop UI across Win/macOS/Linux, Avalonia's self-rendered approach is ideal.

---

## WPF-like, but its own framework

Avalonia deliberately resembles WPF — styles, control templates, data binding, `Grid`/`StackPanel` layouts, MVVM — so WPF developers feel at home and view-models port easily. Differences to note:

- **Styling** uses a CSS-like selector syntax (more powerful/flexible than WPF's style targeting in some ways).
- **Compiled bindings** (`x:CompileBindings` / `x:DataType`) give type-checked, fast bindings (like WinUI's `{x:Bind}`).
- Its **own XAML namespace** and some control/API name differences.
- A vibrant OSS ecosystem and tooling (the Avalonia templates, the previewer).

The upshot: if you know WPF/WinUI MVVM, you're ~90% of the way to Avalonia; the remaining 10% is its specific XAML/styling dialect.

---

## MVVM and the shared foundation

Avalonia uses the same MVVM stack as the rest of this part — **CommunityToolkit.Mvvm** (`[ObservableProperty]`, `[RelayCommand]`, `ObservableObject`) for boilerplate-free view-models, `ObservableCollection<T>` for lists, commands bound to controls, and **Generic Host / DI** ([Ch03](../03-HostingAndDI/README.md), [Ch15 §06–07](../15-MAUI/06-MVVM.md)) for services. Because the pattern is identical, **view-models and application logic are portable** between Avalonia and WPF/WinUI — you can target Avalonia for cross-platform reach while reusing the same logic layer ([01-Comparison.md](01-Comparison.md)).

---

## When to choose Avalonia

- **Cross-platform desktop** (Linux + macOS + Windows) with a single XAML/MVVM codebase — its sweet spot.
- You want **consistent UI** across OSes (self-rendered) rather than per-OS native look.
- You're a **WPF developer** needing cross-platform reach and want familiar concepts.
- Open-source preference / no dependency on Microsoft's first-party desktop frameworks.

Reach for **MAUI** instead if you need **mobile** as the primary target with **native** platform look; reach for **WPF/WinUI** if you're **Windows-only** and want first-party/Fluent. Avalonia fills the "WPF-style, but cross-platform desktop" gap.

---

## Common gotchas

### Expecting pixel-native per-OS look

Avalonia **draws** its controls (Skia), so the app looks consistent everywhere — not like each OS's native widgets. If platform-native appearance is required, use MAUI (native controls) instead.

### Assuming WPF XAML is identical

It's WPF-*like*, not WPF. The namespace, styling selector syntax, and some controls/APIs differ. Porting WPF XAML needs adjustment (though view-models port cleanly).

### Forgetting compiled bindings

Like WinUI's `{x:Bind}`, Avalonia supports compiled bindings (`x:DataType`/`CompileBindings`) for speed and compile-time checking. Reflection bindings work but are slower and fail silently — prefer compiled.

### Treating it as unsupported because it's not Microsoft

Avalonia is mature, widely used, and actively maintained (commercial backing/support available). Being OSS/non-Microsoft isn't a maturity concern here.

---

## Summary

- **Avalonia** brings **WPF-style XAML + binding + MVVM** to **cross-platform desktop** (Windows/macOS/Linux, plus mobile/web) from one codebase — an open-source framework, conceptually close to WPF/WinUI so concepts and view-models transfer.
- It **draws its own controls** (Skia), giving a **consistent look across all platforms** (vs MAUI's per-platform **native** controls) — consistency as a feature/trade-off.
- It's WPF-*like* with its own dialect: CSS-like styling selectors, **compiled bindings** (`x:DataType`), its own namespace and some API differences; uses the same **CommunityToolkit.Mvvm + Generic Host DI** stack, so **logic is portable**.
- **Choose Avalonia** for cross-platform *desktop* with a WPF-style model and consistent UI; choose **MAUI** for mobile-first native, **WPF/WinUI** for Windows-only first-party.

→ Next: [06-Performance.md](06-Performance.md)
