# Testcontainers

## Real dependencies, disposable, in code

Mocks and in-memory fakes can't reproduce the real behavior of a database, message broker, or cache — dialect-specific SQL, real constraints, actual broker semantics. **Testcontainers** (the `Testcontainers` .NET library) spins up **real dependencies in throwaway Docker containers** directly from your test code: a real PostgreSQL/SQL Server/Redis/RabbitMQ instance starts before the test, gets a dynamic connection string, and is destroyed afterward. This gives integration tests ([05-IntegrationTests.md](05-IntegrationTests.md)) **production-fidelity** without manual environment setup or shared test servers — the container's lifecycle is managed in code.

```csharp
public class OrderRepoTests : IAsyncLifetime {
    private readonly PostgreSqlContainer _db = new PostgreSqlBuilder()
        .WithImage("postgres:16")
        .Build();

    public async Task InitializeAsync() => await _db.StartAsync();   // start before tests
    public async Task DisposeAsync() => await _db.DisposeAsync();    // destroy after

    [Fact]
    public async Task Can_save_and_load_order() {
        var conn = _db.GetConnectionString();    // dynamic; points at the container
        // ... run migrations, exercise the real repository against a real PostgreSQL ...
    }
}
```

---

## Why real beats in-memory

The EF Core InMemory provider and hand-rolled fakes diverge from real databases in ways that hide bugs ([Ch05 §14](../05-EFCore/14-Testing.md)):

| Concern | In-memory fake | **Testcontainers (real DB)** |
|---|---|---|
| SQL dialect / raw SQL | not executed | **real** |
| Constraints (FK, unique, checks) | not enforced | **enforced** |
| Transactions / concurrency | simplified | **real** |
| Migrations | untested | **actually applied** |
| Provider-specific behavior (JSON columns, etc.) | absent | **present** |

For repositories, query-heavy code, migrations, and anything touching SQL specifics, a real containerized database catches bugs that in-memory tests pass right over. The cost is speed (containers take seconds to start) and a Docker requirement.

---

## Beyond databases

Testcontainers isn't just for databases — it has modules (or supports generic containers) for the whole dependency landscape, so integration tests can exercise real infrastructure ([Ch06](../06-DataAndCaching/README.md), [Ch07 Messaging](../07-Messaging/README.md)):

```csharp
var redis    = new RedisBuilder().Build();
var rabbit   = new RabbitMqBuilder().Build();
var kafka    = new KafkaBuilder().Build();
var mssql    = new MsSqlBuilder().Build();
var azurite  = new AzuriteBuilder().Build();      // Azure Storage emulator
// Generic container for anything with an image:
var custom   = new ContainerBuilder().WithImage("myorg/dep:tag").WithPortBinding(8080, true).Build();
```

This lets you test caching against **real Redis**, message handlers against **real RabbitMQ/Kafka**, and storage against an emulator — far higher fidelity than mocking the client. The container exposes the dynamic host/port to wire your code to it.

---

## Lifecycle and sharing

Containers are managed via xUnit's `IAsyncLifetime` ([01-xUnitDeep.md](01-xUnitDeep.md)) or fixtures:

- **Per-class** (`IAsyncLifetime` in the test class) — a fresh container per test class; clean isolation, but a container per class adds startup cost.
- **Shared** (collection fixture starting the container once) — one container across many test classes; much faster, but you must **reset state between tests** (truncate/respawn tables) to keep tests independent.

```csharp
[CollectionDefinition("db")]
public class DbCollection : ICollectionFixture<PostgresFixture> { }   // one container for the collection
```

Balance fidelity/isolation against startup cost: share the container (fast) and reset data per test (e.g., `Respawn` — [05-IntegrationTests.md](05-IntegrationTests.md)), rather than a new container per test.

---

## CI considerations

- **Docker must be available** on the CI runner (most CI providers support Docker-in-CI). Tests fail fast if Docker isn't present — gate or skip them where Docker is unavailable (e.g., a `[Trait]`/category to exclude on dev machines without Docker).
- **Startup time**: containers add seconds per image pull/start. Cache images on the runner, share containers across tests, and consider running the slower container-backed suite separately from fast unit tests (the test pyramid — [10-TestStrategy.md](10-TestStrategy.md)).
- **Resource cleanup**: Testcontainers uses a "Ryuk" reaper to clean up containers even if a test crashes — so orphaned containers don't pile up on CI.

---

## Common gotchas

### No Docker available

Testcontainers requires Docker. On machines/CI without it, the tests can't run — categorize them so they're skipped where Docker is absent, and ensure CI has Docker enabled.

### A container per test (slow)

Starting a fresh container for every test is very slow. **Share** one container (collection fixture) and **reset data** between tests instead.

### Forgetting to reset shared state

A shared container accumulates data across tests, causing order-dependence/pollution. Truncate/respawn between tests for independence.

### Pinning `latest` image tags

Using `:latest` makes tests non-reproducible (the image changes under you). Pin a specific version (`postgres:16`) so tests are deterministic.

### Treating container tests like unit tests in the pyramid

They're slow integration tests — don't run hundreds of them on every keystroke. Keep the fast unit-test layer broad and the container-backed layer focused ([10-TestStrategy.md](10-TestStrategy.md)).

---

## Summary

- **Testcontainers** starts **real dependencies in throwaway Docker containers** from test code (managed via `IAsyncLifetime`/fixtures), giving integration tests **production fidelity** with no manual environment setup — the container is created and destroyed around the test.
- A **real database** catches what in-memory fakes miss: SQL dialect, **constraints**, transactions/concurrency, **migrations**, and provider-specific behavior — prefer it for repositories and query-heavy code ([Ch05 §14](../05-EFCore/14-Testing.md)).
- It covers the whole landscape — **Redis, RabbitMQ, Kafka, SQL Server/PostgreSQL, Azurite**, or any image — so caching/messaging/storage tests run against real infrastructure ([Ch06](../06-DataAndCaching/README.md), [Ch07](../07-Messaging/README.md)).
- **Share** a container (collection fixture) and **reset state** per test for speed + isolation; pin image versions; require **Docker** (gate tests where unavailable) and place the slower container suite appropriately in the **test pyramid** ([10-TestStrategy.md](10-TestStrategy.md)).

→ Next: [07-Bunit.md](07-Bunit.md)
