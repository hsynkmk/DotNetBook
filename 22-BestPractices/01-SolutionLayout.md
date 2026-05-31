# Solution Layout

## Structure that scales with the system

How you organize a solution — its projects, folders, and boundaries — shapes how the codebase grows, how teams work in parallel, and how easily you can evolve or split it later. There's no single "correct" layout; the right structure depends on the system's **shape**: a single deployable **monolith**, a **modular monolith** (one deployable, strong internal boundaries), or **microservices** (many deployables). The recurring principles across all of them: a clear **`src/`–`tests/`** split, **dependencies point inward** (toward the domain, not outward to infrastructure), and **boundaries match the architecture** you intend.

```
MyApp/
├── src/
│   ├── MyApp.Domain/           ← entities, value objects, domain logic (no dependencies)
│   ├── MyApp.Application/      ← use cases, interfaces (depends on Domain)
│   ├── MyApp.Infrastructure/   ← EF Core, HTTP clients, impls (depends on Application)
│   └── MyApp.Api/              ← ASP.NET Core host (composition root)
├── tests/
│   ├── MyApp.UnitTests/
│   └── MyApp.IntegrationTests/
├── infra/                      ← Dockerfile, Helm charts, IaC ([Ch19](../19-Deployment/README.md))
├── Directory.Build.props       ← shared MSBuild settings
└── MyApp.sln
```

---

## The `src/` and `tests/` convention

A near-universal convention: separate **`src/`** (production code) from **`tests/`** (test projects), with **`infra/`** (or `deploy/`) for Dockerfiles, Helm charts, and IaC ([Ch19](../19-Deployment/README.md)). Benefits:

- **Clear separation** of what ships vs what verifies.
- **Tooling-friendly** — CI can build `src/`, run `tests/`, package from `infra/`.
- **`Directory.Build.props`** at the root applies shared MSBuild settings (target framework, nullable, analyzers, warnings-as-errors) to all projects, avoiding per-csproj duplication ([Ch19 §09](../19-Deployment/09-CICD.md)).

This is the baseline regardless of architecture.

---

## Layered (Clean/Onion) architecture

A common layout for monoliths/services separates concerns into projects where **dependencies point inward** toward the domain (the "dependency rule" of Clean/Onion architecture):

```
Api  →  Application  →  Domain  ←  Infrastructure
                                    (implements Application's interfaces)
```

- **Domain** — entities, value objects, domain logic, and the *interfaces* it needs (e.g., `IOrderRepository`). **Depends on nothing.** ([02-DomainPersistence.md](02-DomainPersistence.md))
- **Application** — use cases/orchestration (often CQRS handlers — [04-CQRS.md](04-CQRS.md)); depends on Domain.
- **Infrastructure** — concrete implementations: EF Core repositories ([Ch05](../05-EFCore/README.md)), HTTP clients ([Ch09](../09-NetworkingAndHttp/README.md)), message brokers ([Ch07](../07-Messaging/README.md)). Depends inward and **implements** the interfaces the inner layers declare (Dependency Inversion — [CSharpBook Ch17 SOLID](../../CSharpBook/17-BestPractices/README.md)).
- **Api/Host** — the **composition root** where DI wires interfaces to implementations ([Ch03 §02](../03-HostingAndDI/02-DependencyInjection.md)).

The key property: the **domain doesn't depend on infrastructure** — you can swap the database or web framework without touching domain logic, and the domain is testable in isolation. (An alternative, vertical slices — [06-VerticalSlices.md](06-VerticalSlices.md) — organizes by feature instead of layer; both are valid.)

---

## Monolith vs modular monolith vs microservices

| | Monolith | Modular monolith | Microservices |
|---|---|---|---|
| Deployables | one | **one** | many |
| Boundaries | loose (or layered) | **strong internal modules** | network boundaries |
| Complexity | low | medium | **high** (distributed) |
| Best for | most apps, starting out | growing apps needing boundaries | independent scaling/teams at scale |

