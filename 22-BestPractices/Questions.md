# Chapter 22 — Best Practices & Patterns — Q & A

---

### Q1. What are the universal solution-layout conventions?

A **`src/`–`tests/`–`infra/`** split and **`Directory.Build.props`** for shared MSBuild settings. Beyond that, structure matches the system's shape (monolith / modular monolith / microservices).

---

### Q2. What's the dependency rule in layered (Clean/Onion) architecture?

**Dependencies point inward** toward the domain: Domain (no dependencies) ← Application ← Infrastructure, with the Api/Host as the composition root. Infrastructure **implements** interfaces the inner layers declare (Dependency Inversion), so the domain is infrastructure-agnostic and testable.

---

### Q3. Monolith vs modular monolith vs microservices — when each?

**Monolith**: one deployable, simplest — the right default/start for most systems. **Modular monolith**: one deployable with **strong module boundaries** (microservice-like structure without distributed complexity; easy to split later). **Microservices**: many deployables — only when independent scaling/team autonomy justifies the distributed-systems cost. Don't go microservices too early.

---

### Q4. Anemic vs rich domain model?

**Anemic**: entities are data bags (public setters), logic in external services — rules scattered, invariants unprotected. **Rich**: entities own behavior and protect invariants (state via methods, encapsulated collections, no public setters). Use rich for complex domains; anemic DTOs are fine for CRUD.

---

### Q5. How do you keep persistence out of the domain?

Domain declares interfaces (`IOrderRepository`), infrastructure implements them (the domain never references EF Core); **map** a rich model via EF Core configuration (no public setters needed); use **value objects** (owned types) for identity-less concepts. The test: can you unit-test the domain with no database?

---

### Q6. What's an aggregate and why does it matter?

A cluster of entities/value objects with one **root** as the entry point, defining a **transactional consistency boundary** — external code goes through the root (invariants enforced), you reference other aggregates **by id**, and save one aggregate atomically.

---

### Q7. Why is a generic `Repository<T>` over EF Core usually an anti-pattern?

`DbContext` is already a Unit of Work and `DbSet<T>` a repository. A generic wrapper duplicates that, **leaks `IQueryable`** (no real abstraction), or **hides EF Core's power** (Include/projections/AsNoTracking). Use a **specific, domain-oriented** repository per aggregate only when it earns its keep, or use EF Core directly.

---

### Q8. When IS a repository worth it?

As a **domain-meaningful interface per aggregate** (`IOrderRepository` with intention-revealing methods) — to decouple the domain from EF Core (Clean architecture), encapsulate complex/repeated queries, and aid testing. Not as a reflexive generic wrapper.

---

### Q9. What is CQRS and its spectrum?

Separating **commands** (change state, go through the rich domain) from **queries** (read, project to view DTOs, bypass the domain). Spectrum: **lightweight** (separate handlers, one DB — recommended default) → separate **read models** → full **CQRS + event sourcing** (separate stores, eventual consistency — complex). Overkill for simple CRUD.

---

### Q10. What does a mediator (MediatR-style) add to CQRS?

Thin controllers (send request → handler routes it), one handler per use case, and **pipeline behaviors** (cross-cutting validation/logging/transactions wrapped around every handler). The pattern matters more than the (now-licensed) library.

---

### Q11. Domain events vs integration events?

**Domain events**: in-process, within one service, often same transaction — coordinate side effects in *this* app. **Integration events**: cross-service, published via a broker, eventual consistency. A domain event may *cause* an integration event (via outbox), but they differ in scope/delivery/consistency.

---

### Q12. What problem does the outbox pattern solve?

The **dual-write** problem: committing the DB then publishing to a broker can lose the message if the publish fails (DB and downstream diverge). The outbox writes the event to a table **in the same transaction** as the business change, and a separate process publishes it — guaranteeing publish iff committed. Consumers must be idempotent.

---

### Q13. What is vertical slice architecture and its main benefit?

