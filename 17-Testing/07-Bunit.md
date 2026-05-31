# Testing Blazor Components with bUnit

## Component tests, fast and in-memory

**bUnit** is the component-testing library for Blazor ([Ch14](../14-Blazor/README.md)). It renders a component **in-memory** (no browser, no server, no real DOM), lets you pass parameters, inject services, trigger events, and assert on the rendered markup — all on a standard test runner ([01-xUnitDeep.md](01-xUnitDeep.md)). It's the unit-test layer for Blazor UI: fast, isolated, focused on component *logic* (conditional rendering, event handling, parameter reactions), not real-browser behavior.

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

> bUnit is covered in depth in **[Ch14 §12](../14-Blazor/12-Testing.md)** (the Blazor chapter). This section summarizes it in the testing context and connects it to the broader test strategy ([10-TestStrategy.md](10-TestStrategy.md)).

---

## The essentials (recap)

- **`TestContext`** is the render host + DI container. `RenderComponent<T>(...)` renders the component (`cut` = "component under test").
- **Parameters / cascading values** are supplied via a strongly-typed builder: `RenderComponent<T>(p => p.Add(c => c.Name, "Ada").AddCascadingValue(theme))`.
- **Query** the rendered markup with CSS selectors (`cut.Find("button")`, `cut.FindAll("li")`); **assert** with `MarkupMatches` (semantic HTML comparison) or FluentAssertions ([04-FluentAssertions.md](04-FluentAssertions.md)) on text/attributes.
- **Trigger events** (`Click`, `Change`, `Input`, `Submit`) to test the event→handler→re-render loop ([Ch14 §04](../14-Blazor/04-EventHandling.md)).
- **Inject fakes** into `Services` (Moq/NSubstitute — [03-Mocking.md](03-Mocking.md)) so the component's data fetch hits a test double.
- **Async**: use `WaitForState`/`WaitForAssertion` to wait for the render to settle after `OnInitializedAsync` before asserting.
- **JS interop**: configure bUnit's `JSInterop` (Loose/Strict) since there's no real JS runtime ([Ch14 §09](../14-Blazor/09-JSInterop.md)).

```csharp
[Fact]
public async Task Renders_loaded_data() {
    var api = Substitute.For<IProductApi>();
    api.GetAsync(1).Returns(new Product { Name = "Widget" });
    Services.AddSingleton(api);

    var cut = RenderComponent<ProductPage>(p => p.Add(c => c.Id, 1));
    cut.WaitForState(() => cut.Find("h1").TextContent == "Widget");
    cut.Find("h1").TextContent.Should().Be("Widget");
}
```

---

## What bUnit is (and isn't) for

- **Is for**: fast, isolated tests of component *logic* — conditional rendering, event handlers updating state, parameter reactions, validation outcomes, that the right service calls happen.
- **Isn't for**: real-browser concerns — CSS/layout/visual rendering, actual JS execution, cross-browser behavior, or full end-to-end user flows. bUnit renders in-memory without a real DOM/browser, so it can't catch styling or real-browser bugs.

For end-to-end Blazor testing in a real browser, use **Playwright** ([08-BenchmarkDotNet.md](08-BenchmarkDotNet.md) is unrelated; E2E is discussed in [10-TestStrategy.md](10-TestStrategy.md)) — drive the actual rendered app, clicking through real pages. bUnit (unit) and Playwright (E2E) are complementary layers of the strategy.

---

## Where it fits in the strategy

In the test pyramid/trophy ([10-TestStrategy.md](10-TestStrategy.md)), bUnit sits at the **component-unit** layer for Blazor:

```
        E2E (Playwright — real browser, few)
   Integration (WebApplicationFactory — pipeline)
 Component (bUnit — many, fast)  ← Blazor UI logic
      Unit (xUnit + mocks — many, fastest)
```

Most Blazor UI logic should be covered by fast bUnit component tests; reserve a smaller number of Playwright E2E tests for critical real-browser user journeys. This keeps the suite fast (most tests are in-memory) while still validating the real thing where it matters.

---

## Common gotchas

### Asserting before async render settles

Asserting right after `RenderComponent` for a component with async lifecycle sees the pre-data state. Use `WaitForState`/`WaitForAssertion` ([Ch14 §12](../14-Blazor/12-Testing.md)).

### Unregistered services

A component injecting a service not registered in `Services` throws at render. Register all dependencies (real or mocked) first.

### Brittle raw-string markup assertions

Exact string equality on markup breaks on whitespace/attribute order. Use `MarkupMatches` (semantic comparison).

### Treating bUnit as E2E

It renders in-memory without a browser — it won't catch CSS/layout/real-JS issues. Use Playwright for true end-to-end coverage.

### Unconfigured JS interop

Components calling `IJSRuntime` fail in tests unless you configure bUnit's `JSInterop`. Set it up (or use Loose mode) for interop-using components.

---

## Summary

- **bUnit** unit-tests Blazor components **in-memory** (no browser/server) on a standard test runner — fast, isolated tests of rendering, events, parameters, and state; covered in depth in **[Ch14 §12](../14-Blazor/12-Testing.md)**.
- Render with **`TestContext.RenderComponent<T>`** (strongly-typed parameters/cascading values), **query** via CSS selectors, **assert** with `MarkupMatches`/FluentAssertions, **trigger events**, **inject fakes** into `Services`, and use **`WaitForState`/`WaitForAssertion`** for async lifecycle + configure **`JSInterop`** for interop.
- bUnit tests component **logic**, **not** styling/real-browser behavior; pair it with **Playwright** E2E for critical real-browser journeys.
- In the **test strategy** ([10-TestStrategy.md](10-TestStrategy.md)), bUnit is the broad, fast **component-unit** layer for Blazor; keep E2E focused and few.

→ Next: [08-BenchmarkDotNet.md](08-BenchmarkDotNet.md)
