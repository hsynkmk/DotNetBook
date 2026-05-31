# CQRS

## Separating reads from writes

**CQRS** (Command Query Responsibility Segregation) is the principle that **commands** (operations that *change* state) and **queries** (operations that *read* state) are different responsibilities and can be modeled, optimized, and even stored separately. Instead of one model/service doing both, you have **command handlers** (perform an action, enforce business rules — [02-DomainPersistence.md](02-DomainPersistence.md)) and **query handlers** (return data shaped for a view, often bypassing the domain model). It ranges from a lightweight code organization (separate handlers, same database) to a full architecture (separate read/write *stores*). The art is applying the **right amount** — CQRS is frequently **overkill**.

```csharp
// Command — changes state, returns nothing meaningful (or an id/result):
public record PlaceOrderCommand(int CustomerId, List<LineDto> Lines) : IRequest<int>;
public class PlaceOrderHandler(AppDbContext db) : IRequestHandler<PlaceOrderCommand, int> {
    public async Task<int> Handle(PlaceOrderCommand cmd, CancellationToken ct) {
        var order = Order.Place(cmd.CustomerId, cmd.Lines);   // domain enforces rules
        db.Orders.Add(order);
        await db.SaveChangesAsync(ct);
        return order.Id;
    }
}

// Query — returns a read-shaped DTO, can bypass the domain and project directly:
public record GetOrderSummary(int OrderId) : IRequest<OrderSummaryDto>;
public class GetOrderSummaryHandler(AppDbContext db) : IRequestHandler<GetOrderSummary, OrderSummaryDto> {
    public Task<OrderSummaryDto> Handle(GetOrderSummary q, CancellationToken ct) =>
        db.Orders.Where(o => o.Id == q.OrderId)
                 .Select(o => new OrderSummaryDto(o.Id, o.Total, o.Status))   // project for the view
                 .FirstAsync(ct);
}
```

---

## Why separate reads and writes

Reads and writes have genuinely different needs:

| | Commands (writes) | Queries (reads) |
|---|---|---|
| Goal | enforce business rules, change state | return data fast, shaped for a view |
| Model | **rich domain model**, aggregates ([02](02-DomainPersistence.md)) | flat **DTOs/projections**, no domain |
| Concerns | validation, invariants, transactions | performance, denormalization, paging |
| Frequency | fewer | usually **many more** |

Forcing both through one model causes friction: the rich domain model (great for protecting write invariants) is awkward and slow for reads (you load whole aggregates to display two fields). CQRS lets **writes** go through the domain (rules enforced) and **reads** **project directly** to view-shaped DTOs ([Ch05 §09](../05-EFCore/09-Performance.md)) — each optimized for its job. This separation is the core benefit even in its lightweight form.

---

## The mediator pattern (MediatR)

CQRS is commonly implemented with a **mediator** (the popular library is **MediatR**): controllers/endpoints send a **request** (command or query) and a mediator routes it to the matching **handler** — decoupling the caller from the handler:

```csharp
// Endpoint just sends the request — doesn't know the handler:
app.MapPost("/orders", async (PlaceOrderCommand cmd, ISender mediator) =>
    Results.Ok(await mediator.Send(cmd)));
```

Benefits: thin controllers ([10-AntiPatterns.md](10-AntiPatterns.md) warns against fat ones), one handler per use case (single responsibility — [CSharpBook Ch17 SOLID](../../CSharpBook/17-BestPractices/README.md)), and **pipeline behaviors** — cross-cutting concerns (validation, logging, transactions, caching) wrapped around every handler in one place:

```csharp
// A validation behavior runs for every request, before its handler:
public class ValidationBehavior<TReq, TRes> : IPipelineBehavior<TReq, TRes> { /* validate, then next() */ }
```

> **Note**: MediatR became commercially licensed in recent versions; alternatives (or hand-rolled dispatch) exist. The **pattern** (request → handler, pipeline behaviors) is what matters, not the specific library.

---

## The spectrum: lightweight to full CQRS

CQRS isn't all-or-nothing — it's a spectrum:

1. **Lightweight (most common, recommended default when CQRS fits)** — separate command/query **handlers**, **same database**, reads project directly to DTOs while writes go through the domain. Simple, big readability win, no extra infrastructure.
2. **Separate read models** — a denormalized read store/views updated from the write side (often via domain events — [05-DomainEvents.md](05-DomainEvents.md)) for read performance at scale.
3. **Full CQRS with separate stores + event sourcing** — distinct write store (events) and read store (projections), eventually consistent. Powerful but **complex** (eventual consistency, projections, replay) — justified only for specific high-scale/audit needs.

Most teams should use **level 1**; escalate only when read/write scaling or modeling pressure demands it. Jumping to full CQRS+event sourcing for a normal app is a classic over-engineering trap.

---

## When CQRS is overkill

CQRS adds indirection (more types: commands, queries, handlers). For a **simple CRUD app** with no meaningful difference between read and write models, it's ceremony — direct controllers + EF Core ([03-RepositoryUnitOfWork.md](03-RepositoryUnitOfWork.md)) are simpler and clearer. CQRS earns its keep when:

- Read and write models genuinely **diverge** (complex domain writes, very different read shapes).
- You want **uniform cross-cutting handling** (validation/logging/transactions via pipeline behaviors).
- Reads need **independent optimization/scaling** from writes.

If none of those apply, skip it. "CQRS everywhere" produces a maze of handlers for trivial operations.

---

## Common gotchas

### Full CQRS + event sourcing by default

Separate stores and event sourcing bring eventual consistency, projections, and replay complexity that most apps don't need. Start **lightweight** (handlers + one database); escalate only for concrete scaling/audit needs.

### CQRS on a simple CRUD app

Commands/queries/handlers for trivial create-read-update-delete is needless ceremony. Use direct controllers + EF Core when read and write models don't diverge.

### Fat handlers / leaking the domain into queries

Queries should **project to DTOs** (bypass the domain for reads); commands go through the domain. Loading full aggregates for reads (or putting business logic in queries) loses the benefit.

### Coupling to a specific mediator library

The value is the **pattern** (request→handler, pipeline behaviors), not MediatR specifically (now licensed). Keep handlers framework-light so the dispatch mechanism is swappable.

### Pipeline behaviors hiding too much

Cross-cutting behaviors (validation, transactions) are powerful but can obscure control flow if overused. Keep them focused and few.

---

## Summary

- **CQRS** separates **commands** (change state, go through the **rich domain** + enforce rules) from **queries** (read, **project directly to view-shaped DTOs**, bypass the domain) — each optimized for its different needs.
- It's commonly implemented with a **mediator** (MediatR-style: request → handler) giving **thin controllers**, one handler per use case, and **pipeline behaviors** for cross-cutting concerns (validation/logging/transactions) — though the *pattern* matters more than the (now-licensed) library.
- It's a **spectrum**: **lightweight** (separate handlers, one DB — the recommended default when CQRS fits) → separate **read models** → full **CQRS + event sourcing** (separate stores, eventual consistency — complex, only for high-scale/audit needs).
- CQRS is **overkill** for simple CRUD where read/write models don't diverge — use direct controllers + EF Core instead; adopt CQRS when models diverge, you want uniform cross-cutting handling, or reads need independent scaling.

→ Next: [05-DomainEvents.md](05-DomainEvents.md)
