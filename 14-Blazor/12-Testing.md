# Testing Components with bUnit

## Unit-testing the UI

Components contain real logic — conditional rendering, event handling, parameter reactions, state changes — and that logic deserves tests. **bUnit** is the de-facto Blazor component testing library: it renders a component in-memory (no browser, no server), lets you inspect the rendered markup, trigger events, pass parameters, inject services, and assert on the result. It runs on a standard test runner (xUnit/NUnit/MSTest — [Ch17 Testing](../17-Testing/README.md)) and is fast because there's no real DOM or network.

```csharp
public class CounterTests : TestContext {
    [Fact]
    public void Clicking_increments_count() {
        var cut = RenderComponent<Counter>();          // render the component
        cut.Find("button").Click();                    // trigger the click event
        cut.Find("h1").MarkupMatches("<h1>Count: 1</h1>");  // assert rendered output
    }
}
```

---

## The `TestContext` and rendering

bUnit centers on **`TestContext`** (inherit it or `using var ctx = new TestContext();`). It's the render host and the DI container for the test:

```csharp
var cut = RenderComponent<Greeting>(parameters => parameters
    .Add(p => p.Name, "Ada")                          // strongly-typed parameter
    .Add(p => p.Count, 3)
    .AddCascadingValue(theme));                        // cascading value

cut.Find("p").TextContent.Should().Be("Hello, Ada!"); // query + assert
```

`RenderComponent<T>` returns an `IRenderedComponent<T>` (`cut` = "component under test") exposing the rendered markup, the component `Instance`, and query methods. Parameters and cascading values are supplied via the strongly-typed builder.

---

## Querying and asserting

bUnit queries the rendered markup with CSS selectors and offers markup-aware assertions:

```csharp
cut.Find("button")                 // first match (throws if none)
cut.FindAll("li")                  // all matches
cut.Find(".error").TextContent     // text
cut.MarkupMatches("<div>...</div>")        // semantic HTML comparison (ignores insignificant whitespace)
cut.Find("input").GetAttribute("value")
```

`MarkupMatches` does a **semantic** HTML comparison (whitespace/attribute-order insensitive), which is more robust than raw string equality. Combine with FluentAssertions/Shouldly ([Ch17 §04](../17-Testing/README.md)) for readable assertions.

---

## Triggering events

Simulate user interaction by dispatching events to elements — bUnit runs the component's handler and re-renders:

```csharp
cut.Find("button").Click();
cut.Find("input").Change("new value");        // @onchange
cut.Find("input").Input("typing");            // @oninput
cut.Find("form").Submit();
cut.Find("button").Click(new MouseEventArgs { CtrlKey = true });  // with event args
```

After the event, assert the new rendered state. This tests the full loop: event → handler → state change → re-render → markup ([04-EventHandling.md](04-EventHandling.md)).

---

## Injecting services and mocking dependencies

Because `TestContext` is a DI container, register real or fake services the component depends on:

```csharp
var fakeApi = Substitute.For<IProductApi>();      // NSubstitute
fakeApi.GetAsync(1).Returns(new Product { Name = "Widget" });
Services.AddSingleton(fakeApi);                    // register before rendering

var cut = RenderComponent<ProductPage>(p => p.Add(c => c.Id, 1));
await cut.InvokeAsync(() => { });                  // let async lifecycle settle
cut.Find("h1").TextContent.Should().Be("Widget");
```

Register mocks (Moq/NSubstitute — [Ch17 §03](../17-Testing/README.md)) in `Services` so the component's `OnInitializedAsync` data fetch hits the fake. This isolates the component from real I/O — the essence of a unit test.

---

## Testing async lifecycle and re-renders

Components with async lifecycle ([07-Lifecycle.md](07-Lifecycle.md)) need bUnit to await the rendering settle:

```csharp
var cut = RenderComponent<DataList>();
cut.WaitForState(() => cut.FindAll("li").Count > 0);   // wait until data loaded + rendered
// or:
cut.WaitForAssertion(() => cut.Find(".status").TextContent.Should().Be("Loaded"));
```

`WaitForState`/`WaitForAssertion` poll until a condition holds (or time out), handling the asynchronous render after `OnInitializedAsync` completes — avoiding flaky "assert before the async finished" failures.

---

## Mocking JS interop

JS interop ([09-JSInterop.md](09-JSInterop.md)) has no real runtime in tests; bUnit provides a configurable fake `IJSRuntime`:

```csharp
JSInterop.Mode = JSRuntimeMode.Loose;             // unmatched calls return default
JSInterop.Setup<int>("getWindowWidth").SetResult(1024);

var cut = RenderComponent<ResponsivePanel>();
JSInterop.VerifyInvoke("getWindowWidth");          // assert the call happened
```

`Loose` mode no-ops unmatched calls (handy for components that call JS you don't care about); `Strict` mode requires every call to be explicitly set up (catches unexpected interop). You can set return values and verify invocations.

---

## What to test (and what not to)

- **Test**: conditional rendering, event handlers changing state, parameter reactions, validation outcomes, that the right service calls happen — the component's *logic*.
- **Don't over-test**: exact CSS/visual styling (that's not bUnit's job — use visual/E2E tools for that), or third-party component internals.
- For full end-to-end flows (real browser, real backend), use **Playwright**/Selenium ([Ch17 §05](../17-Testing/README.md)) — bUnit is for fast component-level unit tests, not browser E2E.

---

## Common gotchas

### Asserting before async render settles

Asserting immediately after `RenderComponent` for a component with async lifecycle sees the pre-data state. Use `WaitForState`/`WaitForAssertion`.

### Forgetting to register a required service

A component injecting a service that isn't registered in `Services` throws at render. Register all dependencies (real or mocked) before `RenderComponent`.

### Brittle raw-string markup assertions

Comparing rendered markup with exact string equality breaks on whitespace/attribute order. Use `MarkupMatches` (semantic comparison).

### Unconfigured JS interop calls

A component calling `IJSRuntime` fails in tests unless you set up bUnit's `JSInterop` (or use `Loose` mode). Configure it for components that use interop.

### Treating bUnit as E2E

bUnit renders in-memory without a browser — it won't catch CSS/layout/real-browser issues. Use Playwright for true end-to-end testing.

---

## Summary

- **bUnit** unit-tests Blazor components in-memory (no browser/server) on a standard test runner — fast tests of rendering, events, parameters, and state.
- Use **`TestContext`** to **`RenderComponent<T>`** with strongly-typed parameters/cascading values; query with CSS selectors and assert with **`MarkupMatches`** (semantic HTML comparison).
- **Trigger events** (`Click`/`Change`/`Input`/`Submit`) to test the event→handler→re-render loop; **register fake services** in `Services` (Moq/NSubstitute) to isolate from I/O.
- For **async lifecycle**, use **`WaitForState`/`WaitForAssertion`** to avoid asserting before the render settles; **mock JS interop** via bUnit's `JSInterop` (Loose/Strict).
- Test component **logic**, not styling; use **Playwright** for real-browser **E2E** ([Ch17 §05](../17-Testing/README.md)).

→ Next: [Questions.md](Questions.md)
