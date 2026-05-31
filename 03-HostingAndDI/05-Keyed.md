# Keyed Services

## Multiple implementations, selected by key (.NET 8+)

Sometimes you have several implementations of one interface and want to pick a specific one **by name/key** rather than getting all of them. Before .NET 8 this required workarounds (factories, marker interfaces). **Keyed services** make it first-class: register implementations under a key, resolve by that key.

```csharp
// Register implementations under keys
services.AddKeyedSingleton<INotifier, EmailNotifier>("email");
services.AddKeyedSingleton<INotifier, SmsNotifier>("sms");
services.AddKeyedSingleton<INotifier, PushNotifier>("push");

// Resolve by key in a constructor with [FromKeyedServices]
public class AlertService([FromKeyedServices("email")] INotifier emailNotifier) {
    // gets EmailNotifier specifically
}

// Or resolve programmatically
var sms = provider.GetRequiredKeyedService<INotifier>("sms");
```

---

## Why keyed services

The problem they solve: **"I have N implementations of `IFoo` and need to choose one by name."** Classic scenarios:
- Multiple notification channels (`email`/`sms`/`push`) selected per situation.
- Multiple storage backends (`primary`/`replica`, `s3`/`azure`).
- Multiple named HTTP clients or strategies.
- Multiple `JsonSerializerOptions` profiles.

Before keyed services you'd register all and inject `IEnumerable<IFoo>` then filter, or build a factory/dictionary by hand. Keyed services express the intent directly and let the container do the selection.

---

## Registration & resolution API

```csharp
// Register (all lifetimes supported)
services.AddKeyedSingleton<IFoo, FooA>("a");
services.AddKeyedScoped<IFoo, FooB>("b");
services.AddKeyedTransient<IFoo, FooC>("c");

// Keys can be any object, not just strings (enums read well)
services.AddKeyedSingleton<IFoo, FooA>(StorageKind.Primary);

// Factory with access to the key
services.AddKeyedSingleton<IFoo>("x", (sp, key) => new Foo(key!.ToString()!));

// Resolve
[FromKeyedServices("a")] IFoo foo                       // constructor injection
provider.GetRequiredKeyedService<IFoo>("a");            // throws if missing
provider.GetKeyedService<IFoo>("a");                    // null if missing
provider.GetKeyedServices<IFoo>("a");                   // all under that key
```

Keys are `object` — strings are common, but **enums** are clearer and refactor-safe. `[FromKeyedServices(key)]` on a constructor parameter is the declarative way; `GetRequiredKeyedService` is the imperative way.

---

## `KeyedService.AnyKey` and mixing

```csharp
// Resolve a service registered with ANY key (when the specific key is dynamic)
services.AddKeyedSingleton<IFoo>(KeyedService.AnyKey, (sp, key) => Create(key));

// Keyed and non-keyed registrations of the same type coexist independently:
services.AddSingleton<IFoo, DefaultFoo>();              // the "default" (no key)
services.AddKeyedSingleton<IFoo, SpecialFoo>("special"); // keyed
// GetRequiredService<IFoo>() → DefaultFoo;  GetRequiredKeyedService<IFoo>("special") → SpecialFoo
```

Keyed and keyless registrations are separate namespaces — a keyless `GetService<IFoo>()` won't see keyed registrations and vice versa. `KeyedService.AnyKey` lets a single factory handle whatever key is requested (the key is passed to the factory).

---

## Choosing the key at runtime

A common pattern: a dispatcher that selects an implementation based on runtime data:

```csharp
public class NotificationDispatcher(IServiceProvider services) {
    public Task SendAsync(string channel, Message msg) {
        var notifier = services.GetRequiredKeyedService<INotifier>(channel);   // channel = "email"/"sms"/...
        return notifier.SendAsync(msg);
    }
}
```

Here injecting `IServiceProvider` (to resolve by a runtime key) is a legitimate, factory-like use — not service-locator abuse, because the *set* of dependencies is well-defined and the key comes from runtime input. For a fixed key known at compile time, prefer `[FromKeyedServices("...")]`.

---

## Keyed vs the alternatives

| Approach | When |
|---|---|
| **Keyed services** | Select one of N implementations **by a key** (.NET 8+) — the clean default |
| `IEnumerable<IFoo>` + filter | You need **all** implementations (e.g., run every validator) |
| `Func<key, IFoo>` factory | Custom creation logic per key, or pre-.NET 8 |
| Separate interfaces | The implementations are genuinely different abstractions (prefer this if they don't share a real contract) |

If two "implementations" don't actually share a meaningful contract, don't force them under one keyed interface — give them distinct interfaces. Keyed services are for **genuine variants of the same abstraction**.

---

## Common gotchas

### Keyed and keyless are separate

`GetService<IFoo>()` does **not** return keyed registrations; you must use `GetKeyedService<IFoo>(key)`. Mixing them up yields "no service registered" surprises.

### String keys and typos

A mistyped string key fails at resolution (`GetRequiredKeyedService` throws). Prefer **enum keys** for compile-time safety and refactorability.

### Overusing keys instead of distinct types

If variants don't share a real contract, separate interfaces are clearer than one keyed interface. Keyed services suit true variants (channels, backends), not unrelated services crammed together.

### Resolving by runtime key everywhere

Resolving by key in a dispatcher is fine; sprinkling `GetRequiredKeyedService` throughout business logic drifts toward service-location. Centralize key-based selection in a focused dispatcher/factory.

---

## Summary

- **Keyed services** (.NET 8+) register multiple implementations of one interface under **keys** and resolve a specific one by key — `AddKeyed{Singleton,Scoped,Transient}` + `[FromKeyedServices(key)]` / `GetRequiredKeyedService`.
- They cleanly express "pick one of N variants by name" (notification channels, storage backends, strategies) — replacing hand-rolled factories/`IEnumerable` filtering.
- Keys are `object` (**enums** preferred over strings); keyed and keyless registrations are **separate**; `KeyedService.AnyKey` + a factory handles dynamic keys.
- Use keyed services for **genuine variants of one abstraction**; use distinct interfaces when implementations don't share a real contract, and `IEnumerable<T>` when you need them all.

→ Next: [06-Decorate.md](06-Decorate.md)
