# Navigation

## Moving between pages

A multi-screen app needs **navigation** — moving between pages, passing data, going back. MAUI offers two main models: **Shell** (a structured, URI-based navigation host that's the recommended default) and **`NavigationPage`** stack navigation (lower-level, push/pop). Plus **modal** pages for interruptions. Choosing Shell up front gives you a consistent app structure (flyout, tabs, routes) with far less boilerplate.

---

## Shell — the recommended host

**Shell** (`AppShell`) provides app-wide navigation structure: a flyout menu, bottom tabs, top tabs, and **URI-based routing**. You declare the structure in `AppShell.xaml` and navigate by route string:

```xml
<!-- AppShell.xaml -->
<Shell xmlns="..." x:Class="MyApp.AppShell">
    <FlyoutItem Title="Home" Icon="home.png">
        <ShellContent Route="home" ContentTemplate="{DataTemplate views:HomePage}" />
    </FlyoutItem>
    <TabBar>
        <Tab Title="Products" Icon="box.png">
            <ShellContent Route="products" ContentTemplate="{DataTemplate views:ProductListPage}" />
        </Tab>
        <Tab Title="Profile" Icon="user.png">
            <ShellContent Route="profile" ContentTemplate="{DataTemplate views:ProfilePage}" />
        </Tab>
    </TabBar>
</Shell>
```

Navigate with `GoToAsync` using routes (absolute or relative):

```csharp
await Shell.Current.GoToAsync("products");          // navigate to a registered route
await Shell.Current.GoToAsync("product/details");   // push onto the stack
await Shell.Current.GoToAsync("..");                // go back (pop)
```

Shell gives you flyout/tabs, a back stack, deep linking, and route-based navigation out of the box — the modern way to structure a MAUI app.

---

## Registering routes and detail pages

Pages reachable only via navigation (not part of the visible flyout/tabs, like a detail page) are **registered** with a route in `AppShell`'s code-behind:

```csharp
public AppShell() {
    InitializeComponent();
    Routing.RegisterRoute("product/details", typeof(ProductDetailPage));
}
```

Then `GoToAsync("product/details")` instantiates and pushes that page. This separates the *navigable surface* (declared in XAML) from *detail routes* (registered in code), keeping deep links and the back stack coherent.

---

## Passing data with query parameters

Shell passes data between pages via **query parameters** in the route (primitives) or a passed dictionary (objects):

```csharp
// Navigate with parameters:
await Shell.Current.GoToAsync($"product/details?id={product.Id}");

// Or pass objects via a dictionary:
await Shell.Current.GoToAsync("product/details", new Dictionary<string, object> {
    ["Product"] = product
});
```

The destination page (or its view-model) receives them via `[QueryProperty]` or by implementing `IQueryAttributable`:

```csharp
[QueryProperty(nameof(ProductId), "id")]
public partial class ProductDetailPage : ContentPage {
    public string ProductId { set { /* load product by id */ } }
}

// Or, more flexibly, on a view-model:
public class DetailViewModel : IQueryAttributable {
    public void ApplyQueryAttributes(IDictionary<string, object> query) {
        var product = (Product)query["Product"];
    }
}
```

Prefer passing an **id** and reloading (robust to process restarts / deep links) over passing whole objects, unless the object is transient UI state.

---

## Stack navigation (`NavigationPage`)

Without Shell, navigation uses a **`NavigationPage`** stack — `PushAsync`/`PopAsync`:

```csharp
await Navigation.PushAsync(new DetailPage());     // push a page
await Navigation.PopAsync();                       // go back
await Navigation.PopToRootAsync();                 // back to the first page
```

This is the lower-level model (and how Shell works under the hood). Use it for simple apps or fine-grained stack control, but Shell's routing is usually cleaner for anything with multiple sections.

---

## Modal navigation

**Modal** pages interrupt the flow — a full-screen dialog the user must complete or cancel (login, a form). They're pushed onto a *separate* modal stack:

```csharp
await Navigation.PushModalAsync(new LoginPage());   // modal (no automatic back button)
await Navigation.PopModalAsync();                    // dismiss
// With Shell:
await Shell.Current.GoToAsync("login");              // a route can be presented modally
```

Modal pages don't get the navigation bar's back button by default — you provide explicit done/cancel actions. Use modals for tasks that must be resolved before returning, not general navigation.

---

## DI and navigation

Register pages and view-models in DI ([07-DependencyInjection.md](07-DependencyInjection.md)) so navigation resolves them with their dependencies injected. Shell's route resolution and constructor injection work together — a navigated-to page gets its view-model (and the view-model's services) from the container, rather than `new`-ing them with hardcoded dependencies.

---

## Common gotchas

### Forgetting to register a detail route

`GoToAsync("some/route")` to an unregistered route fails. Register non-flyout/tab pages with `Routing.RegisterRoute`.

### Passing large objects through navigation

Passing whole objects is fragile (lost on process restart, awkward for deep links). Pass an **id** and reload on the destination — more robust, and works for deep links.

### Mixing Shell and manual `NavigationPage`

Combining Shell with ad-hoc `NavigationPage` stacks causes confusing back-stack behavior. Pick Shell *or* manual navigation and use it consistently.

### Modal without an exit affordance

Modal pages lack an automatic back button — if you don't add a done/cancel control, the user is stuck. Always provide explicit dismissal.

### Not using DI for pages/view-models

`new DetailPage()` hardcodes dependencies and bypasses DI. Register pages/view-models and let the container inject them, so navigation yields fully-wired instances.

---

## Summary

- MAUI navigation: **Shell** (recommended) provides flyout/tabs + **URI-based routing** (`GoToAsync`), a back stack, and deep linking from one declarative `AppShell`; **`NavigationPage`** is the lower-level push/pop stack; **modal** pages interrupt the flow.
- Declare the navigable surface (flyout/tabs/`ShellContent`) in `AppShell.xaml`; **register detail routes** in code (`Routing.RegisterRoute`).
- Pass data via **query parameters** (`?id=...`) or a dictionary, received with **`[QueryProperty]`**/`IQueryAttributable` — prefer passing an **id** and reloading over whole objects (robust to restarts/deep links).
- **Modal** navigation (`PushModalAsync`) is for must-complete tasks — provide explicit done/cancel (no automatic back button).
- Resolve pages/view-models from **DI** so navigation yields fully-injected instances; don't mix Shell with manual stacks.

→ Next: [06-MVVM.md](06-MVVM.md)
