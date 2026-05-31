# Chapter 16 — Desktop — Coding Problems

Small exercises across WPF, WinUI 3, WinForms, and Avalonia. Each has a hidden solution — attempt it first.

---

### Problem 1 — A WPF MVVM window

Build a WPF window with a bound greeting and a button command, using CommunityToolkit.Mvvm.

<details>
<summary>Solution</summary>

```csharp
public partial class MainViewModel : ObservableObject {
    [ObservableProperty] private string greeting = "Hello";
    [RelayCommand] private void Click() => Greeting = "Clicked!";
}
```
```xml
<Window x:Class="MyApp.MainWindow" ... >
    <StackPanel Margin="16">
        <TextBlock Text="{Binding Greeting}" FontSize="20" />
        <Button Content="Click" Command="{Binding ClickCommand}" />
    </StackPanel>
</Window>
```
```csharp
public MainWindow() { InitializeComponent(); DataContext = new MainViewModel(); }
```

Same MVVM/toolkit as MAUI ([Ch15 §06](../15-MAUI/06-MVVM.md)); WPF uses `DataContext` (vs MAUI's `BindingContext`).
</details>

---

### Problem 2 — A custom WPF dependency property

Add a bindable `Count` dependency property to a custom control with a change callback.

<details>
<summary>Solution</summary>

```csharp
public class CounterControl : Control {
    public static readonly DependencyProperty CountProperty =
        DependencyProperty.Register(nameof(Count), typeof(int), typeof(CounterControl),
            new PropertyMetadata(0, OnCountChanged));

    public int Count { get => (int)GetValue(CountProperty); set => SetValue(CountProperty, value); }

    static void OnCountChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
        => ((CounterControl)d).UpdateVisual();
    void UpdateVisual() { /* react to new value */ }
}
```

Dependency properties are for **controls** (bindable/animatable targets), not view-model properties. The metadata supplies the default and change callback.
</details>

---

### Problem 3 — WPF live-updating TextBox binding

Bind a `TextBox` so the source updates on every keystroke (not on focus loss).

<details>
<summary>Solution</summary>

```xml
<TextBox Text="{Binding SearchTerm, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged}" />
```

By default WPF updates the bound source on **focus loss**; `UpdateSourceTrigger=PropertyChanged` makes it update per keystroke — needed for live search/validation.
</details>

---

### Problem 4 — WPF style with a trigger

Style all buttons blue, turning brighter on mouse-over, with no code-behind.

<details>
<summary>Solution</summary>

```xml
<Style TargetType="Button">
    <Setter Property="Background" Value="SteelBlue" />
    <Setter Property="Foreground" Value="White" />
    <Style.Triggers>
        <Trigger Property="IsMouseOver" Value="True">
            <Setter Property="Background" Value="DodgerBlue" />
        </Trigger>
    </Style.Triggers>
</Style>
```

A keyless `Style` with a `TargetType` is implicit (applies to all buttons in scope). The `Trigger` changes a property declaratively on a condition — no code-behind.
</details>

---

### Problem 5 — WinUI 3 compiled binding

Bind a TextBlock and ListView using `{x:Bind}` with the correct mode.

<details>
<summary>Solution</summary>

```xml
<TextBlock Text="{x:Bind ViewModel.Title, Mode=OneWay}" />
<ListView ItemsSource="{x:Bind ViewModel.Items}" />
```
```csharp
public sealed partial class MainPage : Page {
    public MainViewModel ViewModel { get; } = new();
    public MainPage() => InitializeComponent();
}
```

`{x:Bind}` resolves against the page code-behind (so expose `ViewModel` there), is compiled/type-checked, and defaults to `OneTime` — set `Mode=OneWay` for updating values.
</details>

---

### Problem 6 — WinUI 3 NavigationView shell

Set up a `NavigationView` with two items and a content frame.

<details>
<summary>Solution</summary>

```xml
<NavigationView x:Name="Nav" PaneTitle="My App" SelectionChanged="Nav_SelectionChanged">
    <NavigationView.MenuItems>
        <NavigationViewItem Content="Home" Icon="Home" Tag="home" />
        <NavigationViewItem Content="Settings" Icon="Setting" Tag="settings" />
    </NavigationView.MenuItems>
    <Frame x:Name="ContentFrame" />
</NavigationView>
```
```csharp
void Nav_SelectionChanged(NavigationView s, NavigationViewSelectionChangedEventArgs e) {
    var tag = (e.SelectedItem as NavigationViewItem)?.Tag?.ToString();
    if (tag == "home") ContentFrame.Navigate(typeof(HomePage));
    else if (tag == "settings") ContentFrame.Navigate(typeof(SettingsPage));
}
```

`NavigationView` is the Fluent nav-pane pattern — the modern Windows 11 app shell with minimal effort.
</details>

---

### Problem 7 — WinForms async without freezing

A WinForms button loads data. Keep the UI responsive and update a grid on completion.

<details>
<summary>Solution</summary>

```csharp
private async void loadButton_Click(object sender, EventArgs e) {
    loadButton.Enabled = false;
    try {
        var data = await _service.GetDataAsync();   // frees UI thread; resumes on it
        dataGridView1.DataSource = data;            // safe — back on UI thread after await
    }
    finally { loadButton.Enabled = true; }
}
```

`await` releases the UI thread during the I/O and resumes on the UI synchronization context, so updating the grid afterward is thread-safe. Never call a synchronous `GetData()` here — it would freeze the form.
</details>

---

### Problem 8 — WinForms cross-thread update

A background thread produces progress; update a label safely.

<details>
<summary>Solution</summary>

```csharp
void OnProgress(int percent) {
    if (progressLabel.InvokeRequired)
        progressLabel.Invoke(() => progressLabel.Text = $"{percent}%");
    else
        progressLabel.Text = $"{percent}%";
}
```

Touching a control from a non-UI thread throws; `Control.Invoke` marshals the update onto the UI thread. (Using `await`/`IProgress<T>` is cleaner — `Progress<T>` raises its callback on the captured UI context automatically.)
</details>

---

### Problem 9 — WinForms DataGridView virtual mode

Display a million rows without holding them all in memory.

<details>
<summary>Solution</summary>

```csharp
dataGridView1.VirtualMode = true;
dataGridView1.RowCount = 1_000_000;
dataGridView1.CellValueNeeded += (s, e) => {
    e.Value = _dataSource.GetCell(e.RowIndex, e.ColumnIndex);   // fetch on demand
};
```

Virtual mode asks for cell values only as they become visible (`CellValueNeeded`), so the grid never materializes all million rows — essential for huge data sets.
</details>

---

### Problem 10 — Avalonia window (cross-platform)

Build an Avalonia window with the same MVVM view-model from Problem 1.

<details>
<summary>Solution</summary>

```xml
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:vm="clr-namespace:MyApp.ViewModels"
        x:Class="MyApp.MainWindow" x:DataType="vm:MainViewModel">
    <StackPanel Margin="16" Spacing="8">
        <TextBlock Text="{Binding Greeting}" FontSize="20" />
        <Button Content="Click" Command="{Binding ClickCommand}" />
    </StackPanel>
</Window>
```

The same `MainViewModel` (CommunityToolkit.Mvvm) works unchanged — view-models are portable. Note Avalonia's own `xmlns` and `x:DataType` for compiled bindings. The app runs on Windows, macOS, and Linux.
</details>

---

### Problem 11 — Choose the framework

For each, pick a desktop framework and justify: (a) a Linux+Windows+macOS engineering tool, (b) a quick internal HR data-entry form, (c) a new consumer Windows 11 app for the Store, (d) a large existing WPF LOB app needing new screens.

<details>
<summary>Solution</summary>

- **(a) Cross-platform tool → Avalonia**: WPF-style XAML/MVVM that runs on all three OSes (consistent self-rendered UI). MAUI is more mobile-oriented.
- **(b) Quick internal form → WinForms**: fastest to build with the designer + `DataGridView`; polish doesn't matter.
- **(c) New Store consumer app → WinUI 3** (+ MSIX/Store): modern Fluent look, Store distribution, Microsoft's strategic direction.
- **(d) Extend existing WPF app → WPF**: stay in the framework; mature, and your view-models/controls already exist.
</details>

---

### Problem 12 — Publish self-contained single-file

Produce a single .exe that runs on a Windows machine without .NET installed.

<details>
<summary>Solution</summary>

```bash
dotnet publish -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true
```

`--self-contained` bundles the runtime (no .NET install needed); `PublishSingleFile` packs it into one executable. Add `-p:PublishReadyToRun=true` for faster startup. (Don't expect Native AOT for a XAML UI app — reflection-heavy; use R2R instead.)
</details>

---

### Problem 13 — Fix an event-handler leak

```csharp
public partial class DetailView : UserControl {
    public DetailView(AppState state) {
        state.Changed += OnStateChanged;   // never unsubscribed
    }
}
```

`AppState` is a long-lived singleton; many `DetailView`s are created and discarded. What's wrong and the fix?

<details>
<summary>Solution</summary>

Each `DetailView` subscribes to the singleton's `Changed` event but never unsubscribes — the singleton holds a reference to every view ever created, so none are garbage-collected (a leak). Fix by unsubscribing when the view goes away:

```csharp
public DetailView(AppState state) {
    _state = state;
    Loaded   += (_, __) => _state.Changed += OnStateChanged;
    Unloaded += (_, __) => _state.Changed -= OnStateChanged;
}
```

Or use a **weak event** (WPF's `WeakEventManager`) so the subscription doesn't root the view ([06-Performance.md](06-Performance.md)).
</details>

---

### Problem 14 — Decode an image to display size (WPF)

A list shows 64×64 thumbnails but loads full-resolution photos, spiking memory. Fix it.

<details>
<summary>Solution</summary>

```csharp
var bmp = new BitmapImage();
bmp.BeginInit();
bmp.UriSource = new Uri(path);
bmp.DecodePixelWidth = 64;     // decode to display size, not full resolution
bmp.CacheOption = BitmapCacheOption.OnLoad;
bmp.EndInit();
bmp.Freeze();                  // immutable → shareable, no change tracking
```

`DecodePixelWidth` decodes the image at the target size (huge memory saving vs full-res), and `Freeze()` makes it a shareable immutable resource ([06-Performance.md](06-Performance.md)).
</details>

---

### Problem 15 — Share a view-model across WPF and Avalonia

You have a WPF app and want an Avalonia (Linux/macOS) version. What can you reuse?

<details>
<summary>Solution</summary>

Reuse the **entire view-model and service layer** — `ObservableObject` view-models (CommunityToolkit.Mvvm), commands, `ObservableCollection`s, DI registrations, and all business logic. Only rewrite the **View** (XAML) for Avalonia's dialect/controls:

```
MyApp.Core (netstandard / net10.0)   ← ViewModels + Services + Models  (shared)
MyApp.Wpf                            ← WPF Views (XAML)
MyApp.Avalonia                       ← Avalonia Views (XAML)
```

Because WPF and Avalonia share XAML + binding + MVVM, the logic layer is portable — the framework choice isn't a deep lock-in ([01-Comparison.md](01-Comparison.md)).
</details>
