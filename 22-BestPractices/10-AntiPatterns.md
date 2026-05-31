# Anti-Patterns to Retire

## Recognizing and replacing recurring mistakes

An **anti-pattern** is a common "solution" that looks reasonable but causes more problems than it solves. This file catalogs the .NET/architecture anti-patterns you'll encounter most — what they are, **why** they're harmful, and what to do **instead** — so you can recognize and retire them in code review and your own work. They recur because each *feels* convenient in the moment; the cost shows up later in coupling, untestability, bugs, and outages. (Language-level anti-patterns are in [CSharpBook Ch17 §09](../../CSharpBook/17-BestPractices/README.md); this is the platform/architecture set.)

---

## Service Locator

**What**: pulling dependencies out of a container on demand (`serviceProvider.GetService<T>()`) inside a class, instead of declaring them as constructor parameters.

```csharp
// ✗ Service Locator — hidden dependencies, can't tell what this class needs
public class OrderService {
    public void Process() {
        var repo = _serviceProvider.GetService<IOrderRepository>();   // hidden
        var email = _serviceProvider.GetService<IEmailSender>();      // hidden
    }
}
// ✓ Constructor injection — dependencies explicit, testable
public class OrderService(IOrderRepository repo, IEmailSender email) { ... }
```

**Why bad**: dependencies are **hidden** (you can't tell what the class needs from its constructor), it's **hard to test** (must configure a container, not just pass fakes), and it couples the class to the container. **Instead**: **constructor injection** ([Ch03 §02](../03-HostingAndDI/02-DependencyInjection.md)) — declare dependencies as parameters; the container provides them. (Service location is occasionally justified at a genuine composition boundary, but not as a general pattern.)

---

## Anemic Domain Model

**What**: domain entities that are just data bags (public getters/setters, no behavior), with all logic in external "service" classes ([02-DomainPersistence.md](02-DomainPersistence.md)).

**Why bad**: business rules are **scattered** across services, **invariants are unprotected** (anything can set any property to any value), and the model doesn't express the domain. **Instead**: for complex domains, a **rich domain model** that owns behavior and protects invariants ([02-DomainPersistence.md](02-DomainPersistence.md)) — though anemic DTOs are fine for genuine CRUD.

---

## Sync-over-async

**What**: blocking on async code with `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()`.

```csharp
var data = httpClient.GetStringAsync(url).Result;   // ✗ blocks a thread on async work
```

**Why bad**: it **blocks a thread** waiting on async work, causing **thread-pool starvation** under load ([Ch21 §10](../21-Performance/10-CommonWins.md)) — the app becomes unresponsive — and can **deadlock** in contexts with a synchronization context. **Instead**: **async all the way** ([12-AsyncIdiomsAtScale.md](12-AsyncIdiomsAtScale.md), [CSharpBook Ch08](../../CSharpBook/08-Concurrency/README.md)) — `await` and propagate async up the call chain.

---

## Fat Controllers / God Classes

**What**: controllers (or "services") stuffed with business logic, data access, validation, and orchestration — hundreds of lines doing everything.

**Why bad**: violates **single responsibility** ([CSharpBook Ch17 SOLID](../../CSharpBook/17-BestPractices/README.md)), is hard to test (logic tangled with HTTP concerns), and becomes a change magnet. **Instead**: **thin controllers** that delegate to the application layer — e.g., send a command/query to a handler ([04-CQRS.md](04-CQRS.md)) or call a focused application service; controllers should orchestrate HTTP (bind, call, return), not contain business logic.

---

## Magic Strings (and primitive obsession)

**What**: hardcoded string literals for keys, routes, config paths, roles, etc. scattered through code (`config["ConnctionStrings:Db"]` — note the typo that compiles!), and using primitives where a type belongs (`string customerId` everywhere).

**Why bad**: **typos compile** and fail at runtime, no refactoring safety, no single source of truth. **Instead**: **constants/`nameof`/strongly-typed** options ([Ch13 §05](../13-Configuration/05-Options.md)), typed route helpers, enums for roles, and **value objects** ([02-DomainPersistence.md](02-DomainPersistence.md)) / strongly-typed ids instead of bare strings — let the compiler catch mistakes.

---

## More to retire

| Anti-pattern | Why bad | Instead |
|---|---|---|
| **Generic `Repository<T>` over EF Core** | duplicates/leaks EF Core, hides its power | use `DbContext`/`DbSet` directly or a domain-specific repo ([03](03-RepositoryUnitOfWork.md)) |
| **Catch-all `catch (Exception) {}`** (swallowing) | hides failures, corrupts state silently | catch specific exceptions; let unexpected ones propagate ([CSharpBook Ch17 §13](../../CSharpBook/17-BestPractices/README.md)) |
| **Exceptions for control flow** | slow, obscures intent ([Ch21 §10](../21-Performance/10-CommonWins.md)) | `Try...`/result patterns for expected cases |
| **`async void`** (non-event-handler) | can't await/catch — crashes the process | `async Task` ([CSharpBook Ch08](../../CSharpBook/08-Concurrency/README.md)) |
| **Singleton holding scoped deps (captive dependency)** | the scoped service lives too long ([Ch03 §03](../03-HostingAndDI/03-Lifetimes.md)) | match lifetimes; use a factory/`IServiceScopeFactory` |
| **N+1 queries** | a flood of DB round-trips ([Ch05 §02](../05-EFCore/02-Querying.md)) | eager load (`Include`)/project |
| **Secrets in source/config** | leaks, hard to rotate ([Ch13 §07](../13-Configuration/07-Secrets.md)) | Key Vault/user secrets + managed identity |
| **Premature microservices** | distributed complexity unearned ([01](01-SolutionLayout.md)) | start with a (modular) monolith |
| **Premature optimization** | complexity for unproven gains ([Ch21 §01](../21-Performance/01-Mindset.md)) | measure first, then optimize the hot path |

---

## The recurring theme

Most anti-patterns share roots: **hidden dependencies/coupling** (service locator, magic strings, fat controllers), **broken invariants/safety** (anemic model, swallowed exceptions, captive dependencies), or **doing the convenient thing that doesn't scale** (sync-over-async, N+1, premature microservices). The replacements share principles too — **explicit dependencies** (DI), **encapsulation/invariants** (rich models, value objects), **async correctness**, **single responsibility**, and **measure-don't-guess**. Recognizing the *shape* of these mistakes lets you catch them in review before they ossify.

---

## Common gotchas

### Treating these as absolute rules

Most have legitimate exceptions (service location at a composition boundary; anemic models for true CRUD; `async void` for event handlers). They're *defaults to avoid*, applied with judgment — not dogma.

### Replacing one anti-pattern with over-engineering

Fixing a fat controller by introducing full CQRS + event sourcing for a trivial app trades one problem for another ([04-CQRS.md](04-CQRS.md)). Replace with the **simplest** correct alternative.

### Not recognizing the subtle ones

Captive dependencies, sync-over-async, and N+1 are *invisible* until load/scale exposes them. Use the tooling — counters/dumps ([Ch21](../21-Performance/README.md)), analyzers, code review — to catch them early.

### Cargo-culting the replacement

Constructor injection, value objects, and CQRS are good *when they fit* — applying them everywhere mechanically is its own anti-pattern. Understand *why* the replacement is better and apply it where the problem actually exists.

---

## Summary

- **Anti-patterns** are convenient-looking "solutions" that cause more harm than good; recognize and retire them — chiefly **Service Locator** (→ constructor injection — [Ch03 §02](../03-HostingAndDI/02-DependencyInjection.md)), **Anemic Domain Model** (→ rich model for complex domains — [02](02-DomainPersistence.md)), **sync-over-async** (→ async all the way — thread-pool starvation/deadlocks — [12](12-AsyncIdiomsAtScale.md)), **fat controllers** (→ thin controllers + handlers — [04](04-CQRS.md)), and **magic strings/primitive obsession** (→ constants/`nameof`/value objects).
- Also retire: generic `Repository<T>` over EF Core ([03](03-RepositoryUnitOfWork.md)), swallowed/control-flow exceptions, `async void`, **captive dependencies** ([Ch03 §03](../03-HostingAndDI/03-Lifetimes.md)), **N+1 queries** ([Ch05 §02](../05-EFCore/02-Querying.md)), secrets in source ([Ch13 §07](../13-Configuration/07-Secrets.md)), and **premature** microservices/optimization.
- The shared roots are **hidden coupling**, **broken invariants/safety**, and **convenient-but-unscalable** choices; the replacements share **explicit dependencies, encapsulation, async correctness, single responsibility, and measure-first**.
- Apply with **judgment** (most have legitimate exceptions), replace with the **simplest correct** alternative (not over-engineering), and use **tooling/review** to catch the subtle ones early.

→ Next: [11-CodeOrganization.md](11-CodeOrganization.md)
