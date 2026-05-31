# MVVM

## Separating UI from logic

**MVVM** (Model-View-ViewModel) is the dominant architectural pattern for XAML UIs (MAUI, WPF, WinUI). It separates the **View** (XAML — what the user sees), the **ViewModel** (C# — UI state and commands, with no UI-framework references), and the **Model** (your domain data/services). The View **binds** to the ViewModel ([02-XAML.md](02-XAML.md)) rather than calling logic in code-behind — which makes the logic **testable** (no UI required), reusable across views, and cleanly separated from presentation.

```
View (XAML)  ──binds──▶  ViewModel (C#)  ──uses──▶  Model / Services
   ▲                          │
   └──── notifications ───────┘ (INotifyPropertyChanged)
```

---

## The three pieces

- **Model** — domain entities and services (data access, APIs). No UI knowledge.
- **ViewModel** — exposes **properties** the View binds to (data + UI state like `IsBusy`) and **commands** the View invokes; raises change notifications so the View updates. Has no reference to `Button`, `Page`, etc.
- **View** — XAML + minimal code-behind; its `BindingContext` is the ViewModel; bindings wire controls to ViewModel properties/commands.

The hard rule: **the ViewModel must not reference UI types.** That's what lets you unit-test it ([Ch17 Testing](../17-Testing/README.md)) and is the whole point of the separation.

---

## `INotifyPropertyChanged` — the binding engine

Data binding updates the UI when a ViewModel property changes *only if* the ViewModel raises **`PropertyChanged`** (`INotifyPropertyChanged`). Hand-written, this is boilerplate-heavy:

```csharp
public class CounterViewModel : INotifyPropertyChanged {
    private int _count;
    public int Count {
        get => _count;
        set { if (_count != value) { _count = value; OnPropertyChanged(); } }
    }
    public event PropertyChangedEventHandler? PropertyChanged;
    void OnPropertyChanged([CallerMemberName] string? n = null)
        => PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(n));
}
```

Every bindable property needs this notification pattern — tedious to repeat. This is exactly what the **CommunityToolkit.Mvvm** source generators eliminate (next).

---

## CommunityToolkit.Mvvm — eliminate the boilerplate

**CommunityToolkit.Mvvm** uses **source generators** ([Ch01 / CSharpBook reflection](../01-Runtime/README.md)) to generate the `INotifyPropertyChanged` plumbing and commands from attributes at compile time — no runtime reflection, no hand-written boilerplate:

```csharp
public partial class CounterViewModel : ObservableObject {
    [ObservableProperty]
    private int count;                         // generates a public Count property with notification

    [ObservableProperty]
    [NotifyCanExecuteChangedFor(nameof(IncrementCommand))]
    private bool canIncrement = true;

    [RelayCommand(CanExecute = nameof(canIncrement))]
    private void Increment() => Count++;        // generates IncrementCommand : IRelayCommand
}
```

- **`[ObservableProperty]`** on a field generates a full property with `PropertyChanged` notification (note the field is lowercase; the generated property is PascalCase: `count` → `Count`).
- **`[RelayCommand]`** on a method generates an `ICommand` (`IncrementCommand`) you bind to a button — with optional `CanExecute` gating and automatic async handling (`[RelayCommand]` on an `async Task` method gives an `IAsyncRelayCommand` with `IsRunning`).
- **`ObservableObject`** is the base class providing the notification machinery; **`partial`** is required so the generator can extend the class.

This is the standard modern MAUI approach — the toolkit turns ~30 lines of boilerplate into a few attributes, generated at build time (fast, AOT-friendly, no reflection).

---

## Commands — actions without code-behind

A **command** (`ICommand`) is how the View triggers ViewModel behavior without a code-behind event handler. You bind a control's `Command` to a ViewModel command property ([04-Controls.md](04-Controls.md)):

```xml
<Button Text="Save" Command="{Binding SaveCommand}" CommandParameter="{Binding Item}" />
```

```csharp
[RelayCommand]
private async Task SaveAsync() {
    IsBusy = true;
    await _service.SaveAsync(Item);
    IsBusy = false;
}
```

`CanExecute` controls whether the command is enabled (the button auto-disables when it returns false), and async commands expose `IsRunning` for progress UI. Commands keep the action *in the testable ViewModel*, not in the View — the essence of MVVM.

---

## Binding the View to the ViewModel

Set the View's `BindingContext` to its ViewModel — ideally resolved from **DI** ([07-DependencyInjection.md](07-DependencyInjection.md)) so the ViewModel's services are injected:

```csharp
public MainPage(MainViewModel vm) {       // VM injected by DI
    InitializeComponent();
    BindingContext = vm;
}
```

Register both in `MauiProgram`: `builder.Services.AddTransient<MainViewModel>()` and `AddTransient<MainPage>()`. Use **`x:DataType`** on the page/template for **compiled bindings** — faster and compile-checked ([02-XAML.md](02-XAML.md)):

```xml
<ContentPage x:DataType="vm:MainViewModel" ...>
    <Label Text="{Binding Title}" />
</ContentPage>
```

---

## Lists and `ObservableCollection`

For lists that change at runtime, expose an **`ObservableCollection<T>`** — it raises `CollectionChanged`, so a bound `CollectionView` ([04-Controls.md](04-Controls.md)) automatically reflects add/remove without manual refresh:

```csharp
public ObservableCollection<Product> Products { get; } = [];
[RelayCommand] private async Task LoadAsync() {
    Products.Clear();
    foreach (var p in await _service.GetAllAsync()) Products.Add(p);
}
```

A plain `List<T>` won't notify the UI — the list would appear static. Use `ObservableCollection` for mutable bound collections.

---

## Common gotchas

### UI types in the ViewModel

Referencing `Page`/`Button`/`DisplayAlert` in a ViewModel breaks testability and the separation. Abstract UI concerns (dialogs, navigation) behind injected interfaces so the ViewModel stays UI-free.

### Forgetting `partial` with the toolkit

`[ObservableProperty]`/`[RelayCommand]` generate code into the class — it **must** be `partial`, or the generator can't extend it (compile error).

### Property vs field naming with `[ObservableProperty]`

You annotate a **lowercase field** (`count`); the generator creates the **PascalCase property** (`Count`) — bind to `Count`, not `count`. Binding the field name silently fails.

### Binding to `List<T>` for dynamic data

A `List` doesn't notify on add/remove, so the UI won't update. Use `ObservableCollection<T>`.

### No `x:DataType` (slow, unchecked bindings)

Without compiled bindings, binding errors are silent and slow (reflection). Add `x:DataType` for compile-checked, faster bindings (and AOT friendliness).

---

## Summary

- **MVVM** separates **View** (XAML, binds), **ViewModel** (UI-free C#: bindable properties + commands), and **Model** (domain/services) — making logic **testable**, reusable, and decoupled from the UI. The ViewModel must **never** reference UI types.
- Binding updates require **`INotifyPropertyChanged`**; **CommunityToolkit.Mvvm** generates that plumbing (and commands) from **`[ObservableProperty]`**/**`[RelayCommand]`** via source generators — no boilerplate, no reflection, AOT-friendly (class must be `partial`).
- Trigger behavior with **commands** (`ICommand`, with `CanExecute` gating and async `IsRunning`) bound to controls — keeping actions in the testable ViewModel, not code-behind.
- Set the View's **`BindingContext`** to a DI-resolved ViewModel; use **`x:DataType`** for compiled bindings; expose **`ObservableCollection<T>`** for dynamic lists so `CollectionView` auto-updates.

→ Next: [07-DependencyInjection.md](07-DependencyInjection.md)
