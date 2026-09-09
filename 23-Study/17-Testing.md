# 17 — Testing

## ⚡ 30-second answer

**xUnit** is the default: `[Fact]` for a single test, `[Theory]` + `[InlineData]` for data-driven
ones, and — the distinguishing design — **a new test-class instance per test**, so setup goes in
the constructor and teardown in `Dispose`. Share expensive setup with **fixtures**
(`IClassFixture`, `ICollectionFixture`). **`WebApplicationFactory<Program>`** boots your real app
in-memory and sends requests through the **full production pipeline**, which catches routing, DI,
middleware and serialization bugs unit tests never see; **`ConfigureTestServices`** replaces
specific dependencies while keeping the rest real. For databases, **don't mock `DbContext`** and
don't trust the **InMemory provider** — use **SQLite in-memory** or **Testcontainers** for real
fidelity. Mock only at meaningful boundaries, **test behavior not implementation**, keep tests
**deterministic** (`TimeProvider`, seeded randomness), and keep **E2E few**.

---

## Core mechanics

### xUnit

```csharp
public class OrderTests : IDisposable          // ctor = setup, Dispose = teardown
{
    private readonly OrderService _sut;
    public OrderTests() => _sut = new OrderService(…);   // runs before EVERY test

    [Fact]
    public async Task Ships_open_order() { … }

    [Theory]
    [InlineData(0), InlineData(-1)]
    public void Rejects_invalid_quantity(int qty) =>
        Assert.Throws<DomainException>(() => _sut.AddLine(qty));

    public void Dispose() { … }
}
```

- **New instance per test** — isolation by design. There is no `[SetUp]`/`[TearDown]`; async setup
  is `IAsyncLifetime`.
- **Fixtures** share expensive setup: **`IClassFixture<T>`** (one instance per test class),
  **`ICollectionFixture<T>`** + `[Collection]` (shared across classes) — for a database or a web
  host.
- **Tests in different classes run in parallel.** Use a `[Collection]` to serialize those sharing a
  resource. Parallelism is what exposes hidden shared state as flakiness.
- **`async Task` tests, never `async void`** — an `async void` test's failures are invisible to the
  runner.

### Framework differences

| | xUnit | NUnit | MSTest |
|---|---|---|---|
| Test | `[Fact]` | `[Test]` | `[TestMethod]` |
| Data-driven | `[Theory]`+`[InlineData]` | `[TestCase]` | `[DataTestMethod]`+`[DataRow]` |
| Setup/teardown | **ctor + `Dispose`** | `[SetUp]`/`[TearDown]` | `[TestInitialize]`/`[TestCleanup]` |
| Instance lifetime | **new per test** | **reused** | **reused** |

That last row is the one that matters: NUnit and MSTest reuse one instance across tests, so
leftover state causes order-dependent flakiness unless you reset it in setup.

### Mocking

| Double | Is |
|---|---|
| **Stub** | canned return values |
| **Mock** | a stub that also **verifies interactions** |
| **Fake** | a simplified but working implementation (in-memory repository) |

```csharp
var repo = Substitute.For<IOrderRepository>();      // NSubstitute
repo.GetAsync(id, Arg.Any<CancellationToken>()).Returns(order);
…
await repo.Received(1).SaveAsync(Arg.Any<Order>(), Arg.Any<CancellationToken>());
```

Moq, NSubstitute and FakeItEasy are equivalent in capability — pick one and be consistent.

**Mock abstractions you own** — repositories, domain services, `TimeProvider`, an HTTP boundary.
**Don't over-mock**: a test that asserts every internal call tests the *implementation*, so it
breaks on every refactor while proving nothing about behavior.

The awkward types have standard answers:
- **`HttpClient`** — mock the **`HttpMessageHandler`**, not the client.
- **Time** — inject **`TimeProvider`**, use `FakeTimeProvider`.
- **Sealed/static dependencies** — wrap behind an interface you own.
- **Stateful collaborators** — a **fake** is usually better than a pile of `Setup` calls.

### Integration testing

```csharp
public class ApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public ApiTests(WebApplicationFactory<Program> factory) =>
        _client = factory.WithWebHostBuilder(b => b.ConfigureTestServices(s =>
        {
            s.RemoveAll<IPaymentGateway>();
            s.AddSingleton<IPaymentGateway, FakeGateway>();     // replace only the external edge
        })).CreateClient();

    [Fact]
    public async Task Get_returns_404_for_unknown()
        => Assert.Equal(HttpStatusCode.NotFound, (await _client.GetAsync("/orders/999")).StatusCode);
}
```

