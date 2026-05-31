# Reflection (BCL APIs)

## Inspecting and invoking types at runtime

Reflection lets code examine and use types it didn't know at compile time — read metadata, create instances, invoke members, query attributes. It powers DI containers, serializers, ORMs, test runners, and plugin systems.

> CSharpBook Chapter 12 covers reflection comprehensively (the `Type`/`MemberInfo` hierarchy, `Activator`, attributes, `dynamic`, source generators, performance/caching, AOT). This file is the **BCL API quick-reference** and points you there for the deep mechanics — plus the runtime-internals view (how reflection reads metadata) is in [Ch01 §07](../01-Runtime/07-Metadata.md).

---

## Entry points: getting a `Type`

```csharp
Type t1 = typeof(Customer);              // compile-time literal (cheap: ldtoken)
Type t2 = instance.GetType();             // runtime type of an object
Type? t3 = Type.GetType("Ns.Customer, MyAssembly");  // by assembly-qualified name string
```

`Type` is the gateway to all reflection — it exposes a type's members, base type, interfaces, generic arguments, and attributes. (`typeof` vs `GetType`: CSharpBook Ch12 §06.)

---

## The member hierarchy

```csharp
Type t = typeof(Customer);
MethodInfo[] methods = t.GetMethods(BindingFlags.Public | BindingFlags.Instance);
PropertyInfo? name = t.GetProperty("Name");
FieldInfo? field = t.GetField("_id", BindingFlags.NonPublic | BindingFlags.Instance);
ConstructorInfo? ctor = t.GetConstructor([typeof(int), typeof(string)]);

// Read/invoke
object? value = name?.GetValue(instance);
name?.SetValue(instance, "Alice");
object? result = methods[0].Invoke(instance, args);
```

`BindingFlags` controls visibility/static-vs-instance (the default excludes non-public and static — a common surprise). Full hierarchy (`MemberInfo` → `MethodBase`/`PropertyInfo`/`FieldInfo`/`EventInfo`) and generics (`MakeGenericType`/`MakeGenericMethod`): CSharpBook Ch12 §01.

---

## Creating instances

```csharp
var c = Activator.CreateInstance<Customer>();              // generic, needs new(); JIT-inlined (.NET 6+)
var d = (Customer)Activator.CreateInstance(typeof(Customer), arg1, arg2)!;  // with ctor args (slow)
var e = Activator.CreateInstance(typeof(Singleton), nonPublic: true);       // private ctor
```

`Activator.CreateInstance<T>()` is fast (JIT special-cased); the non-generic with-args overload is slow (~hundreds of ns). For repeated creation, compile a factory. (CSharpBook Ch12 §02.)

---

## Attributes

```csharp
var attr = typeof(Customer).GetCustomAttribute<TableAttribute>();    // instantiates the attribute
bool has = typeof(Customer).IsDefined(typeof(ObsoleteAttribute));     // cheaper presence check
var props = typeof(Customer).GetProperties()
    .Where(p => p.IsDefined(typeof(RequiredAttribute)));
```

Attributes are metadata read at runtime — the basis of declarative frameworks (`[Route]`, `[Required]`, `[Fact]`). Args must be compile-time constants (stored in the metadata blob — [Ch01 §07](../01-Runtime/07-Metadata.md)). `IsDefined` is faster than `GetCustomAttribute` when you only need yes/no. (CSharpBook Ch12 §03.)

---

## The performance reality

Reflection is **correct but slow** — each operation is metadata lookup + dynamic dispatch (~hundreds of ns vs a direct call's ~1 ns), plus boxing of value-type args/returns:

| Approach | Relative cost |
|---|---|
| Direct call | 1× |
| Cached `MemberInfo` + `Invoke` | ~100–500× |
| Compiled delegate (`Expression`/`Delegate.CreateDelegate`) | ~5–20× (after one-time compile) |
| **Source generator** | 1× (direct code, AOT-safe) |

For hot paths: **cache** the `MemberInfo`, **compile** to a typed delegate, or — best — use a **source generator** to do the work at compile time (also the only AOT-safe option). This is the single most important reflection lesson. (CSharpBook Ch12 §05, §07.)

```csharp
// Cache + compile a property getter once, reuse cheaply:
private static readonly Func<Customer, string> GetName =
    (Func<Customer, string>)Delegate.CreateDelegate(
        typeof(Func<Customer, string>), typeof(Customer).GetProperty("Name")!.GetGetMethod()!);
```

---

## Reflection and AOT/trimming

Reflection by runtime string is invisible to the trimmer/AOT analyzer, so referenced types/members may be removed → null/missing at runtime (`IL2xxx` warnings — [Ch01 §03](../01-Runtime/03-NativeAOT.md)). Mitigate by:
- Annotating with `[DynamicallyAccessedMembers]` / `[RequiresUnreferencedCode]`.
- Replacing runtime reflection with **source generators** (the modern direction: STJ source-gen, `[GeneratedRegex]`, `LoggerMessage`, `[LibraryImport]`).

The platform-wide trend: do at **compile time** (source generators read metadata then) what used to be done at **runtime** (reflection) — faster startup, no per-call cost, AOT-safe.

---

## When reflection is the right tool

- One-time setup (DI container registration, MVC controller discovery at startup).
- Plugin/extension loading (with `AssemblyLoadContext` — [Ch01 §06](../01-Runtime/06-Assemblies.md)).
- Genuinely dynamic scenarios where types aren't known until runtime.

When it's the **wrong** tool: hot paths (use compiled delegates/source-gen), AOT/trimmed apps (use source-gen), or anywhere a `Dictionary<string, Func<T>>` dispatch table would do (CSharpBook Ch12 §07, Ch17 §09).

---

## Summary

- Reflection inspects/invokes types at runtime via **`Type`** + the **`MemberInfo`** hierarchy; `Activator` creates instances; attributes are runtime-read metadata.
- It's **correct but slow** (metadata lookup + dynamic dispatch + boxing) — **cache `MemberInfo`, compile to delegates, or use source generators** for hot paths.
- Reflection by string defeats trimming/AOT — annotate or (preferably) replace with **source generators** (the modern, AOT-safe direction).
- Use it for one-time setup, plugins, and truly dynamic needs — not in hot loops or where a dispatch table suffices.
- Full mechanics: CSharpBook Chapter 12; runtime/metadata internals: [Ch01 §07](../01-Runtime/07-Metadata.md).

→ Next: [10-Net.md](10-Net.md)
