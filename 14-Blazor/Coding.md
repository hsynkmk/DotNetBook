# Chapter 14 — Blazor — Coding Problems

Build interactive UI with components, parameters, forms, state, and tests. Each problem has a hidden solution — attempt it first.

---

### Problem 1 — A counter component with automatic re-render

Write a `Counter` component with a count and a button that increments it. Explain why you don't call `StateHasChanged`.

<details>
<summary>Solution</summary>

```razor
<p>Count: @count</p>
<button @onclick="Increment">+1</button>
@code {
    private int count;
    private void Increment() => count++;
}
```

After an event handler returns, Blazor re-renders the component automatically — `StateHasChanged` is only needed for out-of-band changes (timers, background events).
</details>

---

### Problem 2 — Parent/child with `EventCallback`

A `ProductRow` child has a Delete button; the parent removes the item from its list when clicked.

<details>
<summary>Solution</summary>

```razor
@* ProductRow.razor *@
<tr><td>@Product.Name</td>
    <td><button @onclick="() => OnDelete.InvokeAsync(Product)">Delete</button></td></tr>
@code {
    [Parameter] public Product Product { get; set; } = default!;
    [Parameter] public EventCallback<Product> OnDelete { get; set; }
}
```
```razor
@* Parent *@
@foreach (var p in products)
{
    <ProductRow @key="p.Id" Product="p" OnDelete="Remove" />
}
@code { void Remove(Product p) => products.Remove(p); }
```

`EventCallback<Product>` dispatches to the parent and triggers its re-render; `@key` keeps row identity stable across deletes.
</details>

---

### Problem 3 — Reload data when a route parameter changes

A `ProductPage` at `/products/{Id:int}` must load the product whenever `Id` changes, including when navigating `/products/1` → `/products/2`. Which lifecycle method?

<details>
<summary>Solution</summary>

```razor
@page "/products/{Id:int}"
@code {
    [Parameter] public int Id { get; set; }
    private Product? _product;

    protected override async Task OnParametersSetAsync()
        => _product = await Api.GetAsync(Id);   // re-runs on every Id change
}
```

`OnInitializedAsync` runs only once (the component is reused across `/products/1`→`/products/2`), so parameter-dependent loads belong in `OnParametersSetAsync`.
</details>

---

### Problem 4 — A form with validation

Build an `EditForm` for `Person` (`Name` required, `Age` 0–130) that calls `Save` only when valid.

<details>
<summary>Solution</summary>

```razor
<EditForm Model="model" OnValidSubmit="Save">
    <DataAnnotationsValidator />
    <InputText @bind-Value="model.Name" /> <ValidationMessage For="() => model.Name" />
    <InputNumber @bind-Value="model.Age" /> <ValidationMessage For="() => model.Age" />
    <button type="submit">Save</button>
</EditForm>
@code {
    private Person model = new();
    void Save() { /* model is valid */ }
}
public class Person {
    [Required, StringLength(50)] public string Name { get; set; } = "";
    [Range(0, 130)] public int Age { get; set; }
}
```

`<DataAnnotationsValidator />` enforces the attributes; `OnValidSubmit` fires only when validation passes. Re-validate server-side too.
</details>

---

### Problem 5 — Shared state container across unrelated components

Implement a `CartState` service so a navbar badge and a product page both reflect cart count, updating live.

<details>
<summary>Solution</summary>

```csharp
public class CartState {
    private readonly List<Item> _items = [];
    public int Count => _items.Count;
    public event Action? Changed;
    public void Add(Item i) { _items.Add(i); Changed?.Invoke(); }
}
builder.Services.AddScoped<CartState>();   // per circuit (Server) / per session (WASM)
```
```razor
@* Navbar badge *@
@inject CartState Cart
@implements IDisposable
<span>Cart: @Cart.Count</span>
@code {
    protected override void OnInitialized() => Cart.Changed += StateHasChanged;
    public void Dispose() => Cart.Changed -= StateHasChanged;
}
```

DI container + `Changed` event + subscribe/unsubscribe. **Scoped** (per-user), not singleton. Unsubscribe in `Dispose` to avoid a circuit leak.
</details>

