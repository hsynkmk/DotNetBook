# WinForms

## The pragmatic veteran

**WinForms** (Windows Forms) is the oldest and simplest .NET desktop UI model — and still one of the most *productive* for the right job. Its model is straightforward: drag controls from a toolbox onto a form in the **visual designer**, set properties, and wire up **event handlers** in code-behind. No XAML, no binding ceremony, no MVVM required. For internal tools, utilities, prototypes, and data-entry apps, WinForms gets you a working UI faster than anything else — and it's fully supported on modern .NET (.NET 10), rock-solid after two decades.

```csharp
public partial class MainForm : Form {
    public MainForm() {
        InitializeComponent();          // designer-generated control setup
    }
    private void saveButton_Click(object sender, EventArgs e) {
        MessageBox.Show($"Hello, {nameButton.Text}!");
    }
}
```

---

## The designer + event model

WinForms' defining characteristic is the **WYSIWYG designer**: you visually place `Button`, `TextBox`, `DataGridView`, `Label`, etc. on a `Form`, and Visual Studio generates the layout code in a `*.Designer.cs` partial file. You then handle **events** (`Click`, `TextChanged`, `SelectedIndexChanged`) in code-behind. This event-driven, control-centric model is immediate and easy to learn — there's no abstraction between you and the controls.

The trade-off: logic tends to live in **code-behind event handlers**, coupling UI and behavior. That's fine for small tools but doesn't scale to large apps as cleanly as MVVM (next section).

---

## Controls and data binding

WinForms has a solid control library, and a standout for data apps is **`DataGridView`** — a powerful grid for displaying/editing tabular data with minimal code. WinForms also supports **data binding** (simpler than WPF's): bind controls to a `BindingSource` over a `BindingList<T>`/`DataTable`, and the grid/controls reflect the data:

```csharp
var binding = new BindingSource { DataSource = new BindingList<Product>(products) };
dataGridView1.DataSource = binding;
nameTextBox.DataBindings.Add("Text", binding, nameof(Product.Name));
```

`BindingList<T>` (and `INotifyPropertyChanged` on items) propagates changes to bound controls. It's less rich than WPF binding (no templates/converters pipeline of the same depth), but more than enough for forms and grids.

---

## Modern .NET WinForms

On modern .NET, WinForms gains improvements over the old .NET Framework version: better high-DPI support, performance gains, and you can use the **Generic Host / DI** ([Ch03](../03-HostingAndDI/README.md)) to bootstrap services and resolve forms from the container — bringing dependency injection to WinForms:

```csharp
var host = Host.CreateApplicationBuilder().Build();   // configure services first
ApplicationConfiguration.Initialize();
Application.Run(host.Services.GetRequiredService<MainForm>());
```

This lets even a WinForms app use injected services (data access, HTTP clients, options) instead of `new`-ing dependencies — modernizing the architecture while keeping the productive UI model.

---

## When WinForms is the right call

- **Internal line-of-business tools** — admin panels, data-entry, report viewers where speed-to-build matters more than visual polish.
- **Utilities and prototypes** — quick UIs to wrap a script or demonstrate an idea.
- **Data-heavy CRUD** — `DataGridView` + binding makes tabular editing trivial.
- **Maintaining existing WinForms apps** — fully supported; no need to rewrite a working app.

It's *not* the choice for modern consumer-facing apps (no Fluent design, dated look), heavily custom/animated UIs (WPF/WinUI templating wins), or cross-platform (Windows-only — use Avalonia/MAUI — [01-Comparison.md](01-Comparison.md)).

---

## Applying MVVM-ish patterns (optional)

Although WinForms is event-centric, you *can* improve testability by pushing logic out of event handlers into separate classes (a "presenter" / view-model), keeping the form thin. Full MVVM with binding is possible but less natural than in WPF/WinUI. For small tools, the straightforward event model is fine; for larger WinForms apps, separating logic into presenters (the MVP pattern) aids maintainability and testing ([Ch17 Testing](../17-Testing/README.md)).

---

## Common gotchas

### Blocking the UI thread

Long work in an event handler freezes the UI (WinForms is single-UI-threaded). Use `async`/`await` for I/O and `Task.Run` for CPU-bound work, and marshal UI updates back via the form's synchronization context (await resumes on the UI thread, or use `Control.Invoke`).

### Cross-thread control access

Touching a control from a non-UI thread throws (or corrupts). Use `Control.Invoke`/`BeginInvoke` (or `await`, which returns to the UI context) to update controls from background work.

### All logic in code-behind

Stuffing business logic into `*_Click` handlers makes it untestable and tangled. Push logic into separate classes/presenters; keep the form focused on UI.

### High-DPI/scaling issues

Older WinForms apps blur on high-DPI displays. Modern .NET WinForms has improved DPI support — enable per-monitor DPI awareness in the app manifest/config.

### Expecting a modern look

WinForms looks classic, not Fluent. If visual modernity matters, choose WinUI 3/WPF; WinForms trades polish for speed-of-development.

---

## Summary

- **WinForms** is the oldest, simplest .NET desktop model — a **visual designer** + **event handlers** — and the most **productive** for internal tools, utilities, prototypes, and data-entry/CRUD apps; fully supported on modern .NET.
- Its **designer + event** model is immediate (no XAML/MVVM required) but tends to put logic in **code-behind**; `DataGridView` + simple **data binding** (`BindingSource`/`BindingList<T>`) make tabular data apps trivial.
- Modern .NET WinForms adds better **high-DPI**, performance, and **Generic Host / DI** support — so you can inject services and resolve forms from the container.
- Choose it for **speed and simplicity** (not modern visuals or cross-platform); push logic into **presenters** for larger apps' testability, and keep the **UI thread** unblocked (`async`/`Invoke`).

→ Next: [05-Avalonia.md](05-Avalonia.md)