- **Monolith** — one deployable; simplest to build, test, and deploy. The right default for most systems and almost always the right *start*.
- **Modular monolith** — still **one deployable**, but organized into well-bounded **modules** (e.g., `Orders`, `Billing`, `Catalog`), each with its own internal layers and a clear public surface, communicating through defined contracts (not reaching into each other's internals). It gives microservice-like boundaries **without** distributed-systems complexity — and makes a *future* split into services far easier if needed.
- **Microservices** — many independently-deployable services ([Ch18 Aspire](../18-Aspire/README.md) helps compose them). Powerful for independent scaling/team autonomy, but you take on **distributed-systems complexity** (network, consistency, observability, deployment — [Ch11](../11-Resilience/README.md), [Ch12](../12-Observability/README.md)) that's often premature.

The pragmatic guidance: **start with a (modular) monolith**; extract microservices only when a concrete need (independent scaling, team boundaries, separate release cadence) justifies the complexity. "Microservices first" is a common, costly mistake.

---

## Module boundaries in a modular monolith

In a modular monolith, each **module** is a vertical unit (its own domain/application/infrastructure or a vertical slice — [06-VerticalSlices.md](06-VerticalSlices.md)) with:

- A **public contract** (interfaces/DTOs/events) other modules use — internals are `internal` ([11-CodeOrganization.md](11-CodeOrganization.md)).
- **No reaching into another module's internals** — modules talk through their public surface or via **domain/integration events** ([05-DomainEvents.md](05-DomainEvents.md)), not by directly querying each other's tables/types.
- Often a **module per folder/project** so the boundary is enforced by project references (a module can't reference another's internals if it's not referenced).

Enforcing these boundaries keeps the monolith from degrading into a "big ball of mud" and preserves the option to extract a module into a service later.

---

## Common gotchas

### Microservices too early

Adopting microservices before you need them imposes distributed-systems complexity (network failures, consistency, deployment, observability) with little benefit. **Start with a monolith** (modular if it's growing); split only when justified.

### Domain depending on infrastructure

If the domain references EF Core/HTTP/framework types, you can't test it in isolation or swap infrastructure. Keep **dependencies pointing inward** — infrastructure implements interfaces the domain/application declare ([02-DomainPersistence.md](02-DomainPersistence.md)).

### No enforced module boundaries

A "modular monolith" where modules freely reference each other's internals is just a monolith with extra folders. Enforce boundaries (public contracts, `internal` types, project references) or the structure erodes.

### Anemic project structure for the system's complexity

Over-layering a tiny app (four projects for a CRUD utility) adds ceremony; under-structuring a large one becomes unmaintainable. Match the layout's complexity to the system's.

### Scattering infra/build config

Duplicated build settings across csproj files drift. Centralize with **`Directory.Build.props`** and central package management ([Ch19 §09](../19-Deployment/09-CICD.md)).

---

## Summary

- Solution layout should match the system's **shape**; universal baselines are a **`src/`–`tests/`–`infra/`** split and **`Directory.Build.props`** for shared settings.
- **Layered (Clean/Onion)** architecture separates **Domain** (no dependencies), **Application** (use cases), **Infrastructure** (implementations), and **Api/Host** (composition root) — with **dependencies pointing inward** so the domain is infrastructure-agnostic and testable ([02-DomainPersistence.md](02-DomainPersistence.md)); vertical slices ([06](06-VerticalSlices.md)) are an alternative organization.
- Choose **monolith** (simplest, the right default/start), **modular monolith** (one deployable, **strong module boundaries** — microservice-like structure without distributed complexity, easy to split later), or **microservices** (many deployables — only when independent scaling/team autonomy justifies the **distributed-systems cost**).
- **Don't go microservices too early**; enforce **module boundaries** (public contracts, `internal` types, project references) so a modular monolith doesn't degrade into a big ball of mud.

→ Next: [02-DomainPersistence.md](02-DomainPersistence.md)