---

### Problem 6 — JS interop to copy text to the clipboard

Add a "Copy" button that copies text via the browser clipboard API, using isolation-safe interop.

<details>
<summary>Solution</summary>

```razor
@inject IJSRuntime JS
<button @onclick="Copy">Copy</button>
@code {
    [Parameter] public string Text { get; set; } = "";
    async Task Copy() => await JS.InvokeVoidAsync("navigator.clipboard.writeText", Text);
}
```

`InvokeVoidAsync` calls a JS function (always async — it crosses the wire in Server). For component-owned JS, prefer an ES module via `import` → `IJSObjectReference` (disposed in `DisposeAsync`).
</details>

---

### Problem 7 — Chart init in the right lifecycle method

A component must initialize a JS chart on a `<canvas>` element. Where do you call the JS, and why not `OnInitialized`?

<details>
<summary>Solution</summary>

```razor
<canvas @ref="_canvas"></canvas>
@code {
    private ElementReference _canvas;
    private IJSObjectReference? _module;

    protected override async Task OnAfterRenderAsync(bool firstRender) {
        if (firstRender) {
            _module = await JS.InvokeAsync<IJSObjectReference>("import", "./js/chart.js");
            await _module.InvokeVoidAsync("init", _canvas, _data);
        }
    }
    public async ValueTask DisposeAsync() { if (_module is not null) await _module.DisposeAsync(); }
}
```

The DOM element (`ElementReference`) only exists **after render**, and JS interop can't run during prerender. So DOM-touching interop goes in `OnAfterRenderAsync(firstRender)`, not `OnInitialized`.
</details>

---

### Problem 8 — Avoid the prerender double-fetch

`OnInitializedAsync` fetches data, but with prerendering it runs twice. Fix it with `PersistentComponentState`.

<details>
<summary>Solution</summary>

```csharp
[Inject] PersistentComponentState State { get; set; } = default!;
private PersistingComponentStateSubscription _sub;
private Data _data = default!;

protected override async Task OnInitializedAsync() {
    _sub = State.RegisterOnPersisting(() => { State.PersistAsJson("data", _data); return Task.CompletedTask; });
    if (!State.TryTakeFromJson<Data>("data", out var restored))
        _data = await Api.LoadAsync();   // prerender pass fetches
    else
        _data = restored!;               // interactive pass reuses
}
public void Dispose() => _sub.Dispose();
```

The prerender pass fetches and persists; the interactive pass restores from persisted state — one fetch, no flicker.
</details>

---

### Problem 9 — Virtualize a 10,000-row list

Render a huge list efficiently so the DOM holds only visible rows.

<details>
<summary>Solution</summary>

```razor
<Virtualize Items="bigList" Context="item" ItemSize="32">
    <div class="row">@item.Name</div>
</Virtualize>

@* Or page from the backend without loading everything: *@
<Virtualize ItemsProvider="Load" Context="item">
    <div>@item.Name</div>
</Virtualize>
@code {
    async ValueTask<ItemsProviderResult<Item>> Load(ItemsProviderRequest r) {
        var (items, total) = await Api.PageAsync(r.StartIndex, r.Count);
        return new ItemsProviderResult<Item>(items, total);
    }
}
```

`<Virtualize>` renders only the visible window (+buffer). The `ItemsProvider` form pages from the backend, never holding the full dataset in memory.
</details>

---

### Problem 10 — Conditional UI with `AuthorizeView`

Show an Admin panel only to users in the `Admin` role, with a fallback for others — and note the security caveat.

<details>
<summary>Solution</summary>

```razor
<AuthorizeView Roles="Admin">
    <Authorized><AdminPanel /></Authorized>
    <NotAuthorized>You lack permission.</NotAuthorized>
</AuthorizeView>
```

This only hides/shows UI — it's **not** security. The server-side API/handler that the AdminPanel calls must enforce authorization independently ([Ch10](../10-Identity/README.md)).
</details>

---

### Problem 11 — Debounce a search box (Interactive Server)

