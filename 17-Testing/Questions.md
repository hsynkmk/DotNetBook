# Chapter 17 — Testing — Q & A

---

### Q1. `[Fact]` vs `[Theory]` in xUnit?

`[Fact]` is a single, parameterless test. `[Theory]` is data-driven — run once per data row, with arguments from `[InlineData]` (constants), `[MemberData]` (a static source), or `[ClassData]` (a class). Theories eliminate copy-pasted near-identical tests.

---

### Q2. How does xUnit handle setup/teardown and test isolation?

xUnit creates a **new instance of the test class per test method** (isolation by design). Setup goes in the **constructor**, teardown in **`Dispose`** (async: `IAsyncLifetime`). There's no `[SetUp]`/`[TearDown]`.

---

### Q3. What are xUnit fixtures for?

Sharing expensive setup. **`IClassFixture<T>`** shares one instance across tests in a class; **`ICollectionFixture<T>`** + `[Collection]` shares across multiple classes. Use them for databases, web hosts — anything costly to recreate per test.

---

### Q4. How does xUnit parallelism work, and what's the hazard?

Tests in **different classes run in parallel** (within a class, sequentially). The hazard: shared static/mutable state races, causing flaky tests. Keep tests independent; put tests sharing a resource in the same `[Collection]` so they run serially.

---

### Q5. What's the key lifecycle difference between xUnit and NUnit/MSTest?

xUnit makes a **new class instance per test** (fresh state). NUnit/MSTest **reuse one instance** across the class's tests (with `[SetUp]`/`[TestInitialize]` per test) — so instance state persists and must be reset, or you get order-dependent flakiness.

---

### Q6. Define stub, mock, fake.

**Stub**: returns canned values (no verification). **Mock**: a stub you also verify interactions on. **Fake**: a working but simplified implementation (e.g., an in-memory repository). "Mock" is often used loosely for all doubles.

---

### Q7. What should you mock vs not mock?

Mock **abstractions you own** that are slow/nondeterministic/side-effecting (repositories, HTTP boundary, `TimeProvider`, email). Don't mock value objects/POCOs (construct them), types you don't own with complex behavior (wrap them in an interface), or the SUT. Don't **over-mock** — it tests implementation, not behavior, and is brittle.

---

### Q8. How do you mock `HttpClient` and `DateTime.Now`?

`HttpClient` isn't an interface — mock the **`HttpMessageHandler`** it wraps, or abstract it behind a typed-client interface. `DateTime.Now` is static and unmockable — inject **`TimeProvider`** (or `IClock`) and control it in tests.

---

### Q9. Why use FluentAssertions/Shouldly, and what's the standout feature?

They make assertions read like English and produce **rich failure messages** (subject, expectation, actual, which member differed). FluentAssertions' standout is **`BeEquivalentTo`** — structural, member-by-member comparison of objects/collections, ideal for DTO/mapping tests. Note FluentAssertions v8+ is commercially licensed; Shouldly is free/OSS.

---

### Q10. What does `WebApplicationFactory<T>` do?

Boots your real ASP.NET Core app **in-memory** on a `TestServer` (no socket) and gives an `HttpClient` that sends requests through the **full production pipeline** (routing, middleware, filters, binding, DI, serialization) — catching wiring bugs unit tests can't.

---

### Q11. How do you replace a dependency in an integration test?

`factory.WithWebHostBuilder(b => b.ConfigureTestServices(services => { ... }))` — it runs after the app's registrations, so you can `RemoveAll<T>()` and re-add a test version (a fake email sender, a test database). Substitute external/nondeterministic deps; keep the rest real.

---

### Q12. How do you test `[Authorize]` endpoints?

Register a **test authentication handler** (via `ConfigureTestServices`) that returns a known `ClaimsPrincipal` with the needed roles/claims, and set the client's auth header to that scheme. This tests authorization logic without a real identity provider.

---

### Q13. Why prefer Testcontainers over EF Core InMemory?

