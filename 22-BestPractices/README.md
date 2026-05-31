# Chapter 22 — Best Practices & Patterns

> Architectural patterns that hold up at scale. Project structure, repository/UoW, CQRS, domain events, vertical slices, anti-patterns to retire.

**Prerequisites**: most of the book.

**Time to read**: ~6-8 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-SolutionLayout.md](01-SolutionLayout.md) | Project structure for monoliths, modular monoliths, microservices. `src/`, `tests/`, `infra/`. |
| [02-DomainPersistence.md](02-DomainPersistence.md) | Separating domain from persistence; rich domain models vs anemic. |
| [03-RepositoryUnitOfWork.md](03-RepositoryUnitOfWork.md) | When to use, when EF Core's DbContext already provides it. |
| [04-CQRS.md](04-CQRS.md) | Command/query separation, MediatR, when CQRS overkill. |
| [05-DomainEvents.md](05-DomainEvents.md) | Raising and handling events within an aggregate; outbox pattern. |
| [06-VerticalSlices.md](06-VerticalSlices.md) | Vertical slice architecture as an alternative to layers. |
| [07-MultiTenancy.md](07-MultiTenancy.md) | Tenant per row, per schema, per database. |
| [08-FeatureFlags.md](08-FeatureFlags.md) | Toggling features in production; FeatureManagement library. |
| [09-Versioning.md](09-Versioning.md) | API versioning (`Asp.Versioning`), semantic versioning for libraries. |
| [10-AntiPatterns.md](10-AntiPatterns.md) | Service locator, anemic domain, sync-over-async, fat controllers, magic strings. |
| [11-CodeOrganization.md](11-CodeOrganization.md) | Folder structure within a project, namespace conventions, internal types. |
| [12-AsyncIdiomsAtScale.md](12-AsyncIdiomsAtScale.md) | ConfigureAwait in libraries, CancellationToken propagation, deadlock prevention. |
| [Questions.md](Questions.md) | Drilling. |
| [Coding.md](Coding.md) | Refactor an anemic model; introduce CQRS to a CRUD app; extract a vertical slice. |

→ Begin: [01-SolutionLayout.md](01-SolutionLayout.md)
