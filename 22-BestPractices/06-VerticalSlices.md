# Vertical Slice Architecture

## Organizing by feature, not by layer

Traditional **layered** architecture ([01-SolutionLayout.md](01-SolutionLayout.md)) groups code by *technical concern* — all controllers together, all services together, all repositories together. **Vertical slice architecture** organizes by *feature* instead: each use case (e.g., "Place Order," "Get Order Summary") is a **self-contained slice** containing everything it needs — the endpoint, the command/query, its handler, validation, and data access — in one place. The argument: you change software **by feature**, not by layer, so code that **changes together should live together**. A slice cuts vertically through the layers rather than spreading one feature's logic across many horizontal folders.

```
Features/
├── PlaceOrder/
│   ├── PlaceOrderEndpoint.cs       ← the route
│   ├── PlaceOrderCommand.cs        ← request + handler ([04-CQRS.md])
│   ├── PlaceOrderValidator.cs      ← validation for THIS feature
│   └── PlaceOrderResponse.cs
├── GetOrderSummary/
│   ├── GetOrderSummaryEndpoint.cs
│   ├── GetOrderSummaryQuery.cs     ← query + handler (projects to a DTO)
│   └── OrderSummaryDto.cs
└── CancelOrder/
    └── ...
```

---

## Layers vs slices

The contrast:

```
LAYERED (horizontal)                    VERTICAL SLICES (by feature)
  Controllers/                            Features/
    OrderController.cs   ─┐                 PlaceOrder/      ← everything for this feature
  Services/               │                   endpoint, command, handler, validator
    OrderService.cs       ├─ one feature      GetOrderSummary/
  Repositories/           │  spread across     ...
    OrderRepository.cs   ─┘  many folders
  DTOs/
    ...
```

- **Layered** — strong *technical* separation, but one feature's logic is **scattered** across folders; adding a feature touches many places, and unrelated features share layers (changes risk coupling).
- **Vertical slices** — one feature's code is **co-located**; adding/changing a feature is localized to its slice; slices are **independent** (low coupling between features). The trade-off: less enforced technical layering, and potential duplication across slices (often acceptable — see below).

Vertical slices optimize for **change locality and feature independence**; layers optimize for **technical uniformity**. Both are valid; slices have grown popular for feature-rich apps (and pair naturally with CQRS — [04-CQRS.md](04-CQRS.md) — and Minimal APIs — [Ch04 §02](../04-AspNetCore/02-MinimalAPIs.md)).

---

## Why slices reduce coupling

The core benefit is **low coupling between features** and **high cohesion within a feature**:

- Changing "Place Order" touches only the `PlaceOrder/` slice — you don't risk breaking "Get Order Summary" because they don't share a fat `OrderService` ([10-AntiPatterns.md](10-AntiPatterns.md)) or repository.
- Each slice can be **optimized for its own needs** — a query slice projects directly to a DTO (fast reads), a command slice goes through the rich domain (enforced writes — [02-DomainPersistence.md](02-DomainPersistence.md)) — without a shared abstraction forcing compromise.
- New developers find everything for a feature **in one folder**, not by tracing calls across layers.

This aligns code structure with how change actually happens (per feature), which is the recurring argument for slices.

---

## The duplication trade-off (DRY vs decoupling)

The most-cited objection: slices can **duplicate** code (two slices each write similar mapping or query logic). Vertical-slice thinking accepts *some* duplication as **better than the wrong abstraction**:

- **Coupling via a shared abstraction** (a fat shared service used by many features) is often worse than a little duplication — a change to the shared code risks every feature using it.
- A small amount of repeated, simple code in independent slices keeps them **decoupled** and individually changeable.
- **Genuinely shared, stable logic** (domain entities, value objects, true cross-cutting utilities) *should* still be shared — slices share the **domain** ([02-DomainPersistence.md](02-DomainPersistence.md)) and cross-cutting infrastructure; they just don't force-share *application/feature* logic.

So it's not "never DRY" — it's "don't create a shared abstraction *prematurely* to avoid trivial duplication, when that coupling costs more than the duplication." Extract shared code when it's **truly** shared and stable, not reflexively.

---

## What's still shared

Vertical slices don't mean zero structure — some things remain shared/cross-cutting:

- **Domain model** — entities, aggregates, value objects, domain events ([02-DomainPersistence.md](02-DomainPersistence.md), [05-DomainEvents.md](05-DomainEvents.md)) are shared; slices orchestrate the domain, they don't each redefine it.
- **Infrastructure** — DbContext, the host, DI wiring ([Ch03](../03-HostingAndDI/README.md)), cross-cutting pipeline behaviors (validation/logging — [04-CQRS.md](04-CQRS.md)).
- **Cross-cutting concerns** — auth ([Ch10](../10-Identity/README.md)), observability ([Ch12](../12-Observability/README.md)), resilience ([Ch11](../11-Resilience/README.md)).

Slices are about organizing **feature/application logic** by feature; the **domain and infrastructure** beneath remain shared. (This also composes with a **modular monolith** — [01-SolutionLayout.md](01-SolutionLayout.md) — where each module is a set of slices over a shared domain.)

---

## Common gotchas

### Forcing a shared abstraction to avoid trivial duplication

Creating a fat shared service to DRY up two slices re-introduces the coupling slices avoid. Prefer a little duplication over a premature shared abstraction; extract only **truly** shared, stable logic.

### Duplicating the domain across slices

Slices share the **domain** (entities/aggregates/value objects) — don't redefine domain logic per slice. Duplication tolerance applies to *application/feature* glue, not the core domain ([02-DomainPersistence.md](02-DomainPersistence.md)).

### Slices that aren't actually independent

If slices reach into each other or share fat services, you lose the decoupling benefit. Keep slices self-contained; communicate across features via the domain/events, not shared mutable services.

### Treating it as dogma

Vertical slices and layers aren't mutually exclusive or universally "best." Use slices where change-locality matters (feature-rich apps); a simple app may not need either elaborately. Match structure to the system.

### No cross-cutting consistency

Without shared pipeline behaviors/conventions, each slice reinvents validation/logging/error handling inconsistently. Use shared **cross-cutting** infrastructure (e.g., pipeline behaviors — [04-CQRS.md](04-CQRS.md)) even with independent slices.

---

## Summary

- **Vertical slice architecture** organizes code **by feature** (a self-contained slice per use case: endpoint + command/query + handler + validation) rather than **by layer** — because software changes **per feature**, so code that changes together lives together.
- It gives **high cohesion within a feature** and **low coupling between features**: changing one feature is localized to its slice, and each slice can be optimized for its needs (queries project to DTOs, commands go through the rich domain) without a shared abstraction forcing compromise.
- It accepts **some duplication** as better than the **wrong (premature) shared abstraction** — coupling via a fat shared service often costs more than a little repeated glue; still share **truly** stable logic (the **domain**, infrastructure, cross-cutting concerns).
- It pairs naturally with **CQRS** ([04-CQRS.md](04-CQRS.md)), **Minimal APIs**, and **modular monoliths** ([01-SolutionLayout.md](01-SolutionLayout.md)); it's an alternative to layered architecture, not dogma — match the structure to the system.

→ Next: [07-MultiTenancy.md](07-MultiTenancy.md)
