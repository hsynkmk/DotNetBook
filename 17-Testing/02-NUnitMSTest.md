# NUnit and MSTest

## The other two frameworks

xUnit ([01-xUnitDeep.md](01-xUnitDeep.md)) is the modern default, but two other test frameworks are widely used in .NET: **NUnit** (mature, feature-rich, popular in many shops) and **MSTest** (Microsoft's framework, the historical Visual Studio default). All three discover and run tests, integrate with `dotnet test`, and work with the same mocking/assertion libraries — they differ mainly in **attributes, lifecycle, and parameterization syntax**. Knowing the mapping lets you read/maintain any codebase regardless of which it chose.

---

## The attribute mapping

The core concepts are the same across frameworks; only the names differ:

| Concept | xUnit | NUnit | MSTest |
|---|---|---|---|
| Test method | `[Fact]` | `[Test]` | `[TestMethod]` |
| Data-driven | `[Theory]`+`[InlineData]` | `[TestCase(...)]` | `[DataTestMethod]`+`[DataRow]` |
| Test class | (plain class) | `[TestFixture]` | `[TestClass]` |
| Per-test setup | constructor | `[SetUp]` | `[TestInitialize]` |
| Per-test teardown | `Dispose` | `[TearDown]` | `[TestCleanup]` |
| Once per class (setup) | `IClassFixture<T>` | `[OneTimeSetUp]` | `[ClassInitialize]` |
| Once per class (teardown) | `IClassFixture<T>` | `[OneTimeTearDown]` | `[ClassCleanup]` |
| Skip | `[Fact(Skip="…")]` | `[Ignore("…")]` | `[Ignore]` |
| Expected category/trait | `[Trait]` | `[Category]` | `[TestCategory]` |

---

## NUnit specifics

NUnit uses explicit lifecycle attributes and **re-uses a single test-class instance** across tests by default (unlike xUnit's fresh instance per test) — so be careful with instance state:

```csharp
[TestFixture]
public class CalculatorTests {
    private Calculator _calc = null!;

    [SetUp] public void Setup() => _calc = new Calculator();      // before each test
    [TearDown] public void Cleanup() { /* after each test */ }

    [Test]
    public void Add_works() => Assert.That(_calc.Add(2, 3), Is.EqualTo(5));

    [TestCase(2, 3, 5)]
    [TestCase(-1, 1, 0)]
    public void Add_cases(int a, int b, int expected)
        => Assert.That(_calc.Add(a, b), Is.EqualTo(expected));
}
```

NUnit's **constraint model** (`Assert.That(actual, Is.EqualTo(expected))`) is expressive and readable, and `[TestCase]` puts inline data right on the test. NUnit is feature-rich (parameterized fixtures, value sources, rich constraints) and a common choice in enterprise codebases.

---

## MSTest specifics

MSTest uses `[TestClass]`/`[TestMethod]` and `[TestInitialize]`/`[TestCleanup]`. It's the historical Visual Studio default and integrates tightly with VS/Azure DevOps tooling:

```csharp
[TestClass]
public class CalculatorTests {
    private Calculator _calc = null!;

    [TestInitialize] public void Init() => _calc = new Calculator();

    [TestMethod]
    public void Add_works() => Assert.AreEqual(5, _calc.Add(2, 3));

    [DataTestMethod]
    [DataRow(2, 3, 5)]
    [DataRow(-1, 1, 0)]
    public void Add_cases(int a, int b, int expected)
        => Assert.AreEqual(expected, _calc.Add(a, b));
}
```

Modern MSTest (v3) is improved and competitive, but xUnit and NUnit are more common in new OSS/.NET projects.

---

## Lifecycle/isolation difference (the key gotcha)

The most important behavioral difference: **xUnit creates a new test-class instance per test** (strong isolation), while **NUnit and MSTest reuse one instance** across the class's tests by default (with `[SetUp]`/`[TestInitialize]` re-running per test). This means:

- In xUnit, instance fields are naturally fresh each test.
- In NUnit/MSTest, instance fields **persist** across tests unless reset in `[SetUp]`/`[TestInitialize]` — forgetting to reset state is a classic source of order-dependent, flaky tests.

When porting tests between frameworks, this isolation difference is the thing most likely to bite.

---

## Which to choose

- **New projects**: **xUnit** is the common default (clean, isolation-by-design, ASP.NET Core templates use it).
- **Existing codebases**: use whatever's already there — they're all capable; consistency matters more than the choice.
- **NUnit**: pick for its rich constraint model and mature parameterization if the team prefers it.
- **MSTest**: fine, especially with deep VS/Azure DevOps integration or existing MSTest suites.

The framework choice rarely matters for *what* you can test — all three work with Moq/NSubstitute ([03-Mocking.md](03-Mocking.md)), FluentAssertions ([04-FluentAssertions.md](04-FluentAssertions.md)), `WebApplicationFactory` ([05-IntegrationTests.md](05-IntegrationTests.md)), and `dotnet test`.

---

## Common gotchas

### Assuming xUnit isolation in NUnit/MSTest

NUnit/MSTest reuse one class instance; instance state persists across tests. Reset it in `[SetUp]`/`[TestInitialize]`, or you get order-dependent flakiness. xUnit's fresh-instance model doesn't have this.

### Mismatched parameterization attributes

`[Theory]`+`[InlineData]` (xUnit), `[TestCase]` (NUnit), `[DataTestMethod]`+`[DataRow]` (MSTest) are not interchangeable. Use the right one per framework when reading/porting.

### Mixing frameworks in one project

Combining attributes from different frameworks in one test project causes discovery confusion. Pick one framework per test project.

### Expecting identical assertion APIs

`Assert.Equal` (xUnit) vs `Assert.That(...Is.EqualTo...)` (NUnit) vs `Assert.AreEqual` (MSTest) differ. Standardizing on **FluentAssertions** ([04-FluentAssertions.md](04-FluentAssertions.md)) gives a uniform assertion style regardless of framework.

---

## Summary

- **NUnit** and **MSTest** are the two alternatives to **xUnit**; all three discover/run via `dotnet test` and work with the same mocking/assertion libraries — they differ in **attributes, lifecycle, and parameterization**.
- Mapping: test = `[Fact]`/`[Test]`/`[TestMethod]`; data-driven = `[Theory]+[InlineData]` / `[TestCase]` / `[DataTestMethod]+[DataRow]`; setup/teardown = ctor+`Dispose` / `[SetUp]`+`[TearDown]` / `[TestInitialize]`+`[TestCleanup]`.
- **Key difference**: xUnit makes a **new class instance per test** (isolation by design); **NUnit/MSTest reuse one instance** (reset state in setup or risk order-dependent flakiness).
- **Choose xUnit** for new projects (common default); use whatever exists in legacy code. Standardize assertions with **FluentAssertions** for a uniform style across frameworks.

→ Next: [03-Mocking.md](03-Mocking.md)
