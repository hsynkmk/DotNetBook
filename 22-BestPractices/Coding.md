# Chapter 22 — Best Practices & Patterns — Coding Problems

Refactor an anemic model, introduce CQRS, extract a vertical slice, and retire anti-patterns. Each has a hidden solution — attempt it first.

---

### Problem 1 — Refactor an anemic model to a rich one

```csharp
public class Order {
    public List<OrderLine> Lines { get; set; } = new();
    public OrderStatus Status { get; set; }
    public decimal Total { get; set; }
}
// elsewhere: order.Status = OrderStatus.Shipped; order.Total = order.Lines.Sum(...);
```

Make it protect its invariants.

<details>
<summary>Solution</summary>

```csharp
public class Order {
    private readonly List<OrderLine> _lines = new();
    public IReadOnlyList<OrderLine> Lines => _lines;
    public OrderStatus Status { get; private set; } = OrderStatus.Draft;
    public decimal Total => _lines.Sum(l => l.Subtotal);   // computed — can't drift

    public void AddLine(Product p, int qty) {
        if (Status != OrderStatus.Draft) throw new InvalidOperationException("Order already placed.");
        _lines.Add(new OrderLine(p, qty));
    }
    public void Ship() {
        if (Status != OrderStatus.Paid) throw new InvalidOperationException("Only paid orders ship.");
        Status = OrderStatus.Shipped;
    }
}
```

Encapsulated collection, private setters, state changes via methods that enforce rules — invalid states are unreachable ([02-DomainPersistence.md](02-DomainPersistence.md)).
</details>

---

### Problem 2 — Replace Service Locator with constructor injection

```csharp
public class OrderService {
    public OrderService(IServiceProvider sp) => _sp = sp;
    public async Task ProcessAsync(int id) {
        var repo = _sp.GetService<IOrderRepository>();
        var email = _sp.GetService<IEmailSender>();
        // ...
    }
}
```

<details>
<summary>Solution</summary>

```csharp
public class OrderService(IOrderRepository repo, IEmailSender email) {
    public async Task ProcessAsync(int id) {
        var order = await repo.GetAsync(id);
        // ... use repo, email — dependencies are explicit and testable
    }
}
```

Dependencies are now visible in the constructor and trivially faked in tests — no container needed ([10-AntiPatterns.md](10-AntiPatterns.md), [Ch03 §02](../03-HostingAndDI/02-DependencyInjection.md)).
</details>

---

### Problem 3 — Introduce CQRS to a CRUD action

Convert a fat controller action that places an order into a command + handler.

<details>
<summary>Solution</summary>

```csharp
public record PlaceOrderCommand(int CustomerId, List<LineDto> Lines) : IRequest<int>;

public class PlaceOrderHandler(AppDbContext db) : IRequestHandler<PlaceOrderCommand, int> {
    public async Task<int> Handle(PlaceOrderCommand cmd, CancellationToken ct) {
        var order = Order.Place(cmd.CustomerId, cmd.Lines);   // domain enforces rules
        db.Orders.Add(order);
        await db.SaveChangesAsync(ct);
        return order.Id;
    }
}
// Thin controller / minimal API:
app.MapPost("/orders", (PlaceOrderCommand cmd, ISender m) => m.Send(cmd));
```

The controller is thin (delegates to the handler); business logic lives in the handler/domain ([04-CQRS.md](04-CQRS.md)).
</details>

---

### Problem 4 — Write a query that bypasses the domain

Add a `GetOrderSummary` query projecting directly to a DTO.

<details>
<summary>Solution</summary>

```csharp
public record GetOrderSummary(int OrderId) : IRequest<OrderSummaryDto>;
public record OrderSummaryDto(int Id, decimal Total, string Status);

public class GetOrderSummaryHandler(AppDbContext db) : IRequestHandler<GetOrderSummary, OrderSummaryDto> {
    public Task<OrderSummaryDto> Handle(GetOrderSummary q, CancellationToken ct) =>
        db.Orders.Where(o => o.Id == q.OrderId)
                 .Select(o => new OrderSummaryDto(o.Id, o.Total, o.Status.ToString()))   // project
                 .FirstAsync(ct);
}
```

Reads **project to a DTO** (no domain model, no loading whole aggregates) — optimized for the view, unlike writes which go through the domain ([04-CQRS.md](04-CQRS.md)).
</details>

---

### Problem 5 — Raise and handle a domain event

Have `Order.Place` raise an `OrderPlaced` event, handled to send a confirmation.

<details>
<summary>Solution</summary>

