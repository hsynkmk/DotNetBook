# Platform-Specific Code

## Sharing most, specializing some

MAUI's promise is one codebase, but each platform genuinely differs — APIs, permissions, lifecycle, idioms. The art is sharing the **majority** of code (often 80–95%) while cleanly isolating the per-platform bits. MAUI offers several mechanisms at different granularities: **conditional compilation** (`#if`), **partial classes** per platform, **`OnPlatform`** values in XAML, **platform folders**, and the **Essentials** APIs that already abstract common device features. Choosing the right tool keeps platform code contained instead of scattered.

---

## Essentials — the first resort

Before writing platform code, check whether **Essentials** already abstracts it. These cross-platform APIs cover most device features behind one interface ([07-DependencyInjection.md](07-DependencyInjection.md)):

```csharp
var location = await Geolocation.Default.GetLocationAsync();   // GPS, all platforms
var battery = Battery.Default.ChargeLevel;
Preferences.Default.Set("theme", "dark");                       // key-value storage
var result = await FilePicker.Default.PickAsync();
if (Connectivity.Current.NetworkAccess == NetworkAccess.Internet) { ... }
await Share.Default.RequestAsync(new ShareTextRequest("Hi"));
```

Essentials handles GPS, sensors, battery, preferences, file picking, connectivity, sharing, clipboard, and more — *without* any `#if`. Reach for platform-specific code only for features Essentials doesn't cover.

---

## Conditional compilation (`#if`)

For small platform divergences inline, the build defines per-target symbols (`ANDROID`, `IOS`, `MACCATALYST`, `WINDOWS`) you can branch on:

```csharp
public string GetDeviceTag() {
#if ANDROID
    return "android-" + Android.OS.Build.Model;
#elif IOS
    return "ios-" + UIKit.UIDevice.CurrentDevice.Model;
#elif WINDOWS
    return "win";
#else
    return "unknown";
#endif
}
```

`#if` is fine for *small* branches, but it gets unreadable fast and only the active target compiles (so a typo in an inactive branch isn't caught until you build that platform). For anything substantial, prefer partial classes (next).

---

## Partial classes per platform

The cleaner pattern for non-trivial platform code: a **partial class** with a shared declaration and per-platform implementations in the `Platforms/` folders. MAUI's build only includes the file matching the current target:

```csharp
// Services/DeviceInfoService.cs  (shared — the contract)
public partial class DeviceInfoService {
    public partial string GetModel();
}
```
```csharp
// Platforms/Android/DeviceInfoService.cs
public partial class DeviceInfoService {
    public partial string GetModel() => Android.OS.Build.Model;
}
```
```csharp
// Platforms/iOS/DeviceInfoService.cs
public partial class DeviceInfoService {
    public partial string GetModel() => UIKit.UIDevice.CurrentDevice.Model;
}
```

Each platform file lives under `Platforms/<Platform>/` and is compiled **only** for that target. This keeps each implementation in its own file (readable, type-checked against that platform's SDK) and the shared surface clean — far better than sprawling `#if` blocks. The same idea underlies multi-targeting via filename conventions (`Foo.Android.cs`, `Foo.iOS.cs`).

---

## `OnPlatform` in XAML

For per-platform *values* in markup (a different margin on iOS, a platform color), use the **`OnPlatform`** markup extension ([02-XAML.md](02-XAML.md)) — no code-behind:

```xml
<Label Text="Hi">
    <Label.Margin>
        <OnPlatform x:TypeArguments="Thickness">
            <On Platform="iOS" Value="0,20,0,0" />      <!-- account for the notch -->
            <On Platform="Android" Value="0,8,0,0" />
            <On Platform="WinUI" Value="0,4,0,0" />
        </OnPlatform>
    </Label.Margin>
</Label>

<!-- Compact form: -->
<Button HeightRequest="{OnPlatform iOS=44, Android=48, Default=40}" />
```

`OnIdiom` similarly varies values by device idiom (Phone/Tablet/Desktop). Use these for UI tweaks that differ per platform without branching in code.

---

## Platform lifecycle and permissions

- **Lifecycle**: each platform has its own app lifecycle (Android `Activity`, iOS `AppDelegate`). MAUI surfaces cross-platform lifecycle events (`OnStart`/`OnSleep`/`OnResume` on `App`, and `ConfigureLifecycleEvents` for platform-specific hooks). Handle backgrounding/resume where state must be saved.
- **Permissions**: runtime permissions (location, camera, notifications) are requested via Essentials `Permissions` API cross-platform, but each platform also needs its **manifest/Info.plist** entries declared in the `Platforms/` folder. Forgetting the manifest declaration causes the permission request to fail silently — a common platform-specific pitfall.

```csharp
var status = await Permissions.RequestAsync<Permissions.LocationWhenInUse>();
```

---

## Customizing handlers per platform

To tweak how a control maps to its native widget ([01-Overview.md](01-Overview.md)), customize its **handler** — adjust the underlying native view without replacing the whole control:

```csharp
Microsoft.Maui.Handlers.EntryHandler.Mapper.AppendToMapping("NoUnderline", (handler, view) => {
#if ANDROID
    handler.PlatformView.BackgroundTintList =
        Android.Content.Res.ColorStateList.ValueOf(Colors.Transparent.ToPlatform());
#endif
});
```

Handler mappers let you reach the native control (`handler.PlatformView`) for platform-specific styling/behavior that the cross-platform API doesn't expose — the modern replacement for Xamarin.Forms custom renderers, and lighter-weight.

---

## Common gotchas

### Writing platform code Essentials already provides

Reinventing GPS/battery/preferences with `#if` when Essentials covers it. Check Essentials first — it abstracts most device features with no platform branching.

### `#if` sprawl

Large `#if` blocks are unreadable and only the active branch is compiled (errors hide in inactive branches). Use partial classes in `Platforms/` for non-trivial platform code.

### Missing manifest/Info.plist permission entries

The Essentials `Permissions` request fails silently if the platform manifest (Android) / `Info.plist` (iOS) doesn't declare the permission. Always add the per-platform declaration.

### Assuming identical lifecycle/behavior

Backgrounding, resume, and navigation idioms differ per platform. Handle lifecycle events and test on each target — "runs on my emulator" isn't "works everywhere."

### Custom renderers (Xamarin habit)

MAUI replaced renderers with **handlers**. Use handler mappers (`AppendToMapping`) for native customization, not the old renderer pattern.

---

## Summary

- Share the **majority** of code; isolate platform specifics with the right tool: **Essentials** (device features, no branching — try first), **`#if`** (small inline branches), **partial classes in `Platforms/`** (non-trivial per-platform implementations — the clean pattern), and **`OnPlatform`/`OnIdiom`** (per-platform/idiom *values* in XAML).
- **Essentials** covers GPS, sensors, battery, preferences, file picking, connectivity, sharing, permissions — inject them as interfaces for testability ([07-DependencyInjection.md](07-DependencyInjection.md)).
- Per-platform code under `Platforms/<Platform>/` (or `Foo.Android.cs` naming) compiles **only** for that target — readable and type-checked against the right SDK; far better than `#if` sprawl.
- Handle each platform's **lifecycle** and declare **permissions** in the manifest/`Info.plist` (or requests fail silently); customize native rendering via **handler mappers** (not Xamarin renderers).

→ Next: [09-BlazorHybrid.md](09-BlazorHybrid.md)
