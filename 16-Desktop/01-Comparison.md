# Choosing a Desktop Framework

## Three Windows options (plus a cross-platform one)

For Windows desktop apps on .NET, there are three first-party UI frameworks — **WPF**, **WinUI 3**, and **WinForms** — each with a distinct niche, plus the third-party cross-platform **Avalonia** ([05-Avalonia.md](05-Avalonia.md)). They all run on modern .NET (.NET 10), but differ in age, design language, capability, and ideal use case. The decision hinges on: is it a *new* app or *maintaining a legacy* one? Do you need *modern Fluent UI* or *maximum stability*? Windows-only or *cross-platform*?

```
                  WinForms        WPF             WinUI 3            Avalonia
Released          2002            2006            2021 (App SDK)     2020 (OSS)
UI model          controls/       XAML + binding  XAML + binding     XAML + binding
                  designer
Design language   classic Win32   themeable       Fluent / modern    custom/themeable
Maturity          rock-solid      mature, huge    newer, evolving    mature OSS
Platforms         Windows         Windows         Windows (modern)   Win/macOS/Linux
Best for          internal tools, line-of-biz,    new modern         cross-platform
                  quick UIs       MVVM apps       Windows apps        desktop
```

---

## WPF — the mature MVVM workhorse

**WPF** (Windows Presentation Foundation) is the most mature, full-featured Windows UI framework. It uses **XAML** + powerful **data binding** + **MVVM** ([02-WPF.md](02-WPF.md)), with a vector-based, fully themeable/templatable rendering engine. Its ecosystem (controls, tooling, knowledge) is enormous. It's the safe default for serious Windows line-of-business apps: stable, deeply capable (styles, templates, triggers, animations), and still actively supported on modern .NET.

**Choose WPF when**: building or maintaining a substantial Windows business app, you want mature MVVM + rich data binding + a huge control ecosystem, and you don't specifically need the latest Fluent/WinUI controls.

---

## WinUI 3 — the modern Windows successor

**WinUI 3** (part of the **Windows App SDK**) is Microsoft's modern, forward-looking Windows UI framework. It brings the latest **Fluent Design** controls and visuals, decoupled from the OS (shipped via the App SDK, so updates aren't tied to Windows releases), and is the recommended path for *new* apps wanting a modern Windows look ([03-WinUI3.md](03-WinUI3.md)). It also uses XAML + binding + MVVM, so concepts transfer from WPF. It's newer/less mature than WPF (smaller ecosystem, some rough edges historically), but it's where Microsoft's investment in Windows-native UI is going.

**Choose WinUI 3 when**: building a *new* Windows app that should look modern (Fluent), you want the latest Windows controls and Store/MSIX distribution, and you're comfortable with a newer (still-maturing) framework.

---

## WinForms — the pragmatic veteran

**WinForms** (Windows Forms) is the oldest, simplest model: a **designer-driven**, control-and-event approach (drag controls onto a form, handle events) ([04-WinForms.md](04-WinForms.md)). It's not modern or themeable like WPF/WinUI, but it's *extremely* productive for simple internal tools, utilities, and CRUD/data-entry apps — you get a working UI fast with minimal ceremony. It remains fully supported on modern .NET and is rock-solid.

**Choose WinForms when**: building an internal tool, utility, prototype, or simple data-entry app where development speed and stability beat visual polish — or maintaining an existing WinForms app.

---

## Avalonia — cross-platform XAML

**Avalonia** is a mature, open-source, **cross-platform** XAML UI framework (Windows, macOS, Linux, and more) — not from Microsoft, but conceptually close to WPF/WinUI (XAML + binding + MVVM), so skills transfer ([05-Avalonia.md](05-Avalonia.md)). It's the go-to when you need a *desktop* app that runs beyond Windows and prefer the XAML/MVVM model (vs MAUI, which targets mobile+desktop with native controls — [Ch15](../15-MAUI/README.md)).

**Choose Avalonia when**: you need a cross-platform desktop app (Linux/macOS/Windows) with a WPF-like XAML/MVVM model.

---

## The shared MVVM foundation

A crucial point: **WPF, WinUI 3, and Avalonia all use XAML + data binding + MVVM** ([Ch15 §06](../15-MAUI/06-MVVM.md) covers MVVM in depth, and it applies here identically). So:

- The **MVVM pattern**, `INotifyPropertyChanged`, commands, and **CommunityToolkit.Mvvm** source generators ([Ch15 §06](../15-MAUI/06-MVVM.md)) work across all three.
- View-models and most application logic are **portable** between them — only the View (XAML dialect + controls) differs.
- WinForms is the exception (event/designer model, not XAML/binding-first), though you can still apply MVVM-ish patterns.

This means the architectural investment (testable view-models, services, DI) carries across frameworks — picking a View technology doesn't lock your logic in.

---

## Decision guide

```
Maintaining an existing app?              → keep its framework (WPF/WinForms/etc.)
New, modern Windows look (Fluent)?        → WinUI 3
New serious Windows business app,
  mature ecosystem, rich binding?         → WPF
Quick internal tool / utility / CRUD?     → WinForms
Cross-platform desktop (Win/mac/Linux)?   → Avalonia
Cross-platform mobile + desktop, native?  → MAUI ([Ch15](../15-MAUI/README.md))
Web UI in a desktop shell?                → Blazor Hybrid ([Ch15 §09](../15-MAUI/09-BlazorHybrid.md))
```

There's no single "best" — match the framework to the app's needs, your team's familiarity, and whether you're starting fresh or maintaining.

---

## Common gotchas

### Assuming WinUI 3 replaces WPF everywhere

WinUI 3 is newer and Fluent-modern, but WPF is more mature with a larger ecosystem. For many business apps WPF is still the pragmatic choice; WinUI 3 shines for new, modern-looking apps. Both are supported.

### Dismissing WinForms as "obsolete"

WinForms is fully supported on modern .NET and unbeatable for quick internal tools. Don't over-engineer a simple utility with WPF/WinUI when WinForms ships it in a fraction of the time.

### Forgetting Avalonia/MAUI for cross-platform

WPF/WinUI 3/WinForms are **Windows-only**. For cross-platform desktop, use **Avalonia**; for mobile+desktop native, **MAUI**. Don't pick a Windows-only framework if you'll need other platforms.

### Thinking the choice locks in your logic

With MVVM, view-models and services are portable across WPF/WinUI/Avalonia. The View tech is the main difference — architect with MVVM so the choice isn't a deep lock-in.

---

## Summary

- Four desktop options on .NET 10: **WPF** (mature MVVM workhorse, huge ecosystem — serious Windows business apps), **WinUI 3** (modern Fluent, Windows App SDK — new modern Windows apps), **WinForms** (designer/event model — fast internal tools/utilities), and **Avalonia** (OSS **cross-platform** XAML — Win/macOS/Linux desktop).
- **WPF, WinUI 3, and Avalonia share XAML + data binding + MVVM**, so MVVM, `INotifyPropertyChanged`, commands, and CommunityToolkit.Mvvm ([Ch15 §06](../15-MAUI/06-MVVM.md)) — and your **view-models/logic** — are portable across them; **WinForms** is the event/designer exception.
- Choose by: maintaining (keep it) vs new; modern Fluent (WinUI 3) vs mature ecosystem (WPF) vs speed/simplicity (WinForms) vs cross-platform (Avalonia/MAUI).
- All four run on modern .NET and are supported — there's no universal "best," only the right fit.

→ Next: [02-WPF.md](02-WPF.md)
