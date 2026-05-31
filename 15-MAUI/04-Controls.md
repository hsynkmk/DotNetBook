# Controls

## The built-in control library

MAUI ships a rich set of cross-platform controls, each backed by a native widget on every platform ([01-Overview.md](01-Overview.md)). They fall into a few families: **text/labels**, **input**, **buttons/commands**, **collections**, **layout/container**, and **display/media**. You compose pages from these, bind them to view-model data ([06-MVVM.md](06-MVVM.md)), and style them ([02-XAML.md](02-XAML.md)).

---

## Text and labels

| Control | Purpose |
|---|---|
| `Label` | display read-only text (supports `FormattedText`/spans for rich text) |
| `Span` | a run of styled text inside a `Label`'s `FormattedText` |

```xml
<Label FontSize="18" TextColor="Gray">
    <Label.FormattedText>
        <FormattedString>
            <Span Text="Total: " />
            <Span Text="{Binding Total, StringFormat='{0:C}'}" FontAttributes="Bold" />
        </FormattedString>
    </Label.FormattedText>
</Label>
```

---

## Input controls

| Control | For |
|---|---|
| `Entry` | single-line text input (with `Keyboard`, `IsPassword`) |
| `Editor` | multi-line text input |
| `Picker` | choose one from a list (a dropdown/wheel) |
| `DatePicker` / `TimePicker` | date/time selection |
| `Switch` | on/off toggle |
| `CheckBox` | boolean check |
| `Slider` / `Stepper` | numeric value selection |
| `SearchBar` | search input with a search affordance |

```xml
<Entry Placeholder="Email" Keyboard="Email" Text="{Binding Email}" />
<Switch IsToggled="{Binding NotificationsOn}" />
<Picker Title="Country" ItemsSource="{Binding Countries}" SelectedItem="{Binding Country}" />
```

Input controls **two-way bind** to view-model properties (`{Binding ...}` with `Mode=TwoWay`, which is the default for most input properties), so user edits flow into your model and model changes flow back to the UI.

---

## Buttons and commands

| Control | For |
|---|---|
| `Button` | tappable action (`Clicked` event or `Command`) |
| `ImageButton` | a button whose content is an image |
| `RadioButton` | mutually exclusive choice within a group |

In MVVM, prefer binding a **`Command`** over handling the `Clicked` event in code-behind, so the action lives in the view-model ([06-MVVM.md](06-MVVM.md)):

```xml
<Button Text="Save" Command="{Binding SaveCommand}"
        CommandParameter="{Binding CurrentItem}" />
```

Gestures (tap, swipe, pan, pinch) attach to *any* control via **`GestureRecognizers`** — e.g., make a `Label` or `Image` tappable with a `TapGestureRecognizer` bound to a command.

---

## Collection controls

For lists of data, **`CollectionView`** is the modern, virtualizing list control (replaces the older `ListView`). It binds an `ItemsSource` and renders each item via an `ItemTemplate`:

```xml
<CollectionView ItemsSource="{Binding Products}"
                SelectionMode="Single"
                SelectedItem="{Binding Selected}">
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

- **`CollectionView`** virtualizes (renders only visible items) — essential for large lists, and it supports grids (`ItemsLayout`), grouping, and pull-to-refresh (`RefreshView`).
- **`BindableLayout`** turns a simple layout into a (non-virtualizing) bound list — fine for *small*, fixed collections.
- **`DataTemplate`** + `x:DataType` defines (and compiles) each item's UI; binding an `ObservableCollection` ([06-MVVM.md](06-MVVM.md)) makes add/remove update the UI automatically.

Use `CollectionView` for data lists; avoid the legacy `ListView`/`TableView` for new code except for specific cases (`TableView` for settings-style forms).

---

## Containers and pages

| Type | Purpose |
|---|---|
| `ContentPage` | a single screen of content |
| `Frame` / `Border` | a bordered/rounded container (prefer `Border` in modern MAUI) |
| `ScrollView` | scrollable content ([03-Layouts.md](03-Layouts.md)) |
| `RefreshView` | pull-to-refresh wrapper around scrollable content |
| `TabbedPage` / `FlyoutPage` | multi-section navigation containers (often superseded by Shell — [05-Navigation.md](05-Navigation.md)) |

---

## Display and media

| Control | For |
|---|---|
| `Image` | display images (from resources, files, URLs, streams) |
| `ActivityIndicator` | a spinner for in-progress work |
| `ProgressBar` | determinate progress |
| `BoxView` | a simple colored rectangle (separators, placeholders) |
| `WebView` | embed web content |
| `GraphicsView` | custom 2D drawing via `IDrawable` |
| `BlazorWebView` | host Blazor components ([09-BlazorHybrid.md](09-BlazorHybrid.md)) |

```xml
<Image Source="logo.png" Aspect="AspectFit" />
<ActivityIndicator IsRunning="{Binding IsBusy}" IsVisible="{Binding IsBusy}" />
```

---

## Common gotchas

### Using `ListView` for new code

`CollectionView` is the modern, more flexible, better-performing list. Reserve `ListView`/`TableView` for legacy or settings-form scenarios.

### Binding a `List<T>` instead of `ObservableCollection<T>`

A plain `List` doesn't notify the UI of add/remove, so the list won't update. Use `ObservableCollection<T>` (or raise collection-changed) for dynamic lists ([06-MVVM.md](06-MVVM.md)).

### Missing `x:DataType` on `DataTemplate`

Without `x:DataType`, item bindings use slow reflection and aren't compile-checked. Add `x:DataType="model:Foo"` for compiled bindings (faster, error-caught).

### Handling `Clicked` in code-behind instead of `Command`

Event handlers in code-behind couple UI to logic. Bind a `Command` to keep behavior testable in the view-model ([06-MVVM.md](06-MVVM.md)).

### `Frame` everywhere

`Frame` is heavier; modern MAUI prefers `Border` (with `StrokeShape` for rounded corners) for bordered/rounded containers.

---

## Summary

- MAUI provides a full control library backed by **native widgets**: text (`Label`/`Span`), input (`Entry`, `Editor`, `Picker`, `Switch`, `Slider`, `DatePicker`, …), buttons (`Button`/`ImageButton` — bind a **`Command`** in MVVM), and display/media (`Image`, `ActivityIndicator`, `WebView`, `BlazorWebView`).
- For data lists use **`CollectionView`** (virtualizing, supports grids/grouping/pull-to-refresh) with an **`ItemTemplate`** (`DataTemplate` + **`x:DataType`** for compiled bindings); bind an **`ObservableCollection`** so add/remove updates the UI.
- Input controls **two-way bind** to view-model properties; attach **`GestureRecognizers`** to make any control interactive.
- Prefer modern controls (**`CollectionView`** over `ListView`, **`Border`** over `Frame`) and **commands** over code-behind event handlers for testable MVVM.

→ Next: [05-Navigation.md](05-Navigation.md)
