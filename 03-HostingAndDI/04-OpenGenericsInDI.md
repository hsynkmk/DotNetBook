# Open Generics in DI

## Registering a generic family in one line

You often have a generic service (`IRepository<T>`, `IValidator<T>`, `IHandler<TCommand>`) with a single generic implementation. Registering each closed type by hand is tedious and doesn't scale. **Open generic registration** registers the whole family at once — the container closes the generic for whatever `T` is requested.

```csharp
// Register the OPEN generic (note typeof with the empty <>)
services.AddScoped(typeof(IRepository<>), typeof(EfRepository<>));

// Now ANY closed type resolves automatically:
public class OrderService(IRepository<Order> orders, IRepository<Customer> customers) { }
//   IRepository<Order>    → EfRepository<Order>
//   IRepository<Customer> → EfRepository<Customer>
```

The container creates `EfRepository<Order>` for `IRepository<Order>`, `EfRepository<Customer>` for `IRepository<Customer>`, etc. — without you registering each one.

---

## Why this matters

Without open generics you'd write:

```csharp
// ✗ — one registration per entity; doesn't scale, easy to forget one
services.AddScoped<IRepository<Order>, EfRepository<Order>>();
services.AddScoped<IRepository<Customer>, EfRepository<Customer>>();
services.AddScoped<IRepository<Product>, EfRepository<Product>>();
// ...one per entity, forever
```

Open generic registration is **one line for all of them**, present and future. It's the foundation of generic infrastructure patterns: generic repositories, generic validators (`IValidator<T>`), the **MediatR**-style `IRequestHandler<TRequest,TResponse>`, generic caching decorators, and `ILogger<T>` itself (the framework registers `typeof(ILogger<>)` → a logger factory).

```csharp
services.AddTransient(typeof(IValidator<>), typeof(DataAnnotationsValidator<>));
services.AddScoped(typeof(ICache<>), typeof(MemoryCache<>));
```

---

## How resolution works

When you request `IRepository<Order>`:
1. The container finds no closed registration for `IRepository<Order>`.
2. It finds the **open** registration `IRepository<>` → `EfRepository<>`.
3. It closes the implementation with the requested type argument → `EfRepository<Order>` (via `MakeGenericType` — [Ch01 §05](../01-Runtime/05-TypeSystem.md)).
4. It resolves and constructs that closed type's dependency graph.

The generic arguments must match: `IRepository<>` (one type param) maps to `EfRepository<>` (one type param). The implementation's constructor dependencies are resolved normally.

---

## Constraints flow through

If the open implementation has generic **constraints**, they're enforced — a request for a type that doesn't satisfy them fails:

```csharp
public class EntityRepository<T> : IRepository<T> where T : class, IEntity {
    public EntityRepository(AppDbContext db) { /* ... */ }
}
services.AddScoped(typeof(IRepository<>), typeof(EntityRepository<>));

// IRepository<Order> works only if Order : class, IEntity
```

This lets you constrain a generic implementation to applicable types (e.g., only entities) while still registering it open.

---

## Mixing open and closed registrations

You can register the open generic as the default and **override** specific closed types:

```csharp
services.AddScoped(typeof(IRepository<>), typeof(EfRepository<>));       // default for all T
services.AddScoped<IRepository<AuditLog>, AuditLogRepository>();          // special-case one T
```

A closed registration for `IRepository<AuditLog>` takes precedence for that specific type, while everything else falls back to the open generic. Useful when most entities share a generic repo but one needs custom behavior.

---

## Multiple open implementations → `IEnumerable<T>`

Registering several open implementations resolves as a sequence — the basis of pipeline/handler patterns:

```csharp
services.AddScoped(typeof(IBehavior<>), typeof(LoggingBehavior<>));
services.AddScoped(typeof(IBehavior<>), typeof(ValidationBehavior<>));
// constructor: (IEnumerable<IBehavior<MyRequest>> behaviors) → both, in registration order
```

This underpins MediatR pipeline behaviors and similar "chain of generic handlers" designs.

---

## AOT/trimming note

Open generic registration relies on the container closing the generic at runtime (`MakeGenericType`). Under **Native AOT**, instantiations not seen at build time can fail ([Ch01 §03](../01-Runtime/03-NativeAOT.md)) — though the DI container and common patterns are increasingly AOT-annotated, and ASP.NET Core's AOT path uses source-generated DI where possible. For AOT apps, verify your generic registrations resolve (the analyzers will warn about problematic cases).

---

## Common gotchas

### Mismatched arity

`typeof(IRepository<>)` (one param) can't map to a two-param implementation. The open service and implementation must have the same number of type parameters.

### Forgetting the empty `<>`

`typeof(IRepository<>)` is the open generic; `typeof(IRepository<Order>)` is closed. Open registration needs the empty-brackets form. `typeof(IRepository)` (no brackets) won't compile for a generic type.

### Expecting closed registration to "win" globally

A closed registration only overrides that **specific** closed type; other type arguments still use the open generic. That's the intended behavior, but verify precedence matches your intent.

### Constraint violations at runtime

Requesting a closed type that violates the implementation's generic constraints fails at resolution. Keep constraints aligned with how the service is used.

---

## Summary

- **Open generic registration** (`services.Add...(typeof(IRepository<>), typeof(EfRepository<>))`) registers an entire generic family in one line; the container closes the implementation for each requested `T`.
- It powers generic repositories, validators, MediatR-style handlers, caches, and `ILogger<T>` — avoiding one-registration-per-type drudgery.
- **Constraints** on the open implementation are enforced; you can **override** specific closed types alongside the open default; multiple open implementations resolve as `IEnumerable<T>`.
- Use the empty `<>` form; arities must match; verify AOT compatibility for generic instantiations.

→ Next: [05-Keyed.md](05-Keyed.md)
