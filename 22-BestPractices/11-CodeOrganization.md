# Code Organization

## Structure within a project

[01-SolutionLayout.md](01-SolutionLayout.md) covered *projects*; this is the level **below** — how you organize files, folders, namespaces, and visibility **within** a project. Good intra-project organization makes a codebase **navigable** (you can find things), **boundaried** (the public surface is clear and small), and **resistant to erosion** (conventions keep it consistent as it grows). The recurring principles: **organize by feature where it aids change-locality**, **keep the public surface small** (`internal` by default), **name consistently**, and **make conventions enforceable** (analyzers/EditorConfig) rather than relying on memory.

---

## Folder structure: by feature vs by type

Within a project, two organizing axes (mirroring the layers-vs-slices debate — [06-VerticalSlices.md](06-VerticalSlices.md)):

```
BY TYPE (technical)              BY FEATURE (vertical)
  Controllers/                    Orders/
  Services/                         OrdersController.cs
  Models/                           PlaceOrderHandler.cs
  Validators/                       OrderValidator.cs
  ...                             Customers/
                                    ...
```

- **By type** — group `Controllers/`, `Services/`, `Models/`. Familiar, but one feature's files are **scattered**; folders grow large and shapeless.
- **By feature** — group everything for `Orders/` together. Code that **changes together lives together** ([06-VerticalSlices.md](06-VerticalSlices.md)); features are easy to find and reason about; scales better for feature-rich apps.

Prefer **by feature** for non-trivial apps (it aligns with how you actually work), reserving by-type for small projects or genuinely cross-cutting buckets (e.g., a shared `Infrastructure/`). The goal is that someone looking for "the order code" finds it in one place.

---

## Visibility: `internal` by default

A project's **public API** is its contract with other projects ([09-Versioning.md](09-Versioning.md)). The smaller it is, the more freely you can change internals. So: **make types `internal` unless they *need* to be `public`**:

```csharp
public class OrderService { }       // part of the project's public surface — a commitment
internal class OrderValidator { }   // implementation detail — free to change/refactor
```

- **`internal`** types are visible only within the assembly — they're implementation details you can refactor without breaking consumers.
- **`public`** types are the deliberate, minimal surface other projects depend on.
- For **tests** to reach `internal` types, use **`[InternalsVisibleTo("MyApp.Tests")]`** ([Ch17 §05](../17-Testing/05-IntegrationTests.md)) rather than making everything public.
- C# 11's **`file`** modifier scopes a type to a single source file — even tighter than `internal` for a helper used by only one file.

Defaulting to `internal` keeps the public surface intentional and small — easier to version, document, and reason about. (This is what enforces module boundaries in a modular monolith — [01-SolutionLayout.md](01-SolutionLayout.md).)

---

## Namespace conventions

Namespaces should reflect **folder structure** and the project's organization, giving a predictable mapping from "where a type lives" to "what it's called":

- **Match namespaces to folders** (`MyApp.Orders.PlaceOrder` for `Orders/PlaceOrder/`) — the default analyzers expect, and it makes navigation predictable.
- **File-scoped namespaces** (C# 10) reduce nesting/indentation:

```csharp
namespace MyApp.Orders;   // file-scoped — no extra braces/indentation

internal class OrderValidator { }
```

- **Avoid over-deep namespaces** — mirror the meaningful structure, not every folder level.
- **`global using`** (C# 10) for ubiquitous namespaces (in a `GlobalUsings.cs`) reduces per-file using clutter ([CSharpBook Ch10](../../CSharpBook/10-AdvancedLanguage/README.md)).

Consistent namespace/folder alignment is a small thing that compounds — it removes friction in finding and referencing types.

---

## Enforcing conventions

Conventions only hold if they're **enforced automatically**, not left to memory/discipline:

- **`.editorconfig`** — codifies formatting, naming rules, and analyzer severities **per repo**, applied by the IDE and build ([Ch19 §09](../19-Deployment/09-CICD.md)). It makes "the team's style" machine-checked.
- **Roslyn analyzers** — flag violations (naming, async misuse, disposability, security) at build time; treat important ones as **errors** so they can't merge.
- **`Directory.Build.props`** — apply `Nullable`, `TreatWarningsAsErrors`, analyzer packages, and `LangVersion` across all projects from one place ([01-SolutionLayout.md](01-SolutionLayout.md)).
- **Naming conventions** — follow the standard .NET conventions (PascalCase for public members/types, `_camelCase` for private fields, `I`-prefix for interfaces — [CSharpBook Ch17 §01](../../CSharpBook/17-BestPractices/README.md)); encode them in `.editorconfig` so they're enforced.

The principle: **automate the conventions** (analyzers + EditorConfig + build settings) so consistency survives team growth and time, rather than depending on reviewers catching every deviation.

---

## Common gotchas

### Everything `public`

A large public surface makes every refactor a potential breaking change and obscures what's actually a contract. Default to **`internal`**; expose only what other projects truly need; use `[InternalsVisibleTo]` for tests.

### By-type folders that don't scale

A flat `Services/` with 80 files is unnavigable. Organize **by feature** for non-trivial apps so related code is co-located ([06-VerticalSlices.md](06-VerticalSlices.md)).

### Namespaces not matching folders

Mismatched namespace/folder structure confuses navigation and trips analyzers. Keep them aligned (use file-scoped namespaces to reduce noise).

### Conventions only in a wiki

A style guide nobody enforces drifts immediately. Encode rules in **`.editorconfig`** + analyzers (as build errors where it matters) so they're automatic.

### Over-deep namespaces / folders

Mirroring every folder level into deep namespaces (`MyApp.Features.Orders.Commands.PlaceOrder.Handlers`) adds friction. Keep structure meaningful, not maximal.

---

## Summary

- **Within a project**, organize for **navigability**, a **small clear public surface**, and **enforceable consistency** — generally **by feature** (co-locate code that changes together — [06-VerticalSlices.md](06-VerticalSlices.md)) rather than by technical type for non-trivial apps.
- Keep the **public API small**: make types **`internal` by default** (free to refactor), expose only deliberate `public` contracts ([09-Versioning.md](09-Versioning.md)), use **`[InternalsVisibleTo]`** for tests and the **`file`** modifier for single-file helpers.
- **Align namespaces to folders** (use **file-scoped namespaces** and `global using` to cut noise); follow standard .NET naming conventions.
- **Automate conventions** with **`.editorconfig`**, **Roslyn analyzers** (errors for the important ones), and **`Directory.Build.props`** (nullable, warnings-as-errors, analyzers) so consistency survives time and team growth — don't rely on memory/review alone.

→ Next: [12-AsyncIdiomsAtScale.md](12-AsyncIdiomsAtScale.md)
