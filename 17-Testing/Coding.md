# Chapter 17 — Testing — Coding Problems

Build unit + integration tests for an API and benchmark a hot path. Each problem has a hidden solution — attempt it first.

---

### Problem 1 — A data-driven xUnit theory

Test a `Calculator.Add` with several input rows using one test method.

<details>
<summary>Solution</summary>

```csharp
public class CalculatorTests {
    [Theory]
    [InlineData(2, 3, 5)]
    [InlineData(-1, 1, 0)]
    [InlineData(int.MaxValue, 0, int.MaxValue)]
    public void Add_returns_sum(int a, int b, int expected)
        => Assert.Equal(expected, new Calculator().Add(a, b));
}
```

`[Theory]` + `[InlineData]` runs the same logic for each row — no duplicated tests.
</details>

---

### Problem 2 — Unit test with a mocked dependency

`OrderService.PlaceAsync` saves an order via `IOrderRepository` and sends a confirmation via `IEmailSender`. Test that a valid order is saved and an email sent.

<details>
<summary>Solution</summary>

```csharp
[Fact]
public async Task PlaceAsync_saves_and_emails() {
    var repo = Substitute.For<IOrderRepository>();
    var email = Substitute.For<IEmailSender>();
    var sut = new OrderService(repo, email);

    await sut.PlaceAsync(new Order { Id = 1, Total = 100 });

    await repo.Received(1).SaveAsync(Arg.Is<Order>(o => o.Id == 1));
    await email.Received(1).SendAsync(Arg.Any<string>(), Arg.Any<string>());
}
```

Mock the collaborators (NSubstitute); verify the interactions that *are* the behavior (saved once, emailed once). Don't over-verify.
</details>

---

### Problem 3 — Make a time-dependent class testable

`SubscriptionService.IsExpired` compares against `DateTime.UtcNow`, so it can't be tested deterministically. Refactor and test.

<details>
<summary>Solution</summary>

```csharp
public class SubscriptionService(TimeProvider clock) {
    public bool IsExpired(Subscription s) => s.ExpiresAt < clock.GetUtcNow();
}

[Fact]
public void IsExpired_true_after_expiry() {
    var clock = new FakeTimeProvider(new DateTimeOffset(2026, 1, 2, 0, 0, 0, TimeSpan.Zero));
    var sut = new SubscriptionService(clock);
    sut.IsExpired(new Subscription { ExpiresAt = new DateTime(2026, 1, 1) }).Should().BeTrue();
}
```

Inject **`TimeProvider`** (not `DateTime.UtcNow`) and use `FakeTimeProvider` (Microsoft.Extensions.TimeProvider.Testing) to control "now" — deterministic ([Ch02 §06](../02-BCL/06-DateTimeAndTime.md)).
</details>

---

### Problem 4 — Assert object equivalence

An API returns a `CustomerDto`. Assert it matches the expected DTO without a dozen property checks.

<details>
<summary>Solution</summary>

```csharp
actual.Should().BeEquivalentTo(new CustomerDto {
    Id = 5, Name = "Ada", Email = "ada@example.com"
}, opts => opts.Excluding(c => c.LastSeenAt));   // exclude a volatile field
```

`BeEquivalentTo` compares **structurally** (member-by-member), with `Excluding` for volatile fields (timestamps) — one line instead of many asserts ([04-FluentAssertions.md](04-FluentAssertions.md)).
</details>

---

### Problem 5 — Integration test with WebApplicationFactory

Test that `GET /products` returns 200 and a non-empty list, hitting the real pipeline.

<details>
<summary>Solution</summary>

```csharp
public class ProductApiTests : IClassFixture<WebApplicationFactory<Program>> {
    private readonly HttpClient _client;
    public ProductApiTests(WebApplicationFactory<Program> f) => _client = f.CreateClient();

    [Fact]
    public async Task Get_products_returns_ok_with_items() {
        var resp = await _client.GetAsync("/products");
        resp.StatusCode.Should().Be(HttpStatusCode.OK);
        var items = await resp.Content.ReadFromJsonAsync<List<Product>>();
        items.Should().NotBeNull().And.NotBeEmpty();
    }
}
// Ensure Program is accessible: add `public partial class Program { }` in the app.
```

Requests traverse the full production pipeline in-memory ([05-IntegrationTests.md](05-IntegrationTests.md)).
</details>

---

### Problem 6 — Replace a dependency in an integration test