`WebApplicationFactory` runs your **real app** on a `TestServer` (in-memory, no socket) and gives
you an `HttpClient` that goes through the **full pipeline** — routing, model binding, filters,
middleware, DI, serialization. That's the class of bug it catches.

**`ConfigureTestServices` runs after the app's own registrations**, so it overrides them —
`ConfigureServices` runs before and gets overwritten. Replace only what's genuinely external.

For `[Authorize]` endpoints, register a **test authentication handler** that injects a known
principal.

Make `Program` visible to the test project: `public partial class Program { }` (or
`InternalsVisibleTo`).

### Database strategy

| Approach | Fidelity | Use |
|---|---|---|
| Mock `DbContext`/`DbSet` | none | ❌ never — LINQ-to-Objects, not LINQ-to-Entities |
| **InMemory provider** | low | ❌ not relational: no constraints, no SQL translation |
| **SQLite in-memory** | medium | ✅ a real SQL engine — the sensible default |
| **Testcontainers** | **high** | ✅ your real database — query-heavy, DB-specific, migrations |

**Testcontainers** starts real dependencies in throwaway Docker containers from test code, managed
by `IAsyncLifetime` and a collection fixture. It covers the whole landscape — SQL Server,
PostgreSQL, Redis, RabbitMQ, Kafka, Azurite. Share a container across a collection for speed,
**reset state per test** (transaction rollback, or Respawn), and **pin image versions**. It
requires Docker, so gate those tests where it isn't available.

### Test isolation

The three workable strategies: a **transaction rolled back** after each test (fast, but doesn't
work if the code under test commits), **Respawn** (deletes data between tests), or a **fresh
database per test class**. Pick one and apply it consistently — tests that share state pass alone
and fail in CI.

### Assertions

**FluentAssertions** (`x.Should().Be(…)`) gives rich failure messages naming exactly which member
differed. Its standout is **`BeEquivalentTo`** — structural, member-by-member comparison, ideal
for DTO, mapping and API-contract tests. Note **v8+ is commercially licensed** — pin v7, or use
**Shouldly**/AwesomeAssertions. **`AssertionScope`** reports multiple failures at once instead of
stopping at the first.

### BenchmarkDotNet

```csharp
[MemoryDiagnoser]                                     // allocations matter as much as time
public class Bench
{
    [Params(10, 1000)] public int N;                  // see how it scales
    [GlobalSetup] public void Setup() { … }
    [Benchmark(Baseline = true)] public int Old() => …;   // ← RETURN the value
    [Benchmark] public int New() => …;
}
```

It handles what naive `Stopwatch` timing gets wrong: **JIT warmup** (Tier 0 vs Tier 1 —
[01](01-Runtime-GC-JIT.md)), **dead-code elimination** (which is why you return results), GC
noise, and statistics. Read Mean/StdDev (low StdDev = trustworthy), Ratio vs baseline, and
Allocated/Gen0. **Always Release, never with a debugger attached.**

### Property-based testing

FsCheck asserts properties true for **all** inputs, generating hundreds of cases to falsify them.
Good properties: **round-trip** (`decode(encode(x)) == x`), **invariants** (length preserved,
output sorted), **idempotence**, **commutativity**, and **oracle** (matches a known-correct
implementation). Its killer feature is **shrinking** — automatically reducing a failure to the
minimal counterexample. Use it for algorithms, parsers and serializers; example tests for
everything else. **Bogus** generates realistic fake data — seed it for determinism.

### Strategy

The **pyramid** (many unit, some integration, few E2E) and the **testing trophy** (integration
heaviest, static analysis as the free base) disagree about the middle but agree on the ends: keep
**E2E few**, and lean on the **compiler, nullable reference types and analyzers** as the cheapest
defense you have.

Unit-test logic-rich code; integration-test the **seams**; E2E only critical journeys. **Test
behavior, not implementation**, so tests survive refactoring. Use coverage to **find gaps**, not
as a target — and mutation testing if you want to know whether your assertions actually assert
anything.

---

## 🪤 Traps & gotchas

- **`async void` tests** — the runner can't await them, so failures and exceptions vanish and the
  test reports green. Always `async Task`.
- **Shared state between tests** — a static field, a shared database row, a singleton. Passes
  locally, fails in CI, and the failure moves around because xUnit runs classes in parallel.
- **Assuming NUnit/MSTest give you a fresh instance per test** — they don't; state leaks between
  tests in the same class and creates order dependence.
- **Over-mocking** — verifying every internal call couples the test to the implementation, so a
  pure refactor breaks a hundred tests while behavior is unchanged.
