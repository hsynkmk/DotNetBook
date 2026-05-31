# Performance

## What matters on a device

Client-app performance has different stakes than server performance: users *feel* a slow startup, janky scrolling, or a laggy tap directly, and mobile devices have tighter CPU/memory/battery budgets. The big levers in MAUI are **startup time** (cold launch), **list/scroll smoothness** (virtualization), **image handling** (a top memory culprit), **layout efficiency**, and **build/deployment options** (AOT, trimming, R2R). Most user-perceived problems come from startup, lists, and images.

---

## Startup time

Cold-start latency is the first thing users judge. Reduce it by:

- **Lazy registration/navigation**: register services cheaply (avoid heavy work in constructors); defer non-essential initialization until after the first page shows. Don't do blocking I/O on the UI thread during startup.
- **Minimize the startup graph**: a giant DI graph constructed eagerly slows launch. Prefer `Transient`/lazy resolution for things not needed immediately; only the first page and its dependencies should be on the critical path ([07-DependencyInjection.md](07-DependencyInjection.md)).
- **AOT / startup tracing** (below): native AOT and startup profile-guided optimization cut JIT/startup cost, especially on Android.
- **Defer expensive resources**: large images, fonts, and data loads should not block first paint — load them after the UI appears (show a spinner / skeleton).

---

## Lists and virtualization

Lists are the most common scroll-performance problem. The rules ([03-Layouts.md](03-Layouts.md), [04-Controls.md](04-Controls.md)):

- **Use `CollectionView`** (virtualizing) for data lists — it renders only visible items + a buffer, recycling cells as you scroll. Never put a `CollectionView`/`ListView` inside a `ScrollView` or stack (it defeats virtualization).
- **Keep item templates lightweight**: deep nesting, many bindings, and heavy controls per cell multiply across thousands of items. Flatten the template (a Grid over nested stacks), minimize bindings, and avoid expensive converters per item.
- **Compiled bindings** (`x:DataType`) on templates — faster than reflection-based bindings and compile-checked ([02-XAML.md](02-XAML.md)).
- **Incremental loading**: load data in pages (`RemainingItemsThreshold` to fetch more as the user nears the end) rather than loading everything up front.

```xml
<CollectionView ItemsSource="{Binding Items}"
                RemainingItemsThreshold="10"
                RemainingItemsThresholdReachedCommand="{Binding LoadMoreCommand}">
    <CollectionView.ItemTemplate>
        <DataTemplate x:DataType="model:Item">
            <Grid ColumnDefinitions="*,Auto"> ... </Grid>   <!-- flat, compiled bindings -->
        </DataTemplate>
    </CollectionView.ItemTemplate>
</CollectionView>
```

---

## Images — the memory hog

Images are the leading cause of memory spikes and OOM crashes on mobile. Decoding a large photo to a full-resolution bitmap can consume tens of megabytes — multiplied across a list, it crashes the app:

- **Downsample to display size**: use `Image` source options / `DownsampleToViewSize` (or specify target width/height) so a 4000×3000 photo isn't decoded full-size into a 100×100 thumbnail slot.
- **Cache** remote images (the platform image loaders cache; libraries help) so they aren't re-downloaded/re-decoded.
- **Use appropriate formats and resolutions** per platform; ship vector/SVG where possible (MAUI rasterizes them at build to the right densities).
- **Dispose/avoid holding** large bitmaps; don't keep full-res images in memory longer than needed.

This single area — sizing images to their display size — prevents a large share of mobile memory crashes.

---

## Layout efficiency

The layout pass runs on every resize/orientation change and during scrolling:

- **Prefer `Grid`** over deeply nested `StackLayout`s for 2D structure — fewer measure/arrange passes ([03-Layouts.md](03-Layouts.md)).
- **Avoid excessive nesting**: each nested layout adds measure/arrange work; flatten where you can.
- **Avoid `AbsoluteLayout` for general structure** and don't trigger frequent re-layouts (e.g., animating layout properties that force a full pass).

---

## Build and deployment optimizations

For release builds ([Ch19 Deployment](../19-Deployment/README.md)):

| Option | Effect |
|---|---|
| **AOT compilation** | precompile IL → native, faster startup & execution (esp. Android); larger binary |
| **Trimming** | remove unused IL → smaller app; needs trim-safe code (watch reflection) |
| **ReadyToRun (R2R)** (Windows) | precompiled native code alongside IL — faster startup |
| **Release config** | enables optimizations, disables debug overhead — **always profile/ship Release** |

On **Android**, AOT (and startup tracing / profiled AOT) markedly improves cold start; on **iOS**, AOT is mandatory (no JIT allowed by the platform). **Trimming** shrinks size but can remove members accessed via reflection — test trimmed release builds, and use trim annotations/`DynamicDependency` where reflection is needed ([Ch01](../01-Runtime/README.md)).

---

## Measure, don't guess

Profile real devices (not just the fast dev machine/emulator): use platform profilers, `dotnet-trace`/`dotnet-counters` where applicable ([Ch21 Performance & Tooling](../21-Performance/README.md)), and the Blazor/MAUI diagnostics. Measure cold start, scroll frame rate, and memory under realistic data — optimize the proven hot spots, not assumptions.

---

## Common gotchas

### Heavy work on the startup/UI thread

Blocking I/O or large graph construction during launch stalls cold start. Defer non-essential init; keep the first-page critical path small.

### Lists without virtualization (or in a ScrollView)

A non-virtualized list, or a `CollectionView` nested in a `ScrollView`/stack, renders every item — slow and memory-heavy. Use `CollectionView` in a bounded cell with incremental loading.

### Full-resolution images

Decoding large images to full size (especially in lists) causes memory spikes/OOM. Downsample to display size and cache.

### Deeply nested layouts

Stacks-in-stacks multiply layout passes. Use `Grid` for 2D structure and flatten the tree.

### Shipping Debug builds / no AOT/trimming

Debug builds and un-optimized release builds are far slower/larger. Ship Release with AOT/trimming (test trimmed builds for reflection breakage); AOT is required on iOS.

---

## Summary

- The user-perceived levers are **startup**, **lists/scroll**, **images**, **layout**, and **build options** — most problems come from startup, lists, and images.
- **Startup**: defer non-essential init, keep the first-page DI graph small, avoid blocking the UI thread; use AOT/startup tracing.
- **Lists**: use **`CollectionView`** (virtualizing, never inside a ScrollView/stack) with **lightweight templates**, **compiled bindings** (`x:DataType`), and **incremental loading**.
- **Images**: **downsample to display size** and **cache** — the top fix for mobile memory/OOM crashes.
- **Layout**: prefer **Grid**, avoid deep nesting; **build**: ship **Release** with **AOT** (mandatory on iOS) and **trimming** (test for reflection breakage), R2R on Windows.
- **Measure on real devices** ([Ch21](../21-Performance/README.md)) — optimize proven hot spots, not guesses.

→ Next: [Questions.md](Questions.md)