A search-as-you-type input on Interactive Server floods the circuit. Debounce so the query runs ~300ms after typing stops.

<details>
<summary>Solution</summary>

```razor
<input @bind="term" @bind:event="oninput" />
@code {
    private string term = "";
    private CancellationTokenSource? _cts;

    // Hook via a property setter or @oninput handler that debounces:
    async Task OnInput(ChangeEventArgs e) {
        term = e.Value?.ToString() ?? "";
        _cts?.Cancel(); _cts = new CancellationTokenSource();
        var token = _cts.Token;
        try {
            await Task.Delay(300, token);     // wait for typing to settle
            results = await Api.SearchAsync(term, token);
            StateHasChanged();
        } catch (TaskCanceledException) { /* superseded by a newer keystroke */ }
    }
}
```

Each keystroke cancels the previous pending delay; only the last one (after 300ms of quiet) runs the query — drastically reducing circuit round-trips ([10-Performance.md](10-Performance.md)).
</details>

---

### Problem 12 — Test a component with bUnit

Write a bUnit test that renders `Counter`, clicks the button, and asserts the count.

<details>
<summary>Solution</summary>

```csharp
public class CounterTests : TestContext {
    [Fact]
    public void Clicking_increments() {
        var cut = RenderComponent<Counter>();
        cut.Find("button").Click();
        cut.Find("p").MarkupMatches("<p>Count: 1</p>");
    }
}
```

`RenderComponent` renders in-memory; `Find(...).Click()` triggers the handler and re-renders; `MarkupMatches` asserts semantically.
</details>

---

### Problem 13 — Test an async component with a mocked service

`ProductPage` loads a product in `OnInitializedAsync` from `IProductApi`. Test it with a fake API.

<details>
<summary>Solution</summary>

```csharp
[Fact]
public async Task Renders_loaded_product() {
    var api = Substitute.For<IProductApi>();
    api.GetAsync(1).Returns(new Product { Name = "Widget" });
    Services.AddSingleton(api);

    var cut = RenderComponent<ProductPage>(p => p.Add(c => c.Id, 1));
    cut.WaitForState(() => cut.Find("h1").TextContent == "Widget");
    cut.Find("h1").TextContent.Should().Be("Widget");
}
```

Register the mock in `Services` before rendering; `WaitForState` waits for the async lifecycle to complete and re-render before asserting.
</details>

---

### Problem 14 — Spot the leak

```razor
@inject NavigationManager Nav
@code {
    protected override void OnInitialized()
        => Nav.LocationChanged += (_, __) => DoSomething();
}
```

What's wrong, and how do you fix it?

<details>
<summary>Solution</summary>

The component subscribes to `LocationChanged` but never unsubscribes — it leaks (kept alive by the event), severe in Interactive Server circuits. Fix:

```razor
@implements IDisposable
@code {
    protected override void OnInitialized() => Nav.LocationChanged += OnLocationChanged;
    private void OnLocationChanged(object? s, LocationChangedEventArgs e) => DoSomething();
    public void Dispose() => Nav.LocationChanged -= OnLocationChanged;
}
```

Use a named handler and detach in `Dispose` ([07-Lifecycle.md](07-Lifecycle.md)).
</details>

---

### Problem 15 — Choose render modes for a mixed app

For an app with (a) a public marketing homepage, (b) an internal admin dashboard on a reliable network, (c) a customer portal needing offline support — pick a render mode for each and justify.

<details>
<summary>Solution</summary>

- **(a) Marketing homepage → Static SSR**: no interactivity, fastest first paint, SEO-friendly.
- **(b) Admin dashboard → Interactive Server**: reliable network makes round-trip latency acceptable; tiny download, full server access, code stays on the server, low concurrency.
- **(c) Customer portal → Interactive WebAssembly (or Auto)**: needs offline + no per-interaction latency; WASM runs client-side. **Auto** is a good choice if a fast first visit matters (Server first, then WASM once cached).

The whole point of modern Blazor's unified model is mixing modes per component/page ([02-RenderModes.md](02-RenderModes.md)).
</details>
