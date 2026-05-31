# WinUI 3

## The modern Windows UI

**WinUI 3** is Microsoft's modern, forward-looking Windows UI framework, shipped as part of the **Windows App SDK**. It delivers the latest **Fluent Design** controls and visuals (the look of modern Windows 11 apps) and — crucially — is **decoupled from the OS**: it ships via the App SDK NuGet packages rather than being baked into Windows, so you get the newest controls without waiting for a Windows release. For *new* Windows apps that should look modern, WinUI 3 is Microsoft's recommended path. It uses the familiar **XAML + data binding + MVVM** model, so concepts and view-models carry over from WPF ([02-WPF.md](02-WPF.md)) and MAUI ([Ch15 §06](../15-MAUI/06-MVVM.md)).

```xml
<Window x:Class="MyApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <StackPanel Padding="24" Spacing="12">
        <TextBlock Text="{x:Bind ViewModel.Title}" Style="{StaticResource TitleTextBlockStyle}" />
        <Button Content="Go" Command="{x:Bind ViewModel.GoCommand}" />
    </StackPanel>
</Window>
```

---

## Windows App SDK and WinUI lineage

Some history clarifies the landscape:

- **UWP** (Universal Windows Platform) introduced WinUI 2 — modern controls, but tied to the UWP sandbox/lifecycle and to Windows releases.
- **WinUI 3 + Windows App SDK** decouples that modern UI stack from UWP and from the OS: you build a **standard .NET desktop app** (full Win32 capabilities, normal .NET access) that uses the modern WinUI controls. It's "modern Windows UI without the UWP constraints."

So WinUI 3 gives you Fluent UI **and** the freedom of a normal desktop app (full file system, P/Invoke, the whole BCL — [Ch02](../02-BCL/README.md)), unlike the old UWP sandbox.

---

## `x:Bind` — compiled bindings

WinUI 3 (inherited from UWP) features **`{x:Bind}`**, a **compiled** binding that's faster and type-checked at build time, in contrast to the reflection-based `{Binding}`:

```xml
<TextBlock Text="{x:Bind ViewModel.UserName, Mode=OneWay}" />
<ListView ItemsSource="{x:Bind ViewModel.Items}" />
```

- `{x:Bind}` resolves against the **page's code-behind** (not a `DataContext`) by default, generates code at compile time, and **catches binding errors at build** (a typo is a compile error, not a silent failure).
- It defaults to `OneTime` mode (specify `Mode=OneWay`/`TwoWay` for updates) — a common gotcha.
- `{Binding}` still works (DataContext-based, reflection) for dynamic scenarios.

Prefer `{x:Bind}` in WinUI 3 for performance and compile-time safety; this is a notable difference from WPF (which only has reflection `{Binding}`).

---

## Fluent controls and design

WinUI 3 ships the modern Windows control set with Fluent styling out of the box: `NavigationView` (the hamburger/nav pane pattern), `TabView`, `InfoBar`, `TeachingTip`, modern `ContentDialog`, acrylic/mica materials, and theme-aware light/dark resources. These give apps the native Windows 11 look with minimal effort:

```xml
<NavigationView PaneTitle="My App" IsBackButtonVisible="Visible">
    <NavigationView.MenuItems>
        <NavigationViewItem Content="Home" Icon="Home" Tag="home" />
        <NavigationViewItem Content="Settings" Icon="Setting" Tag="settings" />
    </NavigationView.MenuItems>
    <Frame x:Name="ContentFrame" />
</NavigationView>
```

If you want a polished, modern Windows aesthetic without hand-building it, WinUI 3's Fluent controls are the biggest draw over WPF (whose default look is older, though fully restyleable).

---

## MVVM and DI

WinUI 3 uses the same MVVM approach as the rest of this part — **CommunityToolkit.Mvvm** (`[ObservableProperty]`, `[RelayCommand]`) for boilerplate-free view-models, bound via `{x:Bind}`/`{Binding}`, with **Generic Host** DI ([Ch03](../03-HostingAndDI/README.md)) bootstrapping services and resolving windows/pages. Because the pattern matches WPF/MAUI/Avalonia, **view-models and services are portable** — you can share logic across a WinUI 3 app and others, differing only in the View layer ([01-Comparison.md](01-Comparison.md)).

---

## Maturity considerations

WinUI 3 is **newer and less mature** than WPF: a smaller third-party control ecosystem, fewer years of accumulated knowledge, and historically some rough edges (tooling, certain APIs, packaging complexity). It's improved substantially and is where Microsoft invests for Windows-native UI, but for a large, conservative business app with heavy third-party control needs, WPF's maturity may still win. Weigh "modern Fluent + Microsoft's strategic direction" (WinUI 3) against "mature, vast ecosystem, battle-tested" (WPF) — [01-Comparison.md](01-Comparison.md).

---

## Common gotchas

### `{x:Bind}` defaults to `OneTime`

Unlike `{Binding}`, `{x:Bind}` is `OneTime` by default — bound values won't update unless you set `Mode=OneWay`/`TwoWay`. Forgetting this makes the UI appear "stuck."

### Expecting UWP sandbox or assuming full UWP parity

WinUI 3 desktop apps are normal .NET desktop apps (full Win32/BCL access) — *not* sandboxed UWP. Don't carry UWP assumptions; conversely, some UWP-only APIs differ.

### Packaging complexity

WinUI 3 apps often use **MSIX** packaging (and historically required it; unpackaged is supported now). Packaging/identity can complicate deployment vs a plain WPF exe — plan distribution ([07-Packaging.md](07-Packaging.md)).

### Treating it as a drop-in WPF replacement

Concepts transfer (XAML/MVVM), but controls, some XAML dialect, `{x:Bind}`, and the App SDK differ. Migrating from WPF is a port, not a recompile.

### Underestimating maturity gaps

The third-party ecosystem and certain features are less mature than WPF. Verify your required controls/libraries exist for WinUI 3 before committing.

---

## Summary

- **WinUI 3** (Windows App SDK) is Microsoft's **modern Fluent** Windows UI, **decoupled from the OS** (ships via the App SDK), and the recommended path for **new modern Windows apps** — built as full **desktop** apps (not UWP-sandboxed), with full Win32/BCL access.
- It uses **XAML + binding + MVVM** (portable view-models with WPF/MAUI/Avalonia) and features **`{x:Bind}`** — **compiled, type-checked** bindings (faster, build-time error catching) that default to `OneTime` (set `Mode` for updates).
- Ships the modern **Fluent control set** (`NavigationView`, `TabView`, `InfoBar`, mica/acrylic) for the native Windows 11 look with little effort — its main draw over WPF.
- Uses **CommunityToolkit.Mvvm** + **Generic Host DI** like the rest of the stack; it's **less mature** than WPF (smaller ecosystem, MSIX packaging complexity) — weigh modern-Fluent/strategic-direction vs WPF's maturity.

→ Next: [04-WinForms.md](04-WinForms.md)
