# Testing EF Core Code

## How do you test data-access logic?

Testing code that uses `DbContext` raises a question: do you mock EF, use a fake provider, or hit a real database? The answer depends on what you're testing — and the popular shortcut (the InMemory provider) has serious traps. The fidelity ladder: **InMemory** (low) → **SQLite in-memory** (medium) → **Testcontainers** (high, real DB).

> CSharpBook Ch16 §05 covers integration testing with `WebApplicationFactory` and Testcontainers generally; this file is the EF-specific guidance.

---

## Don't mock `DbContext`/`DbSet`

Mocking `DbSet`/`IQueryable` (Moq etc.) to fake EF is **a bad idea**: `IQueryable` over a mock runs LINQ-to-Objects, not LINQ-to-Entities — so it won't catch translation errors, behaves differently for async, and you end up testing your mock setup, not real behavior. Don't mock EF; use a real (or realistic) provider instead. For testing a *service* that depends on a repository abstraction, mock the **repository interface** (CSharpBook Ch16 §03), not `DbContext`.

---

## Option 1: InMemory provider (low fidelity — beware)

```csharp
var options = new DbContextOptionsBuilder<AppDbContext>()
    .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())   // unique per test
    .Options;
using var db = new AppDbContext(options);
```

The InMemory provider is fast and easy — but it is **not a relational database**, and Microsoft explicitly **does not recommend it** for testing query logic. Its traps:
- **No SQL translation** — it can't catch queries that won't translate to real SQL (a query that works in InMemory may throw against SQL Server/Postgres).
- **No relational constraints** — no FK/unique/not-null enforcement, no cascade behavior; tests pass that would fail on a real DB.
- **No transactions** (they're no-ops), **different query semantics** (case sensitivity, ordering, null handling differ).

Use InMemory only for the simplest non-relational logic (and even then, prefer SQLite). It gives **false confidence** — a green test against InMemory doesn't mean the query works in production.

---

## Option 2: SQLite in-memory (medium fidelity — good default)

A **real SQL engine** running in memory — far more faithful than the InMemory provider (real SQL translation, constraints, transactions), while staying fast and dependency-free:

```csharp
public class SqliteTestFixture : IDisposable {
    private readonly SqliteConnection _connection;
    public AppDbContext Db { get; }

    public SqliteTestFixture() {
        _connection = new SqliteConnection("DataSource=:memory:");
        _connection.Open();                                 // keep OPEN — closing drops the in-memory DB
        var options = new DbContextOptionsBuilder<AppDbContext>().UseSqlite(_connection).Options;
        Db = new AppDbContext(options);
        Db.Database.EnsureCreated();                         // create schema from the model
    }
    public void Dispose() { Db.Dispose(); _connection.Dispose(); }
}
```

SQLite-in-memory catches translation errors, enforces constraints/transactions, and validates real query behavior — a great default for fast EF tests. Caveat: SQLite isn't your **production** engine, so some SQL Server/Postgres-specific features, types, and behaviors (specific functions, JSON support, certain constraints) differ. For those, use Testcontainers.

> Keep the `SqliteConnection` **open** for the test's lifetime — an in-memory SQLite DB exists only while a connection is open; closing it drops the data.

---

## Option 3: Testcontainers (highest fidelity)

Run your **actual production database** (Postgres, SQL Server, etc.) in a throwaway Docker container per test run — production-equivalent behavior:

```csharp
public class PostgresFixture : IAsyncLifetime {
    private readonly PostgreSqlContainer _db = new PostgreSqlBuilder().WithImage("postgres:16").Build();
    public AppDbContext CreateContext() {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(_db.GetConnectionString()).Options;
        return new AppDbContext(options);
    }
    public async Task InitializeAsync() {
        await _db.StartAsync();
        await using var ctx = CreateContext();
        await ctx.Database.MigrateAsync();                   // apply real migrations
    }
    public async Task DisposeAsync() => await _db.DisposeAsync();
}
```

Testcontainers gives the **highest fidelity** — your real DB engine, real migrations, real SQL, real constraints and behaviors — catching everything the others miss. Cost: needs Docker, slower startup. Best practice: **start the container once per collection** (`IAsyncLifetime` + `ICollectionFixture` — CSharpBook Ch16 §01/§05) and reset data between tests, rather than a container per test. Use Testcontainers for the tests that must trust real DB behavior (complex queries, DB-specific features, migration validation).

---

## Choosing & isolation

| Approach | Fidelity | Speed | Catches translation/constraints? | Use for |
|---|---|---|---|---|
| Mock `DbContext` | — | — | no | **don't** |
| InMemory provider | low | fastest | no | trivial non-relational logic only |
| SQLite in-memory | medium | fast | mostly (real SQL engine) | **default** for fast EF tests |
| Testcontainers | high | slower | fully (real DB) | query-heavy / DB-specific / migration tests |

**Test isolation** matters regardless of approach — each test should start from known state:
- Unique database name per test (InMemory), or fresh connection (SQLite), or
- **Respawn** (resets a real DB to empty quickly between tests), transaction-rollback-per-test, or unique data per test (enables parallelism).

---

## What to test where

- **Unit-test business logic** that depends on data via a **repository interface mock** — fast, no DB (CSharpBook Ch16 §03).
- **Integration-test EF query/mapping logic** against **SQLite or Testcontainers** — does the query translate, return the right data, enforce constraints?
- **Integration-test the full API** with `WebApplicationFactory` + a real-ish DB (CSharpBook Ch16 §05, [Ch17 Testing](../17-Testing/README.md)) — the whole vertical slice.
- **Validate migrations** against Testcontainers (apply them to a real engine).

Match the test type to what you're verifying; don't integration-test pure business logic, and don't unit-test (mock) the data layer.

---

## Common gotchas

### Mocking `DbContext`/`DbSet`

You test the mock, not EF; it misses translation/async/constraint behavior. Mock a repository interface, or use a real provider.

### Trusting the InMemory provider

It's not relational — passes tests that fail in production (no constraints, no SQL translation, different semantics). Microsoft advises against it; prefer SQLite.

### SQLite connection closed → empty DB

An in-memory SQLite database vanishes when its connection closes. Keep one connection open for the test's lifetime.

### Container-per-test slowness

Starting a Testcontainer per test is very slow. Start once per collection; reset data between tests.

### Tests sharing DB state

Parallel/sequential tests leaking state into each other → flaky failures. Isolate (unique DB/data, Respawn, transaction rollback).

### Testing only against SQLite, deploying to Postgres

SQLite differs from your production engine for some features. For DB-specific behavior, validate against the real engine (Testcontainers).

---

## Summary

- **Don't mock `DbContext`/`DbSet`** (you test the mock, miss translation/constraints) — mock a **repository interface** for unit tests, or use a real provider for data-layer tests.
- Fidelity ladder: **InMemory** (low — not relational, no constraints/translation; Microsoft advises against it), **SQLite in-memory** (medium — real SQL engine; the good **default**, keep the connection open), **Testcontainers** (high — your real DB; for query-heavy/DB-specific/migration tests, start once per collection).
- **Unit-test business logic** via repository mocks; **integration-test EF queries/mapping** against SQLite/Testcontainers; **validate migrations** against the real engine.
- Ensure **test isolation** (unique data, Respawn, or transaction rollback); don't trust InMemory or SQLite for production-DB-specific behavior.
- General testing patterns (`WebApplicationFactory`, Testcontainers, fixtures): CSharpBook Ch16, [Ch17 Testing](../17-Testing/README.md).

→ Next: [Questions.md](Questions.md)