- **Mocking types you don't own** — you encode your *assumptions* about a third-party library, and
  the test passes even when the assumption is wrong.
- **Mocking `HttpClient`** — mock the `HttpMessageHandler` instead; `HttpClient` is a concrete
  class whose important behavior lives in the handler.
- **Mocking `DbContext`/`DbSet`** — a mocked `IQueryable` runs LINQ-to-Objects, so it never
  exercises SQL translation, behaves differently for async, and you end up asserting your own
  `Setup` calls.
- **Trusting the EF InMemory provider** — not a relational database. No foreign keys, no unique
  constraints, no not-null enforcement, no real transactions, and queries that would throw
  "could not be translated" silently work. It passes tests that fail in production; Microsoft
  advises against it.
- **SQLite in-memory database disappearing** — it lives only as long as the connection. Close the
  connection and the next test sees an empty database. Hold it open for the fixture's lifetime.
- **Testing only against SQLite and deploying to PostgreSQL** — dialect differences, case
  sensitivity and type mapping still bite.
- **A container per test** — correct but glacial. Share via a collection fixture and reset state
  between tests instead.
- **Using `ConfigureServices` instead of `ConfigureTestServices`** in `WebApplicationFactory` —
  yours runs *before* the app's registrations and gets overwritten, so your fake is silently
  ignored and the test hits the real service.
- **Forgetting `public partial class Program { }`** — `WebApplicationFactory<Program>` won't
  compile against the implicit entry point from the test project.
- **`DateTime.Now` in code under test** — the test fails at midnight, on the last day of the month,
  or in a different timezone in CI. Inject `TimeProvider`.
- **Unseeded random data** — a test that fails one run in fifty and can't be reproduced. Seed it.
- **`Thread.Sleep` to wait for async work** — slow when it passes and flaky when it doesn't. Poll
  with a timeout, or use `FakeTimeProvider`.
- **Benchmarking in Debug or with the debugger attached** — meaningless numbers; BenchmarkDotNet
  will warn you and people ignore it.
- **A benchmark that doesn't return its result** — the JIT eliminates the whole computation as dead
  code and you measure an empty loop.
- **Chasing 100% coverage** — you get tests asserting that getters return what was set, which
  raises the number and catches nothing.
- **A large E2E suite** — slow, flaky, and expensive to maintain. Push the coverage down to
  integration tests where it's stable.

---

## ❓ Likely questions

**Q: What's distinctive about xUnit's lifecycle?**
A: A **new instance of the test class per test**, so isolation is the default. Setup is the
constructor and teardown is `Dispose` (or `IAsyncLifetime` for async) — there are no `[SetUp]`
attributes. NUnit and MSTest reuse one instance, which is where order-dependent flakiness comes
from.

**Q: `[Fact]` vs `[Theory]`?**
A: `[Fact]` is a single test with no parameters. `[Theory]` is data-driven — the same logic run
against many inputs from `[InlineData]`, `[MemberData]` or `[ClassData]`, which removes duplicated
test bodies.

**Q: How do you share expensive setup between tests?**
A: `IClassFixture<T>` for one instance shared across a test class, and `ICollectionFixture<T>` with
`[Collection]` for sharing across classes — the usual home for a database container or a
`WebApplicationFactory`.

**Q: Stub, mock, fake — what's the difference?**
A: A stub returns canned values. A mock also **verifies interactions** — that a method was called,
with what, how often. A fake is a real but simplified implementation, like an in-memory
repository. Overusing mocks (verification) is what makes suites brittle.

**Q: What should you mock, and what shouldn't you?**
A: Mock abstractions you own at meaningful boundaries — repositories, external service clients,
`TimeProvider`. Don't mock types you don't own (you encode assumptions), value objects, the system
under test, or `DbContext`. And don't verify internal interactions that aren't part of the
behavior you're testing.

**Q: How do you test code that calls `HttpClient`?**
A: Mock the `HttpMessageHandler` and inject an `HttpClient` built on it — that's where request and
response handling actually happens. Alternatively spin up a real endpoint with
`WebApplicationFactory` or a WireMock container.

**Q: What does `WebApplicationFactory` give you that a unit test doesn't?**
A: It runs your real application in-memory and sends requests through the **entire pipeline** —
routing, middleware ordering, model binding, validation, filters, DI wiring, JSON serialization,
error handling. That's a whole category of bug (middleware in the wrong order, a missing
registration, a serialization mismatch) that unit tests cannot see.

**Q: How do you replace a dependency in an integration test?**
A: `WithWebHostBuilder(b => b.ConfigureTestServices(...))`. It runs **after** the app's own
registrations, so `RemoveAll<T>` plus your fake actually overrides. `ConfigureServices` runs
before and gets overwritten — a classic silent failure.