In the integration test, swap the real `IEmailSender` for a fake so no real emails are sent.

<details>
<summary>Solution</summary>

```csharp
var client = factory.WithWebHostBuilder(b => {
    b.ConfigureTestServices(services => {
        services.RemoveAll<IEmailSender>();
        services.AddSingleton<IEmailSender, FakeEmailSender>();
    });
}).CreateClient();
```

`ConfigureTestServices` runs after the app's registrations, so it overrides them. Substitute external/nondeterministic deps; keep the rest real.
</details>

---

### Problem 7 — Integration test against a real database (Testcontainers)

Spin up a real PostgreSQL for repository tests.

<details>
<summary>Solution</summary>

```csharp
public class OrderRepoTests : IAsyncLifetime {
    private readonly PostgreSqlContainer _db = new PostgreSqlBuilder().WithImage("postgres:16").Build();

    public async Task InitializeAsync() {
        await _db.StartAsync();
        await using var ctx = CreateContext(_db.GetConnectionString());
        await ctx.Database.MigrateAsync();           // apply real migrations
    }
    public Task DisposeAsync() => _db.DisposeAsync().AsTask();

    [Fact]
    public async Task Saves_and_loads() {
        await using var ctx = CreateContext(_db.GetConnectionString());
        ctx.Orders.Add(new Order { Total = 50 });
        await ctx.SaveChangesAsync();
        (await ctx.Orders.CountAsync()).Should().Be(1);
    }
}
```

A real DB validates SQL, constraints, and migrations — fidelity the InMemory provider can't give ([06-TestContainers.md](06-TestContainers.md)).
</details>

---

### Problem 8 — Test a Blazor component (bUnit)

Render `Counter`, click the button, assert the count.

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

In-memory render; `Click()` runs the handler and re-renders; `MarkupMatches` asserts semantically ([07-Bunit.md](07-Bunit.md), [Ch14 §12](../14-Blazor/12-Testing.md)).
</details>

---

### Problem 9 — Benchmark string concatenation vs StringBuilder

Set up a BenchmarkDotNet comparison reporting time and allocations.

<details>
<summary>Solution</summary>

```csharp
[MemoryDiagnoser]
public class ConcatBenchmarks {
    private readonly string[] _parts = Enumerable.Range(0, 100).Select(i => i.ToString()).ToArray();

    [Benchmark(Baseline = true)]
    public string Concat() {
        var s = "";
        foreach (var p in _parts) s += p;     // returns result → no dead-code elimination
        return s;
    }

    [Benchmark]
    public string Builder() {
        var sb = new StringBuilder();
        foreach (var p in _parts) sb.Append(p);
        return sb.ToString();
    }
}
// Program.cs (Release): BenchmarkRunner.Run<ConcatBenchmarks>();
```

`[MemoryDiagnoser]` reports allocations (the real story — `+=` is O(n²) allocations). Return the result to avoid elimination; run Release, not attached ([08-BenchmarkDotNet.md](08-BenchmarkDotNet.md)).
</details>

---

### Problem 10 — Property-based test for a round-trip

Assert that serializing then deserializing an `Order` yields an equal object, for all generated orders.

<details>
<summary>Solution</summary>

```csharp
[Property]
public Property Json_roundtrips(Order order) {
    var json = JsonSerializer.Serialize(order);
    var back = JsonSerializer.Deserialize<Order>(json);
    return (back == order).ToProperty();
}
```

FsCheck generates ~100 random orders (including edge cases) and shrinks any failure to the minimal counterexample ([09-PropertyBased.md](09-PropertyBased.md)). Round-trip is a classic property.
</details>

---

### Problem 11 — Generate realistic test data with Bogus

Create 50 realistic, reproducible `Customer` records for a seed/test.

<details>
<summary>Solution</summary>

```csharp
var faker = new Faker<Customer>()
    .UseSeed(123)                                            // deterministic
    .RuleFor(c => c.Id, f => f.Random.Guid())
    .RuleFor(c => c.Name, f => f.Name.FullName())
    .RuleFor(c => c.Email, (f, c) => f.Internet.Email(c.Name))
    .RuleFor(c => c.CreatedAt, f => f.Date.Past());

var customers = faker.Generate(50);
```

Bogus gives realistic, varied data; `UseSeed` makes it reproducible so tests asserting on it don't flake ([09-PropertyBased.md](09-PropertyBased.md)).
</details>

---

### Problem 12 — Test an `[Authorize]` endpoint

