# xUnit (Deep)

## The default .NET test framework

**xUnit** is the most widely used test framework in modern .NET — clean, convention-driven, and the default for new ASP.NET Core/.NET projects. A test is a method marked `[Fact]` (or `[Theory]` for data-driven tests) in a class; the test runner discovers and executes them. This file goes beyond "write a test" into xUnit's specifics — its lifecycle, fixtures, parallelism, and data-driven testing — which matter for writing correct, fast, isolated tests at scale.

```csharp
public class CalculatorTests {
    [Fact]
    public void Add_returns_sum() {
        var calc = new Calculator();
        Assert.Equal(5, calc.Add(2, 3));
    }
}
```

> The basics of unit testing (AAA pattern, what makes a good test) are covered in [CSharpBook Ch16 Testing](../../CSharpBook/16-Testing/README.md). This chapter focuses on the .NET *platform* testing stack — integration tests, real dependencies, and component tests.

---

## `[Fact]` vs `[Theory]`

- **`[Fact]`** — a single, parameterless test asserting one scenario.
- **`[Theory]`** — a **data-driven** test run once per data row, with parameters supplied by data attributes:

```csharp
[Theory]
[InlineData(2, 3, 5)]
[InlineData(-1, 1, 0)]
[InlineData(0, 0, 0)]
public void Add_returns_sum(int a, int b, int expected)
    => Assert.Equal(expected, new Calculator().Add(a, b));
```

Data sources:
- **`[InlineData(...)]`** — constant arguments inline (simplest).
- **`[MemberData(nameof(Source))]`** — a static property/method returning `IEnumerable<object[]>` (for computed/complex cases).
- **`[ClassData(typeof(...))]`** — a class implementing `IEnumerable<object[]>` (reusable data sets).

Theories eliminate copy-pasted near-identical tests — one logic, many cases.

---

## Test lifecycle — constructor and `IDisposable`

xUnit creates a **new instance of the test class for every test method** — this is deliberate isolation: tests can't accidentally share mutable instance state. Setup goes in the **constructor**; teardown in **`Dispose`**:

```csharp
public class OrderServiceTests : IDisposable {
    private readonly OrderService _sut;
    public OrderServiceTests() { _sut = new OrderService(); }   // runs before EACH test
    public void Dispose() { /* cleanup after EACH test */ }
}
```

There's no `[SetUp]`/`[TearDown]` attribute (unlike NUnit — [02-NUnitMSTest.md](02-NUnitMSTest.md)); xUnit uses the constructor/`Dispose` because a fresh instance per test is the cleanest isolation model. For async setup/teardown, implement `IAsyncLifetime` (`InitializeAsync`/`DisposeAsync`).

---

## Fixtures — sharing expensive setup

Creating a fresh instance per test is great for isolation but wasteful when setup is **expensive** (a database, a web host). **Fixtures** share one setup instance across tests:

- **Class fixture** (`IClassFixture<T>`) — one shared instance across all tests *in one class*:

```csharp
public class DbFixture : IDisposable {
    public DbConnection Connection { get; } = CreateAndSeed();
    public void Dispose() => Connection.Dispose();
}
public class RepoTests : IClassFixture<DbFixture> {
    private readonly DbFixture _fixture;
    public RepoTests(DbFixture fixture) => _fixture = fixture;   // injected, shared
}
```

- **Collection fixture** (`ICollectionFixture<T>` + `[Collection("name")]`) — one shared instance across *multiple classes* in a collection (e.g., share one database/web host across an entire test suite).

Use fixtures for expensive, shareable resources (databases — [06-TestContainers.md](06-TestContainers.md), `WebApplicationFactory` — [05-IntegrationTests.md](05-IntegrationTests.md)); use the constructor for cheap per-test setup.

---

## Parallelism — and its hazards

By default, xUnit runs tests in **different classes in parallel** (tests within a single class run sequentially). This speeds up suites but means **shared mutable state across tests will race**:

- Tests in the same **collection** do *not* run in parallel with each other — so a collection fixture's shared resource (a database) is accessed serially, avoiding races.
- Across collections/classes, tests run concurrently — so don't share static mutable state, and isolate per-test data.

```csharp
[Collection("Database collection")]   // these tests run serially, sharing the fixture safely
public class OrderTests { ... }
```

The practical rule: keep tests **independent and stateless**; use collections to serialize tests that must share a resource. Flaky tests are often parallelism exposing hidden shared state.

---

## Async tests

xUnit natively supports `async Task` test methods — just `await` inside:

```csharp
[Fact]
public async Task GetAsync_returns_data() {
    var result = await _service.GetAsync(1);
    Assert.NotNull(result);
}
```

**Return `Task`, never `async void`** — an `async void` test can't be awaited by the runner, so failures/exceptions may be missed or the test reported as passing prematurely. (Same async rule as everywhere — [CSharpBook async](../../CSharpBook/08-Concurrency/README.md).)

---

## Assertions and expected exceptions

xUnit's built-in assertions (`Assert.Equal`, `Assert.True`, `Assert.Contains`, `Assert.Throws<T>`, …) cover the basics; many teams add **FluentAssertions** for readability ([04-FluentAssertions.md](04-FluentAssertions.md)). For exceptions:

```csharp
var ex = Assert.Throws<ArgumentException>(() => _sut.Process(null!));
Assert.Equal("input", ex.ParamName);

await Assert.ThrowsAsync<InvalidOperationException>(() => _sut.RunAsync());
```

`Assert.Throws<T>` asserts the **exact** type (use `Assert.ThrowsAny<T>` for derived types); `ThrowsAsync` for async.

---

## Common gotchas

### Expecting `[SetUp]`/`[TearDown]`

xUnit doesn't have them — use the **constructor** (setup) and **`Dispose`**/`IAsyncLifetime` (teardown). A fresh class instance per test is the isolation model.

### Shared mutable state + parallelism

Tests in different classes run in parallel; shared static/mutable state races and causes flaky tests. Keep tests independent; use **collections** to serialize tests sharing a resource.

### `async void` tests

The runner can't await `async void` — exceptions may be lost and the test wrongly pass. Always use `async Task`.

### Recreating expensive resources per test

Spinning up a database/web host in every test constructor is slow. Use **class/collection fixtures** to share expensive setup.

### Over-using `[InlineData]` for complex objects

`InlineData` only takes constants. For computed/complex cases use `[MemberData]`/`[ClassData]` rather than contorting inline data.

---

## Summary

- **xUnit** is the default .NET test framework: **`[Fact]`** (single test) and **`[Theory]`** (data-driven, via `[InlineData]`/`[MemberData]`/`[ClassData]`) eliminate duplicated test logic.
- A **new test-class instance per test** gives isolation; setup goes in the **constructor**, teardown in **`Dispose`** (async: `IAsyncLifetime`) — there's no `[SetUp]`/`[TearDown]`.
- **Fixtures** share expensive setup: **`IClassFixture<T>`** (per class) and **`ICollectionFixture<T>`** + `[Collection]` (across classes) — for databases/web hosts.
- Tests in **different classes run in parallel**; keep tests independent and use **collections** to serialize those sharing a resource (parallelism exposes hidden shared state as flakiness).
- Use **`async Task`** tests (never `async void`); assert exceptions with **`Assert.Throws<T>`/`ThrowsAsync<T>`**; the AAA basics are in [CSharpBook Ch16](../../CSharpBook/16-Testing/README.md).

→ Next: [02-NUnitMSTest.md](02-NUnitMSTest.md)
