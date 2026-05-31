# Desktop Performance

## Keep the UI thread free, render efficiently

Desktop UI frameworks (WPF, WinUI 3, WinForms, Avalonia) share a fundamental constraint: a **single UI thread** owns the controls and the message loop. Performance and responsiveness come down to three things: **never block the UI thread**, **virtualize large data sets**, and **render efficiently** (avoid layout/visual-tree thrash). Get those right and a desktop app feels instant; get them wrong and it freezes, jank-scrolls, or hogs memory.

---

## Never block the UI thread

The cardinal rule: the UI thread must stay free to process input and repaint. Any long operation on it freezes the app (the dreaded "Not Responding"):

```csharp
// ✗ Freezes the UI for the duration:
private void Load_Click(object s, EventArgs e) {
    var data = _service.GetData();        // synchronous I/O blocks the UI thread
    grid.ItemsSource = data;
}

// ✓ Async keeps the UI responsive:
private async void Load_Click(object s, EventArgs e) {
    var data = await _service.GetDataAsync();   // await frees the UI thread; resumes on it
    grid.ItemsSource = data;                     // back on the UI thread after await
}
```

- Use **`async`/`await`** for I/O — `await` releases the UI thread and resumes on it (the UI `SynchronizationContext`), so you can touch controls after the await.
- Use **`Task.Run`** for CPU-bound work, then update the UI on completion (the await marshals back).
- **Marshal cross-thread updates**: touching a control from a background thread throws. Use the dispatcher (`Dispatcher.Invoke` in WPF/WinUI/Avalonia, `Control.Invoke` in WinForms) — or just `await`, which returns to the UI context automatically.

This is the same async model as everywhere in .NET (CSharpBook async chapter) — desktop just makes thread-affinity violations crash visibly.

---

## Virtualize large lists

Rendering thousands of rows of UI is slow and memory-heavy. **UI virtualization** renders only the items currently visible (plus a buffer), recycling containers as you scroll — the single biggest win for data-heavy desktop apps:

- **WPF**: `VirtualizingStackPanel` is the default for `ListBox`/`ListView`/`DataGrid`; ensure `VirtualizingPanel.IsVirtualizing="True"` and use **container recycling** (`VirtualizationMode=Recycling`). **Don't** put an `ItemsControl` in a `ScrollViewer` that gives it infinite height — that disables virtualization (same trap as MAUI — [Ch15 §03](../15-MAUI/03-Layouts.md)).
- **WinUI 3 / Avalonia**: virtualizing list panels similarly render only visible items.
- **WinForms**: `DataGridView` is efficient for large grids; for huge data use **virtual mode** (`VirtualMode = true`, supply cells on demand via `CellValueNeeded`) so it doesn't hold every row in memory.

```xml
<ListView VirtualizingPanel.IsVirtualizing="True"
          VirtualizingPanel.VirtualizationMode="Recycling"
          ItemsSource="{Binding Items}" />
```

---

## Render efficiently

The layout and visual tree cost real time on every change:

- **Minimize visual-tree depth**: deeply nested panels multiply measure/arrange passes. Prefer a `Grid` over nested `StackPanel`s for structure (WPF/WinUI/Avalonia) — same principle as MAUI ([Ch15 §03](../15-MAUI/03-Layouts.md)).
- **Avoid layout thrash**: changing properties that trigger a full re-layout in a tight loop (or per-frame) is expensive. Batch updates; freeze collections during bulk changes.
- **Freeze immutable resources** (WPF): `Freezable` objects (brushes, geometries) that won't change can be **`Freeze()`d** so WPF skips change tracking and can share them across threads — a cheap perf win for shared brushes/pens.
- **Compiled bindings** (`{x:Bind}` in WinUI, `x:DataType` in WPF/Avalonia) avoid per-binding reflection — faster and compile-checked ([03-WinUI3.md](03-WinUI3.md)).
- **Reduce binding/converter work** per item in templates; keep item templates lightweight (virtualization recycles them, but each is still rendered).

---

## Hardware acceleration

WPF, WinUI 3, and Avalonia render via the GPU (DirectX/Skia) — most rendering is hardware-accelerated by default. Things that **fall back to software rendering** (slow): certain effects (large `BitmapEffect`/legacy effects), very large/complex geometries, or layered windows in some cases. Prefer modern, GPU-friendly effects (`Effect`/shaders) over deprecated CPU effects, and watch for software-fallback triggers in profiling. WinForms is GDI-based (CPU) — fine for typical forms but not for heavy graphics (use a GPU framework for that).

---

## Startup and memory

- **Startup**: defer non-essential initialization; don't construct a huge object/DI graph or do blocking I/O before the first window shows. Show the window fast, load data after.
- **Images**: like mobile ([Ch15 §10](../15-MAUI/10-Performance.md)), decode images to display size — full-resolution bitmaps in a list spike memory. WPF: set `BitmapImage.DecodePixelWidth`/`Height`.
- **Leaks**: a common desktop leak is **event-handler retention** — subscribing to a long-lived object's event from a short-lived view without unsubscribing keeps the view alive. Unsubscribe (or use weak events — WPF's `WeakEventManager`).

---

## Common gotchas

### Synchronous work on the UI thread

Blocking I/O or CPU work in an event handler freezes the app. Use `async`/`await` (I/O) and `Task.Run` (CPU), update UI after the await.

### Cross-thread control access

Updating a control from a background thread throws. Marshal via the dispatcher (`Dispatcher.Invoke`/`Control.Invoke`) or rely on `await` returning to the UI context.

### Disabling virtualization (ItemsControl in a ScrollViewer)

Wrapping a list in something that grants infinite height disables UI virtualization, rendering every item. Keep virtualizing panels bounded; enable recycling.

### Event-handler memory leaks

Not unsubscribing from a long-lived publisher keeps subscribers alive. Unsubscribe or use weak events.

### Heavy/legacy effects forcing software rendering

Deprecated CPU effects (and some complex visuals) drop GPU acceleration. Use modern GPU-friendly effects; profile for software-render fallback.

---

## Summary

- Desktop frameworks have a **single UI thread** — **never block it**: use **`async`/`await`** (I/O) and **`Task.Run`** (CPU), and **marshal** cross-thread UI updates via the dispatcher (`Dispatcher.Invoke`/`Control.Invoke`) or rely on `await` resuming on the UI context.
- **Virtualize** large lists (WPF `VirtualizingStackPanel` + recycling; WinForms `DataGridView` virtual mode) — the biggest data-heavy win; don't disable virtualization by giving lists infinite height.
- **Render efficiently**: shallow visual trees (Grid over nested panels), avoid layout thrash, **`Freeze()`** immutable WPF resources, use **compiled bindings** and lightweight item templates.
- Rendering is **GPU-accelerated** (WPF/WinUI/Avalonia) — avoid legacy effects that force software fallback; **defer startup work**, decode **images to display size**, and **unsubscribe events** (weak events) to avoid leaks.

→ Next: [07-Packaging.md](07-Packaging.md)
