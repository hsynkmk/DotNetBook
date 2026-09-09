# 22 — Best Practices, Patterns & Architecture

## ⚡ 30-second answer

Structure follows the system's shape. Baselines: **`src/`–`tests/`** plus
`Directory.Build.props`, and a **modular monolith** as the sane default — one deployable with
real module boundaries, easy to split later, without paying distributed-systems cost on day one.
Inside it, keep the **domain separate from persistence**: entities own their behavior and protect
invariants, the domain declares interfaces and infrastructure implements them, and the test is
*can you unit-test the domain with no database?* Model **aggregates** with a single root as the
consistency boundary; reference other aggregates **by id** and save one per transaction. A
generic `Repository<T>` over EF Core is usually over-engineering — `DbContext` is already a Unit
of Work. **CQRS** is a spectrum, useful once read and write models genuinely diverge. Cross-service
consistency uses the **outbox**, never a distributed transaction. And the anti-patterns that
actually get asked: service locator, anemic domain, sync-over-async, fat controllers, captive
dependencies, N+1.

---

## Core mechanics

### Solution layout

```text
src/
  Company.Domain/          ← entities, value objects, domain events. NO dependencies.
  Company.Application/     ← use cases, ports (interfaces), DTOs
  Company.Infrastructure/  ← EF Core, HTTP clients, brokers — implements the ports
  Company.Api/             ← composition root: DI wiring, endpoints
tests/
Directory.Build.props      ← nullable, warnings-as-errors, analyzers, shared TFM
```

**Dependencies point inward.** The domain references nothing; infrastructure references the
domain, never the reverse. The composition root is the only place that knows about everything.

### Monolith → modular monolith → microservices

| | Deployables | Boundaries | Cost |
|---|---|---|---|
| **Monolith** | 1 | none enforced | lowest — the right *start* |
| **Modular monolith** | 1 | **enforced module contracts** | low; splits cleanly later |
| **Microservices** | many | network | **distributed systems** — only when justified |

Microservices buy independent scaling and team autonomy, and cost you network failure modes,
eventual consistency, distributed tracing, and deployment complexity. Choose them when those
first two are worth the second three — not because the architecture diagram looks better.

Enforce module boundaries with `internal` types, deliberate public contracts, and project
references (or an architecture test with NetArchTest) — otherwise a modular monolith degrades
into a big ball of mud within a year.

### Rich domain vs anemic model

```csharp
// ❌ anemic — a data bag; invariants live in whichever service remembers them
public class Order { public List<Line> Lines { get; set; } public OrderStatus Status { get; set; } }

// ✅ rich — behavior and invariants live with the data
public class Order
{
    private readonly List<Line> _lines = [];
    public IReadOnlyList<Line> Lines => _lines;             // encapsulated collection
    public OrderStatus Status { get; private set; }         // no public setter

    public void AddLine(Product p, int qty)
    {
        if (Status != OrderStatus.Draft) throw new DomainException("Order is not editable.");
        _lines.Add(new Line(p.Id, qty, p.Price));
    }
}
```

Match richness to actual business complexity — **CRUD can stay anemic**, and pretending otherwise
is its own anti-pattern. EF Core maps a rich model fine (backing fields, private setters), so
"the ORM needs public setters" is not a reason.

### Aggregates

An **aggregate** is a consistency boundary accessed through its **root**, which guards the
invariants. Four rules:

1. Protect true invariants together — that's what defines the boundary.
2. Keep aggregates **small**.
3. Reference other aggregates **by id**, not by navigation.
4. **One aggregate per transaction**; update others eventually (domain/integration events).

### Repository & Unit of Work

`DbContext` **is** a Unit of Work; `DbSet<T>` **is** a repository. So:

- ❌ A generic `Repository<T>` over EF Core: it duplicates `DbSet`, and either **leaks
  `IQueryable`** (no real abstraction — the caller can still write any query) or **hides EF's
  power** (`Include`, projections, `AsNoTracking`, `ExecuteUpdate`).
