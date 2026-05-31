# Domain and Persistence

## Separating *what the business does* from *how it's stored*

A recurring design tension: your **domain** (the business rules, entities, and behavior) and your **persistence** (how data is stored — EF Core, a database, tables) are *different concerns*, yet they're easy to entangle. When persistence concerns leak into the domain — entities shaped by table columns, business logic scattered across services that manipulate property bags — you get an **anemic domain model** that's hard to reason about and protect. Keeping the domain focused on **behavior and invariants**, and persistence as a separable detail ([01-SolutionLayout.md](01-SolutionLayout.md)), produces code that's testable, expresses the business clearly, and resists corruption.

---

## Anemic vs rich domain models

The contrast that frames this whole topic:

```csharp
// ✗ Anemic — a property bag; logic lives elsewhere (in "services" that mutate it)
public class Order {
    public List<OrderLine> Lines { get; set; } = new();
    public OrderStatus Status { get; set; }
    public decimal Total { get; set; }
}
// ...somewhere far away:
order.Status = OrderStatus.Shipped;     // anyone can set any state, any time — no rules enforced
order.Total = order.Lines.Sum(...);     // invariant maintenance scattered & easily forgotten
```

```csharp
// ✓ Rich — behavior and invariants live with the data
public class Order {
    private readonly List<OrderLine> _lines = new();
    public IReadOnlyList<OrderLine> Lines => _lines;          // encapsulated — no external mutation
    public OrderStatus Status { get; private set; }           // state changes only via methods
    public decimal Total => _lines.Sum(l => l.Subtotal);      // invariant computed, can't drift

    public void AddLine(Product p, int qty) {
        if (Status != OrderStatus.Draft) throw new InvalidOperationException("Can't modify a placed order.");
        _lines.Add(new OrderLine(p, qty));
    }
    public void Ship() {
        if (Status != OrderStatus.Paid) throw new InvalidOperationException("Only paid orders can ship.");
        Status = OrderStatus.Shipped;
    }
}
```

- **Anemic** — entities are data bags with public setters; logic lives in external "service" classes that manipulate them. The business rules are *scattered*, invariants are *unprotected* (anyone can set `Status` to anything), and the model doesn't express the domain. (It's listed as an anti-pattern — [10-AntiPatterns.md](10-AntiPatterns.md).)
- **Rich** — entities own their behavior and **protect their invariants**: state changes go through **methods** that enforce rules, collections are **encapsulated**, and invalid states are **unrepresentable**. The model *is* the business logic.

Rich models concentrate rules where the data lives (high cohesion, encapsulation — [CSharpBook Ch02](../../CSharpBook/02-OOP/README.md)), so you can't accidentally violate an invariant from afar.

> **Caveat**: not every system needs a rich domain model. A simple CRUD app may be fine with anemic DTOs ([04-CQRS.md](04-CQRS.md)). Richness pays off when there's **genuine business complexity** (rules, invariants, workflows) to protect — match the modeling effort to the domain's complexity.

---

## Keeping persistence out of the domain

The domain should express business concepts, not storage mechanics. Practices that keep persistence separable ([01-SolutionLayout.md](01-SolutionLayout.md)):

- **Domain declares interfaces; infrastructure implements them** — `IOrderRepository` lives with the domain/application; the EF Core implementation lives in Infrastructure (Dependency Inversion — [CSharpBook Ch17 SOLID](../../CSharpBook/17-BestPractices/README.md)). The domain never references EF Core.
- **Map, don't leak** — EF Core can map a **rich** model (private fields, encapsulated collections, value objects) to tables via configuration ([Ch05 §04](../05-EFCore/04-Relationships.md)), so persistence adapts to the domain, not vice versa. You don't need public setters for EF Core.
- **Value objects** for concepts without identity (Money, Address, DateRange) — immutable, equality-by-value ([CSharpBook Ch03 §03](../../CSharpBook/03-TypeSystem/README.md)), mapped as **owned types** ([Ch05 §12](../05-EFCore/12-OwnedTypes.md)). They make the model expressive and prevent primitive-obsession bugs.

The test: *can you unit-test the domain with no database?* If yes, persistence is properly separated.

---

## Aggregates — consistency boundaries

In domain modeling (DDD), an **aggregate** is a cluster of entities/value objects treated as a single unit for consistency, with one **aggregate root** as the entry point. The root enforces the aggregate's invariants and is the only thing external code references:

```csharp
// Order is the aggregate root; OrderLines are part of the aggregate.
// External code goes through Order — it can't manipulate OrderLines directly:
order.AddLine(product, 2);     // ✓ through the root (rules enforced)
// order.Lines.Add(...)        // ✗ can't — Lines is read-only; mutation only via the root
```

Aggregates define **transactional consistency boundaries**: changes within one aggregate are saved atomically (one `SaveChanges` — [Ch05 §01](../05-EFCore/01-DbContext.md)), and you reference other aggregates **by id**, not by direct object reference. This keeps invariants enforceable and transactions small — and aligns with how repositories ([03-RepositoryUnitOfWork.md](03-RepositoryUnitOfWork.md)) and domain events ([05-DomainEvents.md](05-DomainEvents.md)) work.

---

## Common gotchas

### Anemic model with scattered logic

Data-bag entities + logic in external services scatter rules and leave invariants unprotected. For complex domains, use **rich** models that own behavior and protect invariants ([10-AntiPatterns.md](10-AntiPatterns.md)).

### Persistence concerns leaking into the domain

Entities shaped by table columns, EF Core attributes/types in the domain, or public setters everywhere couple the domain to storage. Keep the domain persistence-agnostic; **map** a rich model via EF Core configuration ([Ch05 §04](../05-EFCore/04-Relationships.md)).

### Over-modeling a simple CRUD app

A rich domain model and aggregates add ceremony that a simple data-in/data-out app doesn't need. Match modeling richness to **business complexity** — anemic DTOs are fine for CRUD ([04-CQRS.md](04-CQRS.md)).

### Public setters defeating encapsulation

`{ get; set; }` everywhere lets any code put the entity in an invalid state. Use private setters + methods that enforce rules so invalid states are unreachable.

### Referencing other aggregates by object instead of id

Direct references across aggregate boundaries blur consistency boundaries and bloat transactions. Reference other aggregates **by id**; keep each aggregate's transaction self-contained.

---

## Summary

- Keep the **domain** (business behavior + invariants) separate from **persistence** (how it's stored) — entangling them yields an **anemic model** that's hard to protect and reason about.
- Prefer a **rich domain model** for complex domains: entities **own their behavior**, **protect invariants** (state changes via methods, encapsulated collections, no public setters), making invalid states unrepresentable — vs an **anemic** data-bag with logic scattered in services (an anti-pattern — [10-AntiPatterns.md](10-AntiPatterns.md)). Match richness to actual business complexity (CRUD can stay anemic).
- Keep persistence separable: **domain declares interfaces, infrastructure implements** them (the domain never references EF Core); **map** a rich model via EF Core config (no public setters needed); use **value objects** (owned types) for identity-less concepts. The test: *can you unit-test the domain with no database?*
- Model **aggregates** with a single **root** as the consistency/transaction boundary; reference other aggregates **by id**, save one aggregate atomically.

→ Next: [03-RepositoryUnitOfWork.md](03-RepositoryUnitOfWork.md)