The InMemory provider isn't relational — it doesn't run real SQL, enforce constraints, or apply migrations, so it hides query/constraint bugs. **Testcontainers** runs your *actual* database (PostgreSQL/SQL Server) in a throwaway Docker container — production fidelity (real SQL, constraints, transactions, migrations).

---

### Q14. What does Testcontainers manage, and how do you keep it fast?

It starts/stops **real dependencies in Docker containers** (databases, Redis, RabbitMQ, Kafka, Azurite, any image) from test code, via `IAsyncLifetime`/fixtures. Keep it fast by **sharing** one container across tests (collection fixture) and **resetting data** between tests (Respawn/truncate) rather than a container per test. Requires Docker on CI.

---

### Q15. What is bUnit and what's it for?

bUnit unit-tests Blazor components **in-memory** (no browser/server): render with `TestContext.RenderComponent<T>`, query markup, trigger events, inject fakes, assert with `MarkupMatches`. It tests component **logic**, not styling/real-browser behavior — pair with Playwright for E2E. (Depth in [Ch14 §12](../14-Blazor/12-Testing.md).)

---

### Q16. Why is naive `Stopwatch` benchmarking misleading?

JIT warmup/tiered compilation, dead-code elimination (the JIT deletes unused results), GC noise, and CPU scaling all distort single-run timing. **BenchmarkDotNet** handles warmup, repeated/multi-process measurement, consumes returned values (no elimination), and reports statistics + allocations.

---

### Q17. What should you enable and watch in BenchmarkDotNet?

Enable **`[MemoryDiagnoser]`** (allocations often matter more than time — GC pressure). Use a `Baseline` for ratios and `[Params]` to see scaling. Watch **Mean/StdDev** (low StdDev = trustworthy), **Ratio**, and **Allocated/Gen0**. Always run **Release, not attached**; put setup in `[GlobalSetup]`; **return** results.

---

### Q18. What is property-based testing and what are FsCheck's two superpowers?

Asserting **properties true for all inputs**, with random generated inputs trying to falsify them. FsCheck's superpowers: **generation** (boundary-heavy random inputs catching unimagined edge cases) and **shrinking** (auto-reducing a failure to the **minimal** counterexample). Great for algorithms/parsers/serializers with clear invariants.

---

### Q19. Give examples of good properties.

Round-trip/inverse (`decode(encode(x))==x`), invariants (sort preserves length and orders), idempotence (`f(f(x))==f(x)`), commutativity, and oracle (match a known-correct implementation). The classic catch: `Math.Abs(int.MinValue)` overflows — found by property testing, missed by examples.

---

### Q20. What's the difference between the test pyramid and trophy?

**Pyramid**: many unit, some integration, few E2E. **Trophy**: weights **integration** heaviest (most confidence per test — bugs live in the seams) with **static analysis** as the free base. Both agree: keep **E2E few**, leverage static analysis, balance fast-isolated vs realistic.

---

### Q21. "Test behavior, not implementation" — what does it mean?

Couple tests to *what* code does (inputs → outputs/effects), not *how* (internal calls, over-mocking). Behavior-coupled tests survive refactors and document intent; implementation-coupled tests break on harmless changes and discourage refactoring.

---

### Q22. Is code coverage a good goal?

No. Coverage shows which lines tests execute — useful for finding **gaps**, but a poor target: 100% coverage with weak assertions tests nothing, and chasing the number produces low-value tests. **Mutation testing** (Stryker.NET) is a stronger signal — it measures whether tests actually *catch* changes.

---

### Q23. Why are flaky tests so damaging?

A flaky (nondeterministic) test makes the whole suite untrustworthy — people stop believing red builds and ignore failures, defeating the suite's purpose. Eliminate nondeterminism (time via `TimeProvider`, seeded randomness, no shared mutable state, reset DB data) — fix or quarantine flakiness immediately.

---

### Q24. What determinism practices keep tests reliable?

Inject **`TimeProvider`** (not `DateTime.Now`), **seed** randomness/Bogus, avoid order-dependence and shared mutable state, and **reset** integration-test data between runs (transactions/Respawn). Deterministic tests are reproducible; nondeterminism causes flakiness.
