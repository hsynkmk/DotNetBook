# Chapter 15 — .NET MAUI — Coding Problems

Build a multi-page MVVM app with services, navigation, and platform code. Each problem has a hidden solution — attempt it first.

---

### Problem 1 — Bootstrap a MAUI app with DI

Write `MauiProgram.CreateMauiApp()` registering a service, a view-model, and a page.

<details>
<summary>Solution</summary>

```csharp
public static class MauiProgram {
    public static MauiApp CreateMauiApp() {
        var builder = MauiApp.CreateBuilder();
        builder.UseMauiApp<App>()
               .ConfigureFonts(f => f.AddFont("OpenSans-Regular.ttf", "OpenSansRegular"));

        builder.Services.AddSingleton<IProductService, ProductService>();
        builder.Services.AddSingleton<MainViewModel>();
        builder.Services.AddSingleton<MainPage>();
        builder.Services.AddTransient<DetailViewModel>();
        builder.Services.AddTransient<DetailPage>();

        return builder.Build();
    }
}
```

Built on the Generic Host — same DI container as ASP.NET Core. Singleton for the persistent main page/VM; transient for the per-item detail page/VM.
</details>

---

### Problem 2 — A MVVM view-model with the toolkit

Write a `CounterViewModel` with an observable `Count` and an `Increment` command, using CommunityToolkit.Mvvm.

<details>
<summary>Solution</summary>

```csharp
public partial class CounterViewModel : ObservableObject {
    [ObservableProperty]
    private int count;                       // generates public int Count with notification

    [RelayCommand]
    private void Increment() => Count++;      // generates IncrementCommand
}
```
```xml
<Label Text="{Binding Count}" />
<Button Text="+1" Command="{Binding IncrementCommand}" />
```

`partial` is required (the generator extends the class). Annotate the lowercase field `count`; bind to the generated `Count`.
</details>

---

### Problem 3 — A responsive Grid layout

Lay out a header (auto height, full width), a content area that fills remaining space, and a footer button — with a two-column content row.

<details>
<summary>Solution</summary>

```xml
<Grid RowDefinitions="Auto,*,Auto" ColumnDefinitions="*,2*" RowSpacing="8">
    <Label Grid.Row="0" Grid.ColumnSpan="2" Text="Header" FontSize="20" />
    <CollectionView Grid.Row="1" Grid.Column="0" ItemsSource="{Binding Items}" />
    <BoxView Grid.Row="1" Grid.Column="1" Color="LightGray" />
    <Button Grid.Row="2" Grid.ColumnSpan="2" Text="Done" Command="{Binding DoneCommand}" />
</Grid>
```

`Auto,*,Auto` rows: header/footer size to content, content row absorbs the rest. The `*` row is what makes it responsive.
</details>

---

### Problem 4 — A bound list with CollectionView

Display a list of products (name + price) bound to an `ObservableCollection`, with compiled bindings.

<details>
<summary>Solution</summary>

```csharp
public ObservableCollection<Product> Products { get; } = [];
[RelayCommand] private async Task LoadAsync() {
    Products.Clear();
    foreach (var p in await _service.GetAllAsync()) Products.Add(p);
}
```
```xml
<CollectionView ItemsSource="{Binding Products}">
    <CollectionView.ItemTemplate>
        <DataTemplate x:DataType="model:Product">
            <Grid Padding="10" ColumnDefinitions="*,Auto">
                <Label Text="{Binding Name}" />
                <Label Grid.Column="1" Text="{Binding Price, StringFormat='{0:C}'}" />
            </Grid>
        </DataTemplate>
    </CollectionView.ItemTemplate>
</CollectionView>
```

`ObservableCollection` → UI updates on add/remove. `x:DataType` → compiled, checked, fast bindings. Flat Grid template for scroll performance.
</details>

---

### Problem 5 — Shell navigation with a parameter

Navigate from a list to a detail page, passing a product id, and load the product on arrival.

<details>
<summary>Solution</summary>