```csharp
public record OrderPlaced(int OrderId, int CustomerId) : IDomainEvent;

public class Order {
    private readonly List<IDomainEvent> _events = new();
    public IReadOnlyList<IDomainEvent> DomainEvents => _events;
    public static Order Place(int customerId, ...) {
        var o = new Order(customerId, ...);
        o._events.Add(new OrderPlaced(o.Id, customerId));   // record, don't dispatch
        return o;
    }
}
public class SendConfirmation : IDomainEventHandler<OrderPlaced> {
    public Task Handle(OrderPlaced e, CancellationToken ct) => _email.SendAsync(e.OrderId, ct);
}
// Infrastructure dispatches events after SaveChanges (e.g., a SaveChanges interceptor).
```

The aggregate records the event; infrastructure dispatches after persistence; one handler per reaction ([05-DomainEvents.md](05-DomainEvents.md)).
</details>

---

### Problem 6 — Extract a vertical slice

Reorganize the order-placement code from layered folders into a single feature slice.

<details>
<summary>Solution</summary>

```
Features/PlaceOrder/
├── PlaceOrderEndpoint.cs     // app.MapPost("/orders", ...)
├── PlaceOrderCommand.cs      // the command + handler
├── PlaceOrderValidator.cs    // validation for THIS feature
└── PlaceOrderResponse.cs
```

Everything for "place order" lives together — changing the feature is localized to this folder; it's independent of other slices. The slice shares the **domain** (`Order`) and infrastructure (DbContext) but not application glue ([06-VerticalSlices.md](06-VerticalSlices.md)).
</details>

---

### Problem 7 — Add tenant isolation with a global query filter

Ensure every `Order` query is scoped to the current tenant automatically.

<details>
<summary>Solution</summary>

```csharp
public class AppDbContext(ITenantContext tenant) : DbContext {
    protected override void OnModelCreating(ModelBuilder mb) {
        mb.Entity<Order>().HasQueryFilter(o => o.TenantId == tenant.TenantId);   // auto-applied
    }
}
```

The **global query filter** ([Ch05 §13](../05-EFCore/13-GlobalQueryFilters.md)) scopes *every* query to the tenant, so a forgotten manual `WHERE TenantId` can't leak data. Resolve `ITenantContext` from the authenticated identity, not client input ([07-MultiTenancy.md](07-MultiTenancy.md)).
</details>

---

### Problem 8 — Add a feature flag

Gate a new pricing engine behind a runtime-changeable flag.

<details>
<summary>Solution</summary>

```csharp
builder.Services.AddFeatureManagement();   // backed by Azure App Configuration for dynamic control

public class PricingService(IFeatureManager features) {
    public async Task<decimal> PriceAsync(Cart cart) =>
        await features.IsEnabledAsync("NewPricingEngine")
            ? NewEngine(cart) : LegacyEngine(cart);
}
```

The flag toggles at runtime (no redeploy) when backed by App Configuration — decoupling release from deployment; remove the flag once fully rolled out ([08-FeatureFlags.md](08-FeatureFlags.md), [Ch20 §09](../20-AzureIntegration/09-AppConfig.md)).
</details>

---

### Problem 9 — Version an API

Serve v1 and v2 of an orders endpoint so v1 clients aren't broken.

<details>
<summary>Solution</summary>

```csharp
builder.Services.AddApiVersioning(o => {
    o.DefaultApiVersion = new ApiVersion(1, 0);
    o.ReportApiVersions = true;
}).AddApiExplorer(o => o.GroupNameFormat = "'v'VVV");
```
```csharp
[ApiVersion("1.0"), ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/orders")]
public class OrdersController : ControllerBase {
    [HttpGet, MapToApiVersion("1.0")] public IActionResult GetV1() => ...;
    [HttpGet, MapToApiVersion("2.0")] public IActionResult GetV2() => ...;   // breaking changes here
}
```

`/api/v1/orders` and `/api/v2/orders` coexist; introduce breaking changes in v2 without breaking v1; deprecate v1 with notice before retiring ([09-Versioning.md](09-Versioning.md)).
</details>

---

### Problem 10 — Fix sync-over-async in a library

```csharp
public string GetConfig(string key) =>
    _httpClient.GetStringAsync($"/config/{key}").Result;   // ✗ in a library
```

<details>
<summary>Solution</summary>

```csharp
public async Task<string> GetConfigAsync(string key, CancellationToken ct = default) =>
    await _httpClient.GetStringAsync($"/config/{key}", ct).ConfigureAwait(false);
```

Make it async (no `.Result` → no starvation/deadlock), **accept and pass a `CancellationToken`**, and use **`ConfigureAwait(false)`** because it's library code that doesn't need the caller's context ([12-AsyncIdiomsAtScale.md](12-AsyncIdiomsAtScale.md), [10-AntiPatterns.md](10-AntiPatterns.md)).
</details>

---

### Problem 11 — Implement the outbox pattern (conceptually)

An order placement must reliably publish an `OrderPlaced` integration event. Avoid the dual-write problem.

<details>
<summary>Solution</summary>