**Q: How do you test `[Authorize]` endpoints?**
A: Register a test authentication handler that always succeeds with a known `ClaimsPrincipal`, and
make it the default scheme in `ConfigureTestServices`. That lets you test authorization policies
for real without standing up an identity provider.

**Q: Why not use the EF InMemory provider?**
A: It isn't relational. There are no foreign-key, unique or not-null constraints, no real
transactions, and no SQL translation — so queries that would throw in production pass, and data
your database would reject is accepted. It gives false confidence; Microsoft recommends against
it.

**Q: SQLite in-memory or Testcontainers?**
A: SQLite in-memory is a real SQL engine, fast, and a fine default for most data-layer tests — just
keep the connection open for the fixture's lifetime or the database vanishes. Testcontainers runs
your **actual** database, which is what you want for query-heavy code, provider-specific SQL, and
validating migrations.

**Q: How do you keep integration tests isolated?**
A: Pick one strategy and apply it everywhere: wrap each test in a transaction and roll back, use
Respawn to delete data between tests, or give each class a fresh database. Share the expensive
container via a collection fixture, and serialize database tests with `[Collection]`.

**Q: Why does BenchmarkDotNet exist — can't you use a `Stopwatch`?**
A: Because a naive loop measures the wrong thing. It handles JIT warmup (your first calls run
unoptimized Tier 0 code), prevents dead-code elimination, isolates GC noise, and reports proper
statistics. Plus `[MemoryDiagnoser]` shows allocations, which often matter more than the time.

**Q: What is property-based testing good for?**
A: Code with clear invariants — parsers, serializers, algorithms. You assert a property like
"decode(encode(x)) equals x" and the framework generates hundreds of boundary-heavy inputs trying
to break it, then **shrinks** any failure to the minimal counterexample.

**Q: Pyramid or trophy?**
A: They agree where it counts — keep E2E few, and let static analysis (the compiler, nullable
reference types, analyzers) do the cheap work. The trophy weights integration tests more heavily,
which fits modern .NET well because `WebApplicationFactory` plus Testcontainers makes realistic
tests fast enough to be the main layer.

**Q: Is code coverage a good target?**
A: It's a good way to **find untested areas** and a bad target. Optimizing the number produces
tests that execute code without asserting anything meaningful. Mutation testing measures whether
your assertions would actually catch a bug.

---

## 🎓 Senior Extra

- **Test behavior at the seam that matches the abstraction.** The reason `WebApplicationFactory`
  tests age so much better than heavily-mocked unit tests is that HTTP is a real contract, while
  "OrderService calls IOrderRepository.SaveAsync once" is an implementation detail you'll change
  next sprint.
- **Flakiness is a defect, not a nuisance.** A suite people re-run to get green has stopped being a
  signal. Quarantine and fix, don't retry.
- **Determinism has three usual suspects**: time, randomness, and concurrency. `TimeProvider` and a
  seeded generator handle the first two; the third needs explicit synchronization points rather
  than sleeps.
- **`FakeTimeProvider` also fakes timers and delays**, which turns a retry-backoff or scheduler test
  from a 30-second sleepy integration test into an instant deterministic unit test
  ([11](11-Resilience.md)).
- **Testcontainers is a fixture-lifetime problem, not a Docker problem.** One container per
  collection, `Respawn` between tests, and a `[CollectionDefinition]` shared by every data test —
  get that right and real-database testing is fast enough for the inner loop.
- **Architecture tests** (NetArchTest) enforce structural rules the compiler can't — "the domain
  project must not reference EF Core". They're the cheapest way to make an architectural decision
  survive team growth ([22](22-BestPractices-Architecture.md)).
- **Contract tests** between services (Pact, or shared schema tests) catch the class of failure
  that integration tests within one service structurally cannot — the producer changed and nobody
  told the consumer ([07](07-Messaging.md)).
- **Snapshot/approval testing** (Verify) is excellent for serialization, generated SQL, and API
  responses — the assertion is "this didn't change", which is exactly what you want for contracts.
- **FluentAssertions v8 changed to a commercial licence.** Pin v7, or move to Shouldly or
  AwesomeAssertions — worth knowing because it's a live decision in most .NET codebases right now.
- **Benchmark at steady state, profile in production.** A microbenchmark tells you which of two
  implementations is faster; it does not tell you that this method matters. Get that from
  telemetry first ([12](12-Observability.md), [21](21-Performance-Tooling.md)).

→ Deeper: [`../17-Testing/`](../17-Testing/README.md)
