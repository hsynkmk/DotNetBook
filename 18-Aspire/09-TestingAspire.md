# Testing Aspire Apps

## Integration tests over the whole app graph

Unit tests cover logic, and `WebApplicationFactory` tests a single ASP.NET Core app in-memory ([Ch17 §05](../17-Testing/05-IntegrationTests.md)). But an Aspire app is a *graph* — multiple services plus real backing resources (Postgres, Redis, a broker) wired together. **`Aspire.Hosting.Testing`** lets you spin up the **entire app model** (the AppHost graph, with its real containerized dependencies) in a test, then exercise it through real HTTP clients with service discovery and connection strings wired exactly as in development. It's the highest-fidelity integration test for a distributed .NET app: the same composition that runs locally, driven by a test.

```csharp
public class OrderFlowTests {
    [Fact]
    public async Task Placing_an_order_persists_and_notifies() {
        var appHost = await DistributedApplicationTestingBuilder
            .CreateAsync<Projects.AppHost>();          // the real AppHost
        await using var app = await appHost.BuildAsync();
        await app.StartAsync();                         // brings up the whole graph (services + containers)

        var client = app.CreateHttpClient("orderapi");  // discovery-resolved client to the API
        var resp = await client.PostAsJsonAsync("/orders", new { Total = 100 });

        resp.StatusCode.Should().Be(HttpStatusCode.Created);
    }
}
```

---

## `DistributedApplicationTestingBuilder`

The entry point is `DistributedApplicationTestingBuilder.CreateAsync<Projects.AppHost>()` — it loads your **real AppHost project**, so the test uses the *actual* app model (same resources, references, integrations as production-shaped dev). You then:

1. **Build** the distributed application (`BuildAsync`).
2. **Start** it (`StartAsync`) — Aspire launches the services and their backing containers (Postgres, Redis, etc.), waiting for readiness.
3. **Get clients** to services via `CreateHttpClient("servicename")` — the client is wired with **service discovery** ([04-ServiceDiscovery.md](04-ServiceDiscovery.md)), so it resolves the service's real test endpoint.
4. **Exercise** the system through those clients and **assert** on results (and, because real resources are running, on persisted state).

This tests the **seams across services** — routing, serialization, service-to-service calls, database persistence, cache behavior, message publishing — end to end, with everything wired as it really is.

---

## Relationship to `WebApplicationFactory`

The two are complementary, at different scopes ([Ch17 §05](../17-Testing/05-IntegrationTests.md)):

| | `WebApplicationFactory<T>` | `Aspire.Hosting.Testing` |
|---|---|---|
| Scope | **one** ASP.NET Core app, in-memory | the **whole app graph** (multiple services + real resources) |
| Dependencies | substituted/mocked or in-memory | **real** (containerized via Aspire) |
| Transport | in-memory `TestServer` (no socket) | real HTTP across started services |
| Fidelity | high for one app's pipeline | highest for distributed behavior |
| Speed | fast | slower (starts containers) |

Use **`WebApplicationFactory`** to test a single service's pipeline quickly (substituting external deps); use **`Aspire.Hosting.Testing`** to test **cross-service flows** with real dependencies. They sit at adjacent layers of the test strategy ([Ch17 §10](../17-Testing/10-TestStrategy.md)) — many fast single-service tests, fewer full-graph tests.

---

## Real dependencies = Testcontainers-grade fidelity

Because Aspire starts the actual backing resources (a real Postgres/Redis container, not an in-memory fake), these tests have the **production fidelity** discussed for Testcontainers ([Ch17 §06](../17-Testing/06-TestContainers.md)): real SQL, real constraints, real broker semantics. In effect, Aspire *is* orchestrating Testcontainers-like real dependencies for you, as part of the app model — so you don't separately wire up containers in the test; starting the AppHost graph brings them up. (This requires **Docker** on the test machine/CI, same as Testcontainers.)

---

## Practical concerns

- **Speed/isolation**: starting the full graph (with containers) is slow — share it across a test class/collection ([Ch17 §01](../17-Testing/01-xUnitDeep.md)) rather than per test, and **reset data between tests** (truncate/respawn) for independence.
- **Overriding for tests**: you can customize the app model for testing (e.g., swap an external dependency, add a test parameter) before `BuildAsync`, similar in spirit to `ConfigureTestServices`.
- **CI**: needs Docker; gate/categorize these slower tests so they run in an appropriate CI stage, not on every keystroke ([Ch17 §10](../17-Testing/10-TestStrategy.md)).
- **Waiting for readiness**: the test should wait for resources/services to be healthy before asserting (Aspire's start + `WaitFor` semantics help), or early requests race startup.

---

## Common gotchas

### Starting the full graph per test

Bringing up multiple services + containers for every test is very slow. Share the started application across a class/collection and **reset state** between tests.

### Forgetting Docker / readiness

These tests start real containers (Docker required) and need services healthy before requests — assert after startup completes, and ensure CI has Docker. Without readiness waits, early calls flake.

### Using it where `WebApplicationFactory` suffices

Full-graph tests are slower and heavier. For testing a *single* service's pipeline, prefer `WebApplicationFactory` ([Ch17 §05](../17-Testing/05-IntegrationTests.md)); reserve `Aspire.Hosting.Testing` for genuine cross-service flows.

### Not resetting shared resource state

A shared Postgres/Redis accumulates data across tests, causing order-dependence. Reset (Respawn/truncate/flush) between tests for isolation ([Ch17 §06](../17-Testing/06-TestContainers.md)).

### Treating it as a unit test in the pyramid

It's a slow, high-fidelity integration test — keep these focused and few, with the fast unit/single-service layer broad ([Ch17 §10](../17-Testing/10-TestStrategy.md)).

---

## Summary

- **`Aspire.Hosting.Testing`** spins up the **entire Aspire app graph** (the real AppHost — services + real containerized dependencies) in a test via **`DistributedApplicationTestingBuilder.CreateAsync<Projects.AppHost>()`** → `BuildAsync` → `StartAsync`, then exercises services through **discovery-wired** `CreateHttpClient("name")` — the highest-fidelity test of **cross-service flows**.
- It **complements `WebApplicationFactory`**: that tests **one** app's pipeline in-memory (fast, deps substituted); Aspire testing tests the **whole graph with real dependencies** (slower, highest fidelity) — adjacent layers of the test strategy.
- Real backing resources give **Testcontainers-grade fidelity** ([Ch17 §06](../17-Testing/06-TestContainers.md)) (real SQL/constraints/broker semantics) — Aspire orchestrates them as part of the model; requires **Docker**.
- **Practically**: share the started graph (don't start per test), **reset data** between tests, wait for **readiness**, gate the slow suite in CI, and reserve it for genuine distributed flows ([Ch17 §10](../17-Testing/10-TestStrategy.md)).

→ Next: [Questions.md](Questions.md)