```csharp
// 1) In the SAME transaction as the order, write the event to an outbox table:
db.Orders.Add(order);
db.OutboxMessages.Add(new OutboxMessage(typeof(OrderPlaced).Name,
    JsonSerializer.Serialize(new OrderPlaced(order.Id))));
await db.SaveChangesAsync(ct);   // order + outbox row commit atomically

// 2) A separate background worker reads unpublished outbox rows, publishes to the broker,
//    and marks them sent (at-least-once; consumers are idempotent).
```

The DB change and the outbox row commit together, so the event is published **iff** the order was saved — no dual-write race ([05-DomainEvents.md](05-DomainEvents.md), [Ch07 §07](../07-Messaging/07-Patterns.md)).
</details>

---

### Problem 12 — Shrink the public surface

A utility project exposes everything `public`. Reduce the surface and keep tests working.

<details>
<summary>Solution</summary>

```csharp
public class JsonExporter { }       // the one type consumers actually use
internal class JsonFormatter { }    // implementation detail — was public
internal class FieldMapper { }      // implementation detail — was public
```
```csharp
// In the .csproj, so tests can still reach internals:
[assembly: InternalsVisibleTo("MyApp.Utils.Tests")]
```

Default to **`internal`**; expose only the deliberate public contract; use `[InternalsVisibleTo]` for tests rather than making everything public ([11-CodeOrganization.md](11-CodeOrganization.md)).
</details>

---

### Problem 13 — Decide: repository or not?

You're building a CRUD admin panel over EF Core. A teammate proposes a generic `Repository<T>`. Argue for/against and decide.

<details>
<summary>Solution</summary>

**Against (and the decision)**: `DbContext`/`DbSet<T>` is already a Unit of Work + repository. A generic `Repository<T>` would duplicate it, likely **leak `IQueryable`** (no real abstraction) or **hide EF Core's power** (projections, `AsNoTracking`, `Include` — risking N+1/over-fetch). For a CRUD panel, use **`DbContext`/`DbSet` directly** (or CQRS handlers using EF Core).

**When you'd reconsider**: if the domain must not reference EF Core (Clean architecture) or you have complex repeated queries for an aggregate, introduce a **specific** `IOrderRepository` with intention-revealing methods — not a generic wrapper ([03-RepositoryUnitOfWork.md](03-RepositoryUnitOfWork.md)).
</details>

---

### Problem 14 — Spot the captive dependency

```csharp
builder.Services.AddSingleton<CacheWarmer>();   // singleton
public class CacheWarmer(AppDbContext db) { }    // DbContext is scoped
```

What's wrong and how to fix it?

<details>
<summary>Solution</summary>

A **captive dependency**: a **singleton** captures a **scoped** `DbContext`, keeping one DbContext alive for the app's lifetime (DbContext isn't thread-safe and accumulates tracked entities — corruption/leaks). Fix by resolving a scope per use:

```csharp
public class CacheWarmer(IServiceScopeFactory scopeFactory) {
    public async Task WarmAsync(CancellationToken ct) {
        using var scope = scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        // use db within this scope
    }
}
```

Match lifetimes; use `IServiceScopeFactory` to create a scope for scoped dependencies inside a singleton ([10-AntiPatterns.md](10-AntiPatterns.md), [Ch03 §03](../03-HostingAndDI/03-Lifetimes.md)).
</details>

---

### Problem 15 — Choose the architecture

A startup is building a new SaaS product: small team, evolving requirements, a handful of bounded areas (orders, billing, catalog), Azure-hosted. Recommend an architecture and justify.

<details>
<summary>Solution</summary>

**Start with a modular monolith** ([01-SolutionLayout.md](01-SolutionLayout.md)):

- **One deployable** — simplest to build/test/deploy for a small team and evolving requirements (no distributed-systems tax — [10-AntiPatterns.md](10-AntiPatterns.md) warns against premature microservices).
- **Strong module boundaries** — `Orders`, `Billing`, `Catalog` as modules with public contracts and `internal` internals, communicating via interfaces/domain events ([05-DomainEvents.md](05-DomainEvents.md)) — microservice-like boundaries **without** the complexity, and **easy to extract** a module into a service later if scaling/team needs justify it.
- **Within modules**: vertical slices ([06-VerticalSlices.md](06-VerticalSlices.md)) + lightweight CQRS ([04-CQRS.md](04-CQRS.md)) for change-locality; rich domain models where there's real complexity (billing), anemic/CRUD where there isn't (catalog admin).
- **Multi-tenancy** via per-row + global query filters initially ([07-MultiTenancy.md](07-MultiTenancy.md)), feature flags for safe rollout ([08-FeatureFlags.md](08-FeatureFlags.md)), and Aspire ([Ch18](../18-Aspire/README.md)) to compose it on Azure.

Defer microservices until a concrete need (independent scaling, separate team cadence) appears — the modular monolith preserves that option cheaply.
</details>
