# Integration Testing with WebApplicationFactory

## Testing the whole app, in-memory

Unit tests isolate a single class; **integration tests** verify that the pieces work *together* — routing, model binding, middleware, filters, DI wiring, serialization, the actual request pipeline ([Ch04 ASP.NET Core](../04-AspNetCore/README.md)). For ASP.NET Core, **`WebApplicationFactory<TEntryPoint>`** boots your real application **in-memory** (no network, no Kestrel port) and gives you an `HttpClient` that sends real HTTP requests through the full pipeline. This catches the bugs unit tests can't: a misconfigured route, a missing service registration, a broken filter, a serialization mismatch.

```csharp
public class ApiTests : IClassFixture<WebApplicationFactory<Program>> {
    private readonly HttpClient _client;
    public ApiTests(WebApplicationFactory<Program> factory) => _client = factory.CreateClient();

    [Fact]
    public async Task Get_products_returns_ok() {
        var response = await _client.GetAsync("/products");
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var products = await response.Content.ReadFromJsonAsync<List<Product>>();
        products.Should().NotBeEmpty();
    }
}
```

---

## How it works

`WebApplicationFactory<TEntryPoint>` uses your app's real `Program` (the entry point — hence `Program` must be accessible; in top-level-statement apps, add `public partial class Program { }` or it's auto-generated as internal — use `InternalsVisibleTo`). It:

1. Builds your app's host with all its real DI registrations, middleware, and endpoints.
2. Hosts it on an **in-memory `TestServer`** (no real socket — requests go straight into the pipeline).
3. Hands you an `HttpClient` wired to that server.

So a request from the test client traverses **the same pipeline as production** — middleware, routing, filters, model binding, your endpoint, serialization — making these tests high-fidelity. They're slower than unit tests but far faster than hitting a deployed instance.

---

## Customizing the host with `WithWebHostBuilder`

The key technique: **override services for the test** — swap a real dependency (e.g., the database, an external API client) for a test version — via `WithWebHostBuilder` + `ConfigureTestServices`:

```csharp
var client = factory.WithWebHostBuilder(builder => {
    builder.ConfigureTestServices(services => {
        // Replace the real email sender with a fake:
        services.RemoveAll<IEmailSender>();
        services.AddSingleton<IEmailSender, FakeEmailSender>();

        // Point EF Core at a test database:
        services.RemoveAll<DbContextOptions<AppDbContext>>();
        services.AddDbContext<AppDbContext>(o => o.UseSqlite(_connection));
    });
}).CreateClient();
```

`ConfigureTestServices` runs **after** the app's own registrations, so it can replace them. This lets you keep most of the app real while substituting the parts you don't want in a test (external services, or a real DB for a faster test double). Decide per dependency: substitute external/nondeterministic things; keep the rest real for fidelity.

---

## Authentication in integration tests

Endpoints behind `[Authorize]` ([Ch10 §06](../10-Identity/06-Authorization.md)) need an authenticated request. The clean approach is a **test authentication handler** that injects a known principal, registered via `ConfigureTestServices`:

```csharp
builder.ConfigureTestServices(services => {
    services.AddAuthentication("Test")
        .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>("Test", _ => { });
});
// TestAuthHandler returns a ClaimsPrincipal with the roles/claims the test needs.
```

Then set the client's auth header to the test scheme. This tests the *authorization* logic (policies/roles) without a real identity provider — substituting only the authentication step.

---

## Database strategy

Integration tests need data. Options, from least to most realistic:

- **EF Core InMemory provider** — fast, but **not a real database** (no relational constraints, no real SQL) — fine for simple wiring tests, misleading for query/constraint behavior.
- **SQLite in-memory** — a real relational engine, fast, closer to production than InMemory (but not your actual DB dialect).
- **Real database via Testcontainers** ([06-TestContainers.md](06-TestContainers.md)) — a throwaway Dockerized instance of your *actual* database (SQL Server/PostgreSQL) — the highest fidelity (catches dialect-specific SQL, migrations, constraints).

Use the most realistic option your speed budget allows; for query-heavy code, prefer a real database (Testcontainers) over InMemory, which can hide real query bugs ([Ch05 §14](../05-EFCore/14-Testing.md)).

---

## Isolation and parallelism

Integration tests share a host/database, so isolation matters ([01-xUnitDeep.md](01-xUnitDeep.md)):

- Use a **collection fixture** to share one factory/host across the suite (booting the host per test is slow).
- **Reset state between tests** — wrap each test in a transaction and roll back, or recreate/respawn the database (libraries like `Respawn` clear data between tests). Without this, tests pollute each other's data and become order-dependent.
- Mark database-touching tests into a **`[Collection]`** so they run serially against the shared resource (xUnit parallelizes classes by default).

---

## Common gotchas

### `Program` not accessible

`WebApplicationFactory<Program>` needs the entry point visible. With top-level statements, add `public partial class Program { }` or use `InternalsVisibleTo` — otherwise the factory can't find it.

### EF Core InMemory hiding real bugs

The InMemory provider isn't relational — it won't enforce constraints or run real SQL, so query/constraint behavior differs from production. Use SQLite or Testcontainers for fidelity ([06-TestContainers.md](06-TestContainers.md)).

### No state reset between tests

Sharing a database without resetting it between tests causes data pollution and order-dependent flakiness. Use transactions/rollback or Respawn; serialize via a collection.

### Mocking too much (defeating the point)

Replacing the database, auth, and half the services turns an integration test back into a unit test. Keep most of the app real; substitute only external/nondeterministic dependencies.

### Booting the host per test

Creating a new factory/host for every test is slow. Share it via a class/collection fixture and reuse the `HttpClient`.

---

## Summary

- **`WebApplicationFactory<TEntryPoint>`** boots your real ASP.NET Core app **in-memory** (on a `TestServer`, no socket) and gives an `HttpClient` that sends requests through the **full production pipeline** — catching routing/DI/middleware/serialization bugs unit tests miss.
- Customize via **`WithWebHostBuilder` + `ConfigureTestServices`** to **replace specific dependencies** (external services, the database, auth) while keeping the rest real — runs after the app's registrations, so it overrides them.
- Handle `[Authorize]` endpoints with a **test authentication handler** injecting a known principal; choose a **database strategy** by fidelity need (EF InMemory < SQLite < real DB via **Testcontainers** — prefer real for query-heavy code).
- **Isolate**: share the host via a **collection fixture**, **reset state** between tests (transaction rollback / Respawn), and serialize DB tests in a `[Collection]`. Make `Program` accessible (`public partial class Program`).

→ Next: [06-TestContainers.md](06-TestContainers.md)