- ✅ A **specific, domain-oriented repository per aggregate root** — `IOrderRepository` with
  intention-revealing methods — to decouple the domain from EF and encapsulate complex queries.
- For simple CRUD, use `DbContext` directly. For cross-service atomicity use the **outbox**, not a
  UoW wrapper ([05](05-EFCore.md)).

### CQRS

Separate **commands** (change state, go through the rich domain, enforce rules) from **queries**
(read, project straight to view-shaped DTOs, bypass the domain entirely).

```csharp
// query: no domain model, no tracking — just the shape the screen needs
public sealed record GetOrderSummary(Guid Id) : IRequest<OrderSummaryDto>;
```

A spectrum, not a switch:

| Level | Shape | When |
|---|---|---|
| **Lightweight** | separate handlers, one database | the recommended default when CQRS fits |
| **Separate read model** | projected denormalized read store | reads diverge from writes, or scale separately |
| **CQRS + event sourcing** | separate stores, projections, eventual consistency | high scale, or audit is a hard requirement |

The mediator (MediatR-style) gives thin controllers, one handler per use case, and **pipeline
behaviors** for validation/logging/transactions. The *pattern* matters more than the library —
which now has a commercial license.

**CQRS is overkill for CRUD** where the read and write models are the same shape.

### Domain events and the outbox

The aggregate **records** events; **infrastructure dispatches them after persistence** (a
`SaveChanges` interceptor collecting events off tracked entities). The aggregate never calls its
own side effects.

- **Domain event** — in-process, this service, usually the same transaction.
- **Integration event** — crosses a service boundary via a broker, eventually consistent.

**The dual-write problem**: you can't atomically write the database *and* publish to a broker. The
**outbox pattern** solves it — write the message to an outbox table **in the same transaction** as
the business change, then a separate process publishes it. The message is published **iff** the
transaction committed. Handlers must be **idempotent**, because delivery is at-least-once
([07](07-Messaging.md)).

### Vertical slices

Organize **by feature**, not by layer — because software changes per feature, so code that changes
together should live together.

```text
Features/Orders/PlaceOrder/{ Endpoint.cs, Command.cs, Handler.cs, Validator.cs }
```

High cohesion within a feature, low coupling between features, and each slice optimized for its
own needs. It **accepts some duplication** rather than a premature shared abstraction — a fat
shared service usually costs more than a little repeated glue. Still share what's *truly* stable:
the domain, infrastructure, cross-cutting concerns.

### Multi-tenancy

The isolation model is the core decision:

| Model | Isolation | Density/cost | For |
|---|---|---|---|
| **Per-row** (`TenantId`) | weakest | best | many small tenants |
| **Per-schema** | middle | middle | mixed |
| **Per-database** | **physical** | worst | few large or regulated tenants |