Organizing code **by feature** (a self-contained slice per use case: endpoint + command/query + handler + validation) instead of by layer. Benefit: **high cohesion within a feature, low coupling between features** — changing one feature is localized; each slice optimizes for its needs.

---

### Q14. What's the duplication trade-off in vertical slices?

Slices accept **some duplication** as better than the **wrong (premature) shared abstraction** — coupling via a fat shared service often costs more than a little repeated glue. Still share **truly** stable logic (the domain, infrastructure, cross-cutting concerns).

---

### Q15. Compare the three multi-tenancy isolation models.

**Per-row** (shared tables + `TenantId` — lowest cost/density, weakest isolation), **per-schema** (middle ground), **per-database** (strongest/physical isolation, highest cost — for few large/regulated tenants). Often mixed.

---

### Q16. What's the cardinal multi-tenancy risk and how do you defend it?

**Cross-tenant data leakage** (one forgotten `TenantId` filter in per-row). Defend with **automatic EF Core global query filters** (not hand-filtering), **trusted** tenant resolution (from the authenticated identity, not unverified client input), stronger isolation models, and explicit tests. Treat isolation as a security boundary; include tenant in cache keys and propagate tenant context to background work.

---

### Q17. What's the core value of feature flags?

**Decoupling deployment from release**: ship code with a feature off, deploy safely, enable later (all / a %/ groups), roll back instantly by flipping the switch. Enables progressive rollout, A/B testing, kill-switches, and trunk-based development.

---

### Q18. Where must feature flags live for the full benefit?

In a **runtime-changeable** source — **Azure App Configuration** (or a flag service) — so changes propagate to running apps without redeploy. Flags in `appsettings.json` only change on redeploy, losing the "release without deploy" benefit.

---

### Q19. Explain SemVer.

`MAJOR.MINOR.PATCH`: **MAJOR** = breaking change (upgrade deliberately), **MINOR** = backward-compatible new features, **PATCH** = backward-compatible bug fixes. It communicates upgrade safety; a breaking change requires a MAJOR bump — doing it in a minor/patch violates the contract.

---

### Q20. How do you version an HTTP API and why?

Serve **multiple versions simultaneously** (via `Asp.Versioning`) using URL path/query/header/media type, so clients migrate on their own schedule — introduce breaking changes in v2 without breaking v1. Deprecate (sunset headers) with notice before retiring; integrate with OpenAPI for per-version docs.

---

### Q21. Name five anti-patterns and their replacements.

**Service Locator** → constructor injection; **anemic domain** → rich model (for complex domains); **sync-over-async** → async all the way; **fat controllers** → thin controllers + handlers (CQRS); **magic strings/primitive obsession** → constants/`nameof`/value objects. (Also: generic repo over EF Core, swallowed exceptions, `async void`, captive dependencies, N+1, secrets in source, premature microservices/optimization.)

---

### Q22. Why default types to `internal`?

A project's **public** surface is its contract — the smaller it is, the more freely you can refactor internals without breaking consumers. Make types `internal` unless they need to be `public`; use `[InternalsVisibleTo]` for tests and the `file` modifier for single-file helpers.

---

### Q23. Why and how should you organize code by feature within a project?

Because software changes **per feature**, so co-locating a feature's files (endpoint, handler, validator) makes change localized and navigation easy (vs scattered by-type folders). Match namespaces to folders; reserve by-type for small projects or cross-cutting buckets.

---

### Q24. When use `ConfigureAwait(false)`?

In **library** code that doesn't need the caller's synchronization context — it avoids capturing the context, preventing deadlocks/overhead in consumers that *have* a context (UI/legacy ASP.NET). It's a no-op in ASP.NET Core but still correct for portable libraries. **Don't** use it in UI/app code that needs to resume on its context.

---

### Q25. Why propagate `CancellationToken` everywhere?

So a cancelled request or shutdown actually **stops wasted work** (an abandoned HTTP call/query keeps running otherwise — wasted threads/connections/DB at scale). Accept a token parameter and pass it down the whole async chain; use `HttpContext.RequestAborted` / the host shutdown token, and link tokens for timeouts.