```csharp
// AppShell ctor: register the detail route
Routing.RegisterRoute("product/details", typeof(DetailPage));

// From the list VM:
[RelayCommand] private async Task SelectAsync(Product p)
    => await Shell.Current.GoToAsync($"product/details?id={p.Id}");
```
```csharp
// DetailViewModel receives the parameter:
[QueryProperty(nameof(ProductId), "id")]
public partial class DetailViewModel : ObservableObject {
    public string ProductId { set => _ = LoadAsync(int.Parse(value)); }
    async Task LoadAsync(int id) { Product = await _service.GetAsync(id); }
}
```

Pass an **id** (not the whole object) — robust to deep links/restarts; reload on the destination.
</details>

---

### Problem 6 — Inject the page's view-model

Wire `MainPage` to receive `MainViewModel` via constructor injection.

<details>
<summary>Solution</summary>

```csharp
public partial class MainPage : ContentPage {
    public MainPage(MainViewModel vm) {       // DI injects the VM
        InitializeComponent();
        BindingContext = vm;
    }
}
```
Registered in `MauiProgram` (`AddSingleton<MainPage>()`, `AddSingleton<MainViewModel>()`). DI resolves the VM (and its services) and injects it — no `new`, no hardcoded dependencies.
</details>

---

### Problem 7 — Use an Essentials API testably

A `SyncViewModel` should only sync when online. Make it unit-testable.

<details>
<summary>Solution</summary>

```csharp
public partial class SyncViewModel(IConnectivity connectivity) : ObservableObject {
    [RelayCommand]
    private async Task SyncAsync() {
        if (connectivity.NetworkAccess != NetworkAccess.Internet) return;
        await DoSyncAsync();
    }
}
// Register: builder.Services.AddSingleton(Connectivity.Current);
```

Inject `IConnectivity` (not `Connectivity.Current` static) so a test can pass a fake returning `NetworkAccess.None`/`Internet`.
</details>

---

### Problem 8 — Platform-specific code with a partial class

Implement `GetDeviceModel()` per platform without `#if` sprawl.

<details>
<summary>Solution</summary>

```csharp
// Services/DeviceInfoService.cs (shared)
public partial class DeviceInfoService { public partial string GetModel(); }

// Platforms/Android/DeviceInfoService.cs
public partial class DeviceInfoService {
    public partial string GetModel() => Android.OS.Build.Model;
}
// Platforms/iOS/DeviceInfoService.cs
public partial class DeviceInfoService {
    public partial string GetModel() => UIKit.UIDevice.CurrentDevice.Model;
}
```

Each platform file under `Platforms/<Platform>/` compiles only for that target — readable and type-checked against that SDK.
</details>

---

### Problem 9 — Per-platform value in XAML

Give a label a different top margin on iOS (notch) vs Android vs Windows, in XAML only.

<details>
<summary>Solution</summary>

```xml
<Label Text="Title">
    <Label.Margin>
        <OnPlatform x:TypeArguments="Thickness">
            <On Platform="iOS" Value="0,20,0,0" />
            <On Platform="Android" Value="0,8,0,0" />
            <On Platform="WinUI" Value="0,4,0,0" />
        </OnPlatform>
    </Label.Margin>
</Label>
<!-- Or compact: -->
<Button HeightRequest="{OnPlatform iOS=44, Android=48, Default=40}" />
```

`OnPlatform` varies values per platform with no code-behind branching.
</details>

---

### Problem 10 — Async command with busy state

A `LoadCommand` should disable the button and show a spinner while loading.

<details>
<summary>Solution</summary>

```csharp
public partial class ListViewModel : ObservableObject {
    [ObservableProperty] private bool isBusy;

    [RelayCommand]
    private async Task LoadAsync() {
        IsBusy = true;
        try { Items = await _service.GetAsync(); }
        finally { IsBusy = false; }
    }
}
```
```xml
<ActivityIndicator IsRunning="{Binding IsBusy}" IsVisible="{Binding IsBusy}" />
<Button Text="Load" Command="{Binding LoadCommand}" />
```

