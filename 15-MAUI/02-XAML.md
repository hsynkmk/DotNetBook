# XAML

## Declarative UI markup

XAML (eXtensible Application Markup Language) is an XML-based language for describing UI declaratively. In MAUI, a page is typically a `.xaml` file (the layout) paired with a `.xaml.cs` **code-behind** (the C# logic). XAML maps elements to .NET objects and attributes to properties — `<Button Text="Save" />` constructs a `Button` and sets its `Text`. It's the idiomatic way to build MAUI UI (though you *can* build everything in C#), and it pairs naturally with **data binding** and **MVVM** ([06-MVVM.md](06-MVVM.md)).

```xml
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="MyApp.MainPage">
    <VerticalStackLayout Padding="20" Spacing="10">
        <Label Text="Hello, MAUI!" FontSize="24" />
        <Button Text="Click me" Clicked="OnClicked" />
    </VerticalStackLayout>
</ContentPage>
```

---

## XAML maps to objects

Every XAML element is a .NET type; every attribute is a property or event. The two are equivalent:

```xml
<Label Text="Hi" TextColor="Blue" />
```
```csharp
var label = new Label { Text = "Hi", TextColor = Colors.Blue };
```

- **`x:Class`** links the XAML to its code-behind partial class.
- **`x:Name`** gives an element a field name so code-behind can reference it.
- **Namespaces** (`xmlns`) import types: the default is MAUI controls, `x:` is XAML language constructs, and you add your own (`xmlns:vm="clr-namespace:MyApp.ViewModels"`) to reference your types.

Because XAML is just object construction, anything you can express in XAML you can express in C# — XAML is a convenience for the *tree-shaped, declarative* nature of UI.

---

## Property element syntax

When a property's value is too complex for an attribute string (a nested object, a collection), use **property element syntax** — `<Type.Property>` child elements:

```xml
<Button Text="Styled">
    <Button.GradientBrush>
        <LinearGradientBrush>
            <GradientStop Color="Red" Offset="0" />
            <GradientStop Color="Blue" Offset="1" />
        </LinearGradientBrush>
    </Button.GradientBrush>
</Button>
```

The element's *content* (children placed directly inside) maps to its `ContentProperty` — e.g., a `ContentPage`'s content is its single child layout, a layout's content is its `Children` collection. This is why you can nest controls directly inside a layout without naming the `Children` property.

---

## Attached properties

**Attached properties** let a child set a value that its *parent* layout interprets — the classic example is `Grid.Row`/`Grid.Column`, where the child declares where it sits but the `Grid` does the positioning ([03-Layouts.md](03-Layouts.md)):

```xml
<Grid RowDefinitions="Auto,*" ColumnDefinitions="*,*">
    <Label Grid.Row="0" Grid.Column="0" Text="Top-left" />
    <Button Grid.Row="1" Grid.Column="1" Text="Bottom-right" />
</Grid>
```

`Grid.Row="0"` is an *attached property*: it's defined by `Grid` but set on the `Label`. The pattern lets layout containers define positioning data that lives on the children they arrange — a powerful XAML idiom you'll use constantly in layouts.

---

## Markup extensions

**Markup extensions** (in `{curly braces}`) compute an attribute value at parse time rather than using a literal string — the most important being **`{Binding}`** for data binding and **`{StaticResource}`** for referencing resources:

```xml
<Label Text="{Binding UserName}" />                       <!-- bind to a view-model property -->
<Label Text="{Binding Price, StringFormat='{0:C}'}" />    <!-- formatted binding -->
<Label TextColor="{StaticResource PrimaryColor}" />       <!-- resource lookup -->
<Button Command="{Binding SaveCommand}" />                <!-- bind a command (MVVM) -->
<Image Source="{x:Static local:Constants.Logo}" />        <!-- static member -->
```

- **`{Binding path}`** — bind to a property on the `BindingContext` (the heart of MVVM — [06-MVVM.md](06-MVVM.md)).
- **`{StaticResource key}`** — look up a resource defined in a `ResourceDictionary`.
- **`{x:Static}`**, **`{OnPlatform}`**, **`{AppThemeBinding}`** — static members, per-platform values, and light/dark theme values respectively.

Markup extensions are what make XAML *dynamic* — without them it would be only static object construction.

---

## Resources and styles

Reusable values (colors, fonts, styles) live in a **`ResourceDictionary`**, scoped to an element, page, or the whole app (`App.xaml`):

```xml
<ContentPage.Resources>
    <Color x:Key="PrimaryColor">#512BD4</Color>
    <Style TargetType="Button">
        <Setter Property="BackgroundColor" Value="{StaticResource PrimaryColor}" />
        <Setter Property="TextColor" Value="White" />
    </Style>
</ContentPage.Resources>
```

A `Style` with a `TargetType` and no `x:Key` is an **implicit style** — applied automatically to every control of that type in scope. Define app-wide styles in `App.xaml` for consistent theming; use `{AppThemeBinding}` for light/dark variants.

---

## Common gotchas

### Forgetting `BindingContext`

`{Binding Foo}` resolves against the element's `BindingContext` (inherited down the tree). If it's not set (or set to the wrong object), bindings silently produce nothing. Set the page's `BindingContext` to its view-model ([06-MVVM.md](06-MVVM.md)).

### Confusing attached vs regular properties

`Grid.Row` is *attached* (defined by `Grid`, set on a child); `Label.Text` is regular. Attached properties always use the `Owner.Property` form. Mixing them up causes parse errors.

### Heavy logic in code-behind

XAML + code-behind can devolve into spaghetti. Prefer MVVM: bind to a view-model and keep code-behind thin (UI-only concerns) — [06-MVVM.md](06-MVVM.md).

### Missing `xmlns` for your own types

Referencing `vm:MainViewModel` without declaring `xmlns:vm="clr-namespace:..."` fails. Import every namespace you use.

### Silent binding failures

A typo'd binding path doesn't throw — it just shows nothing. Enable binding diagnostics / compiled bindings (`x:DataType`) to catch these at build time and for performance.

---

## Summary

- **XAML** declaratively describes the UI tree; each element is a .NET object and each attribute a property/event — equivalent to constructing the same objects in C#. A `.xaml` file pairs with a `.xaml.cs` **code-behind** (linked by `x:Class`).
- Use **property element syntax** (`<Type.Property>`) for complex values; an element's direct children map to its **`ContentProperty`** (why layouts nest controls directly).
- **Attached properties** (`Grid.Row`) let children carry layout data their parent interprets; **markup extensions** (`{Binding}`, `{StaticResource}`, `{OnPlatform}`, `{AppThemeBinding}`) compute attribute values — `{Binding}` is the basis of MVVM.
- Reusable values/**styles** live in a **`ResourceDictionary`** (scoped or app-wide in `App.xaml`); implicit styles (no `x:Key`) auto-apply by `TargetType`.
- Bindings resolve against **`BindingContext`** and fail silently on typos — set the context to a view-model and use compiled bindings (`x:DataType`) to catch errors.

→ Next: [03-Layouts.md](03-Layouts.md)