Resolve the tenant **early** per request (subdomain/path/header/**claim**) into a scoped
`ITenantContext`, and **verify it against the authenticated identity** — never trust an
unverified client value. The defining risk is **cross-tenant leakage**; defend with **EF global
query filters** so the filter can't be forgotten. Include the tenant in **cache keys**, propagate
it to **background work and messages**, and tag it in **logs and traces**.

### Versioning

**SemVer** for libraries: MAJOR = breaking, MINOR = compatible feature, PATCH = compatible fix.
**`Asp.Versioning`** for APIs, serving multiple versions simultaneously so clients migrate on
their own schedule. Manage change by **adding rather than changing**, **deprecating before
removing** (`[Obsolete]`, sunset headers, notice), and writing **tolerant readers** that ignore
unknown fields.

### Async at scale

**Never block on async** (`.Result`, `.Wait()`) — thread-pool starvation under load, deadlock with
a sync context ([01](01-Runtime-GC-JIT.md)). **`ConfigureAwait(false)` in library code** (a no-op
in ASP.NET Core, but correct for portability). **Propagate `CancellationToken`** through every
async call. `Task.WhenAll` for concurrency, `ValueTask` for often-synchronous hot paths,
`IAsyncEnumerable` for streaming, and **`async Task`, never `async void`**.

---

## The anti-pattern list (know these cold)

| Anti-pattern | Why it hurts | Replace with |
|---|---|---|
| **Service locator** | hides dependencies; fails at runtime, not startup | constructor injection ([03](03-Hosting-DI-Config.md)) |
| **Anemic domain model** | invariants unenforceable, logic scattered | rich model — *when complexity warrants* |
| **Sync-over-async** | thread-pool starvation, deadlocks | async all the way |
| **`async void`** | exceptions can't be caught; nothing to await | `async Task` |
| **Fat controllers** | untestable, mixes concerns | thin endpoints + handlers |
| **Generic `Repository<T>` over EF** | duplicates `DbSet`, leaks or cripples | `DbContext`, or a per-aggregate repository |
| **Captive dependency** | scoped service pinned in a singleton | scope per unit of work |
| **N+1 queries** | a query per row | `Include` or projection ([05](05-EFCore.md)) |
| **Swallowed exceptions** | failures become silent corruption | handle at the boundary, log with the exception |
| **Exceptions as control flow** | slow, obscures intent | `Try*` pattern, result types |
| **Magic strings / primitive obsession** | typos compile; no invariants | constants, `nameof`, value objects |
| **Secrets in source** | leaked permanently via git history | user secrets, Key Vault, managed identity |
| **Premature microservices** | distributed cost, no benefit yet | modular monolith |
| **Premature optimization** | complexity for unmeasured gain | measure first ([21](21-Performance-Tooling.md)) |

The shared roots: **hidden coupling**, **broken invariants**, and **convenient-but-unscalable**
choices. The shared cure: explicit dependencies, encapsulation, async correctness, single
responsibility, measure first.

---

## 🪤 Traps & gotchas

- **Reaching for microservices first.** The distributed-systems tax — network partitions,
  eventual consistency, distributed tracing, N deployment pipelines — is real and immediate; the
  benefits are speculative until you actually need independent scaling or team autonomy.
- **A "modular monolith" with no enforced boundaries** is just a monolith with optimistic naming.
  Without `internal`, contracts, and an architecture test, modules grow references into each
  other's internals within months.
- **Clean Architecture on a CRUD app** — four projects, a mapper, and an interface per class to
  save a row. Over-engineering is an anti-pattern with better PR.
- **A rich domain model that still can't be unit-tested** — usually because EF types, `DbContext`,
  or `HttpContext` leaked into the domain project. Check the project's references.
- **Aggregates that are too big** — "the Customer aggregate contains all their orders" means every
  order edit loads and locks the whole customer. Reference by id.
- **Two aggregates in one transaction** — you've just made a distributed-consistency problem
  local, and the boundary you drew was wrong.
- **Generic repository leaking `IQueryable`** — `IRepository<T>.Query()` returning `IQueryable<T>`
  abstracts nothing: callers still write EF-specific expressions, and now you can't see them.
- **Dual write** — saving to the database and then publishing to the broker. The process dies in
  between and the two are permanently inconsistent. Outbox.
- **Non-idempotent message handlers** — delivery is at-least-once, so "increment the balance"
  will eventually run twice.
- **Dispatching domain events inside the aggregate** couples the domain to infrastructure and
  fires side effects for a transaction that may still roll back. Record, then dispatch after save.
- **Cross-tenant leak from one missing `Where`** — the single highest-severity bug in a per-row
  multi-tenant app. Global query filters exist precisely so this can't be forgotten.
- **`IgnoreQueryFilters()` in multi-tenant code** removes the tenant filter along with soft delete.
- **Trusting a tenant id from a header or route** without checking it against the authenticated
  principal — trivial horizontal privilege escalation.
- **Forgetting the tenant in cache keys** — one tenant's data served to another out of
  `IMemoryCache`.
- **Losing tenant/user context when work moves to the background** — the queue item has no
  `HttpContext`; capture what you need at enqueue time ([08](08-BackgroundProcessing.md)).
- **Feature flags that never get removed** — flag explosion, and 2^n untested code paths. Removal
  is part of the feature, not a follow-up ticket.
- **Feature flags in `appsettings.json`** only change on deploy — which forfeits the entire point
  of decoupling release from deployment.
- **Breaking a public API without a MAJOR bump** — including subtler cases: adding a member to an
  interface, changing a return type, tightening a parameter.
- **Removing an API version without a deprecation window** — sunset headers and notice, then
  removal.
- **`ConfigureAwait(false)` cargo-culted into application code** — it's a no-op in ASP.NET Core,
  and in a UI app it's precisely wrong when you need to resume on the UI thread.
- **Not propagating `CancellationToken`** — the client disconnected 30 seconds ago and you're
  still running their report.

---

## ❓ Likely questions

**Q: Monolith, modular monolith, or microservices — how do you choose?**
A: Start with a monolith; make it modular as it grows. Go to microservices only when you need
independent scaling or independent team deployment badly enough to accept network failure modes,
eventual consistency and N pipelines. A modular monolith with real boundaries gives you most of
the structural benefit and splits cleanly when the need is genuine.

**Q: What is the Dependency Rule?**
A: Source dependencies point inward. The domain depends on nothing; the application layer depends
on the domain; infrastructure depends on both. You invert infrastructure by declaring interfaces
(ports) in the inner layers and implementing them outside, wired in the composition root.

**Q: What's an anemic domain model and is it always wrong?**
A: Entities that are data bags with all behavior in services, so invariants can be bypassed and
logic scatters. It's wrong for genuinely complex domains. For CRUD, it's fine — matching richness
to complexity is the actual skill.

**Q: What is an aggregate and what are the rules?**
A: A consistency boundary accessed through a root that guards its invariants. Keep it small,
protect true invariants together, reference other aggregates by id, and update one aggregate per
transaction — others eventually, via events.

**Q: Should you put a repository over EF Core?**
A: Usually not a generic one — `DbContext` is already a Unit of Work and `DbSet<T>` a repository,
so `Repository<T>` either leaks `IQueryable` or hides `Include`/projections/`AsNoTracking`. A
specific `IOrderRepository` per aggregate is worth it to keep EF out of the domain and to
encapsulate complex queries.

**Q: What is CQRS and when do you use it?**
A: Separating the read model from the write model because they have opposite needs — writes need
invariants and a rich domain, reads need denormalized view-shaped data. Adopt it when the models
genuinely diverge, when you want uniform cross-cutting handling via pipeline behaviors, or when
reads need to scale independently. It's overkill for CRUD.

**Q: Domain event vs integration event?**
A: A domain event is in-process within one service, usually in the same transaction. An
integration event crosses a service boundary via a broker and is eventually consistent. A domain
event often *causes* an integration event.

**Q: What is the dual-write problem, and how does the outbox fix it?**
A: You can't atomically commit a database transaction and publish to a broker — a crash between
them leaves them inconsistent. The outbox writes the message into an outbox table in the *same*
transaction, and a separate publisher reads and sends it. The message goes out if and only if the
business change committed.

**Q: Why must message handlers be idempotent?**
A: Because brokers deliver at-least-once. Retries, redelivery after a failed ack, and publisher
retries all mean the same message can arrive twice. Deduplicate on a message id, or make the
operation naturally idempotent.

**Q: What is vertical slice architecture and what's the trade-off?**
A: Organizing by feature rather than by layer, so everything for one use case lives together.
High cohesion per feature, low coupling between features, and each slice optimized for its own
needs. The trade-off is accepting some duplication instead of a premature shared abstraction.

**Q: What are the multi-tenancy isolation models?**
A: Per-row (shared tables with `TenantId`) — cheapest and densest, weakest isolation; per-schema —
a middle ground; per-database — physical isolation, highest cost, for few large or regulated
tenants. Real systems often mix, putting big customers on their own database.

**Q: How do you prevent cross-tenant data leakage?**
A: Treat it as a security boundary. Resolve the tenant from a *verified* source (a claim, not a
header), put it in a scoped context, and apply EF **global query filters** so the filter is
automatic rather than remembered. Then include the tenant in cache keys and propagate it to
background work — and test it explicitly.

**Q: Why does a feature flag need a runtime-changeable source?**
A: The value of a flag is decoupling release from deployment — ship dark, enable later, roll back
instantly. A flag in `appsettings.json` only changes on deploy, so you've kept all the complexity
and lost the benefit. Azure App Configuration or similar gives dynamic flags.

**Q: What counts as a breaking change?**
A: Anything that makes a working consumer stop working — removing or renaming a public member,
changing a return type or parameter, tightening validation, changing a default, adding a member to
an interface others implement. It requires a MAJOR bump, or a new API version.

**Q: Why is sync-over-async so bad at scale?**
A: `.Result` blocks a thread-pool thread while the async operation completes. The pool injects
replacements slowly, so under load throughput collapses and latency spikes while CPU sits idle —
and with a synchronization context it deadlocks outright.

**Q: Name the anti-patterns you'd flag in a .NET code review.**
A: Service locator, anemic domain where the domain is complex, sync-over-async, `async void`, fat
controllers, generic repository over EF, captive dependencies, N+1 queries, swallowed exceptions,
magic strings, secrets in source, and premature microservices or optimization.

---

## 🎓 Senior Extra

- **Architecture is the set of decisions that are hard to reverse.** The litmus test: if getting
  it wrong means a migration rather than a refactor, it's architecture — which is exactly why
  service boundaries and data ownership deserve more care than folder structure.
- **Conway's Law is a design constraint, not a joke.** Service boundaries that cut across team
  boundaries generate permanent coordination cost; the modular monolith is often the honest
  response to a team that isn't shaped like its diagram yet.
- **Write architecture tests.** NetArchTest or a similar library turns "the domain must not
  reference EF" from a convention into a failing build — the only version of a rule that survives
  team growth.
- **`Directory.Build.props` is where consistency actually lives**: nullable enabled,
  warnings-as-errors, analyzers, the TFM. Combined with `.editorconfig` and analyzer severity,
  conventions stop depending on reviewer memory.
- **Types `internal` by default** keeps your refactoring freedom; `public` is a commitment you
  version. `[InternalsVisibleTo]` for tests, the `file` modifier for single-file helpers.
- **The outbox has a sibling — the inbox** (dedupe table on the consumer side), and together they
  give you effectively-once semantics on top of at-least-once transport.
- **Eventual consistency is a product decision, not just a technical one.** "The order page may
  show a stale total for two seconds" needs a product owner's agreement, and the UI usually needs
  to acknowledge it.
- **Pipeline behaviors are where CQRS earns its keep** — validation, transactions, logging,
  retries, and idempotency applied uniformly to every handler, rather than remembered per
  endpoint.
- **Prefer expand/contract for every breaking change**, in the database ([05](05-EFCore.md)), the
  API, and the message contract. It's the same shape each time: add the new thing, write both,
  migrate readers, remove the old thing.
- **Tolerant readers** (ignore unknown fields, don't assume field order, tolerate new enum values)
  turn most producer changes into non-events — the cheapest versioning strategy there is.
- **Flag hygiene**: give every flag an owner and an expiry date at creation, and treat removal as
  part of the feature's definition of done. Otherwise you accumulate untested combinations
  exponentially.

→ Deeper: [`../22-BestPractices/`](../22-BestPractices/README.md)