`[RelayCommand]` on an `async Task` generates an `IAsyncRelayCommand` (with `IsRunning`); bind `IsBusy` to a spinner. The generated command also disables itself while running.
</details>

---

### Problem 11 — Spot the stale-state bug

`DetailPage` and `DetailViewModel` are registered as `AddSingleton`. Users report the detail page shows the *previous* product's data. Why, and the fix?

<details>
<summary>Solution</summary>

A **singleton** detail page/VM is reused across every navigation, retaining the old product's state. Register them as **transient** so each navigation gets a fresh instance:

```csharp
builder.Services.AddTransient<DetailPage>();
builder.Services.AddTransient<DetailViewModel>();
```

Singleton is for persistent roots/app services; per-item detail pages should be transient.
</details>

---

### Problem 12 — Downsample images in a list

A product list crashes with OOM because each cell loads a full-resolution photo. Fix it.

<details>
<summary>Solution</summary>

```xml
<Image Aspect="AspectFill" HeightRequest="80" WidthRequest="80">
    <Image.Source>
        <UriImageSource Uri="{Binding PhotoUrl}" CacheValidity="7" />
    </Image.Source>
</Image>
```

Decode/scale images to their **display size** (80×80 here) rather than full resolution, and **cache** them (`CacheValidity`). Full-res bitmaps × many cells cause memory spikes/OOM — sizing to the slot is the top fix ([10-Performance.md](10-Performance.md)).
</details>

---

### Problem 13 — Incremental (paged) loading

A list of thousands of items should load 50 at a time as the user scrolls near the end.

<details>
<summary>Solution</summary>

```xml
<CollectionView ItemsSource="{Binding Items}"
                RemainingItemsThreshold="10"
                RemainingItemsThresholdReachedCommand="{Binding LoadMoreCommand}">
    ...
</CollectionView>
```
```csharp
[RelayCommand] private async Task LoadMoreAsync() {
    var next = await _service.GetPageAsync(_page++, 50);
    foreach (var item in next) Items.Add(item);
}
```

`RemainingItemsThreshold` fires the command when the user nears the end — fetch the next page incrementally instead of loading everything up front.
</details>

---

### Problem 14 — Set up Blazor Hybrid

Add a `BlazorWebView` to a MAUI app and let a Blazor component read GPS natively.

<details>
<summary>Solution</summary>

```csharp
// MauiProgram.cs
builder.Services.AddMauiBlazorWebView();
builder.Services.AddSingleton(Geolocation.Default);
```
```xml
<BlazorWebView HostPage="wwwroot/index.html">
    <BlazorWebView.RootComponents>
        <RootComponent Selector="#app" ComponentType="{x:Type local:Routes}" />
    </BlazorWebView.RootComponents>
</BlazorWebView>
```
```razor
@inject IGeolocation Geo
<button @onclick="Where">Locate</button>
@code {
    string? _c;
    async Task Where() { var l = await Geo.GetLocationAsync(); _c = $"{l?.Latitude},{l?.Longitude}"; }
}
```

Hybrid runs the component in-process on the native runtime, so it injects `IGeolocation` and reads real GPS — no sandbox, no JS bridge for device features.
</details>

---

### Problem 15 — Make a ViewModel testable (no UI dependency)

A ViewModel calls `await Application.Current.MainPage.DisplayAlert(...)` for confirmations, which can't be unit-tested. Refactor.

<details>
<summary>Solution</summary>

```csharp
public interface IDialogService { Task<bool> ConfirmAsync(string title, string msg); }

public partial class EditViewModel(IDialogService dialogs, IRepo repo) : ObservableObject {
    [RelayCommand]
    private async Task DeleteAsync() {
        if (await dialogs.ConfirmAsync("Delete?", "Are you sure?"))
            await repo.DeleteAsync(Item.Id);
    }
}
```

Abstract the UI concern (dialog) behind an injected `IDialogService` interface; the real implementation calls `DisplayAlert`, but a test substitutes a fake returning true/false. The ViewModel now has **no UI-type references** — the MVVM rule that makes it testable ([06-MVVM.md](06-MVVM.md)).
</details>