Integration-test a protected `GET /admin/stats` endpoint by injecting a test principal.

<details>
<summary>Solution</summary>

```csharp
public class TestAuthHandler(IOptionsMonitor<AuthenticationSchemeOptions> o, ILoggerFactory l, UrlEncoder e)
    : AuthenticationHandler<AuthenticationSchemeOptions>(o, l, e) {
    protected override Task<AuthenticateResult> HandleAuthenticateAsync() {
        var claims = new[] { new Claim(ClaimTypes.Name, "tester"), new Claim(ClaimTypes.Role, "Admin") };
        var principal = new ClaimsPrincipal(new ClaimsIdentity(claims, "Test"));
        return Task.FromResult(AuthenticateResult.Success(new AuthenticationTicket(principal, "Test")));
    }
}
// In the test factory:
b.ConfigureTestServices(s => s.AddAuthentication("Test")
    .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>("Test", _ => { }));
// client.DefaultRequestHeaders.Authorization = new("Test");
```

The test handler injects a known principal with the `Admin` role, so the authorization policy is exercised without a real identity provider ([05-IntegrationTests.md](05-IntegrationTests.md)).
</details>

---

### Problem 13 — Spot the flaky test

```csharp
[Fact]
public async Task Recent_orders_excludes_old() {
    var orders = await _service.GetRecentAsync();   // "recent" = created in last 24h
    orders.Should().OnlyContain(o => o.CreatedAt > DateTime.UtcNow.AddDays(-1));
}
```

Why might this flake, and how do you fix it?

<details>
<summary>Solution</summary>

It depends on **real wall-clock time** (`DateTime.UtcNow`) and on data created at unknown times — non-deterministic. The "now" used inside the service may differ from the assertion's `UtcNow`, and seeded data ages. Fix by injecting **`TimeProvider`** into the service and the test, controlling "now," and seeding data relative to that fixed time:

```csharp
var clock = new FakeTimeProvider(fixedNow);
// service uses clock.GetUtcNow(); seed one order at fixedNow-1h and one at fixedNow-2d;
// assert against fixedNow, not DateTime.UtcNow.
```

Eliminating the wall-clock nondeterminism makes it reliable ([10-TestStrategy.md](10-TestStrategy.md)).
</details>

---

### Problem 14 — Choose the test level

For each, pick unit / integration / E2E and justify: (a) a discount-calculation algorithm with many rules, (b) "POST /orders persists and returns 201", (c) "user can complete checkout in the browser", (d) an EF Core query with a complex `GroupBy`.

<details>
<summary>Solution</summary>

- **(a) Discount algorithm → unit** (mocked/isolated): logic-rich, many branches — cheapest to cover exhaustively in isolation.
- **(b) POST /orders → integration** (`WebApplicationFactory` + real/Testcontainers DB): tests the seam — routing, binding, validation, persistence, 201 — end-to-end in-process.
- **(c) Checkout in browser → E2E** (Playwright): a critical real-browser user journey; keep these few.
- **(d) Complex EF GroupBy → integration** against a **real DB** (Testcontainers): the InMemory provider may not translate/behave like real SQL, hiding query bugs ([06-TestContainers.md](06-TestContainers.md)).
</details>

---

### Problem 15 — Fix an over-mocked, brittle test

```csharp
[Fact]
public void Process_calls_steps_in_order() {
    // mocks every internal helper and asserts each is called in sequence
    _validator.Verify(...); _mapper.Verify(...); _repo.Verify(...); _logger.Verify(...);
}
```

Why is this brittle, and what's better?

<details>
<summary>Solution</summary>

It asserts the **implementation** (which internal calls happen, in what order), so any harmless refactor (reordering, inlining a helper) breaks it even though behavior is unchanged — and it doesn't actually verify the *outcome*. Better: test the **behavior** — given an input, assert the observable result/effect:

```csharp
[Fact]
public async Task Process_persists_a_valid_mapped_entity() {
    var repo = Substitute.For<IRepo>();
    var sut = new Processor(new RealValidator(), new RealMapper(), repo);
    await sut.ProcessAsync(validInput);
    await repo.Received(1).SaveAsync(Arg.Is<Entity>(e => e.Name == "expected"));
}
```

Mock only the meaningful boundary (the repository), use real internals where cheap, and assert the **outcome** — robust to refactors ([03-Mocking.md](03-Mocking.md), [10-TestStrategy.md](10-TestStrategy.md)).
</details>
