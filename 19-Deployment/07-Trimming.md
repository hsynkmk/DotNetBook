# Trimming

## Removing unused code

**Trimming** (the IL trimmer / "linker") removes assemblies, types, and members your app **doesn't use** from a self-contained publish, shrinking the output — sometimes dramatically. It's enabled with `PublishTrimmed=true` and is **required by Native AOT** ([06-NativeAOT.md](06-NativeAOT.md)). The trimmer works by **static analysis**: starting from your app's entry point, it follows references to determine what's reachable and discards the rest. The fundamental tension: **reflection breaks static analysis** — code reached only via reflection looks "unused" to the trimmer and gets removed, causing runtime failures. Understanding and resolving this is the whole art of trimming.

```bash
dotnet publish -c Release -r linux-x64 --self-contained -p:PublishTrimmed=true
```

---

## How the trimmer decides what to keep

The trimmer builds a reachability graph from the entry point: a method that's called keeps its callees; a type that's instantiated keeps its used members. Anything not reachable through this **static** analysis is removed. This works perfectly for code with explicit references — but **reflection** (`Type.GetType("...")`, `Activator.CreateInstance`, accessing members by name) creates references the trimmer **can't see**, so the targets appear unreachable and are trimmed:

```csharp
// The trimmer can't tell that MyPlugin is used — it may be trimmed away:
var t = Type.GetType("MyApp.MyPlugin");
var instance = Activator.CreateInstance(t!);   // runtime: type not found / members missing
```

The result is a runtime `MissingMethodException`, `TypeLoadException`, or a null type — bugs that only appear in the **trimmed** build, not in development.

---

## Trim warnings — your early-warning system

The compiler/publish emits **trim warnings** (`IL2xxx`) for patterns the trimmer can't analyze safely — reflection over possibly-trimmed members, calls into trim-unsafe APIs, etc. **These warnings predict runtime failures.** A "trim-safe" app resolves all of them:

```
IL2075: 'this' argument does not satisfy 'DynamicallyAccessedMemberTypes' in call to ...
IL2026: Using member 'X' which has 'RequiresUnreferencedCodeAttribute' ...
```

Treat trim warnings seriously (ideally as errors for trimmed/AOT builds). Each one is a place where the trimmer might remove something you actually need at runtime.

---

## Annotations to guide the trimmer

When you *must* use reflection, annotations tell the trimmer what to preserve:

- **`[DynamicallyAccessedMembers]`** — declares which members of a type are accessed via reflection, so the trimmer keeps them:

```csharp
void Create([DynamicallyAccessedMembers(DynamicallyAccessedMemberTypes.PublicConstructors)] Type t)
    => Activator.CreateInstance(t);   // trimmer now preserves t's public constructors
```

- **`[DynamicDependency]`** — explicitly forces a specific member/type to be kept:

```csharp
[DynamicDependency(DynamicallyAccessedMemberTypes.All, typeof(MyPlugin))]
void EnsureKept() { }   // guarantees MyPlugin (and all its members) survive trimming
```

- **`[RequiresUnreferencedCode]`** — marks *your* API as trim-unsafe, propagating a warning to callers (so library authors flag reflection-using methods).

These let you keep necessary reflection while making the trimmer aware of it — the bridge between dynamic code and static analysis.

---

## The real fix: source generators

Annotations preserve reflection, but the **better** answer is to avoid runtime reflection entirely via **source generators** that emit the needed code at compile time — inherently trim/AOT-safe ([06-NativeAOT.md](06-NativeAOT.md)):

- **System.Text.Json source generation** instead of reflection-based serialization ([Ch02 §05](../02-BCL/05-Serialization.md)).
- **Configuration binding source generator** instead of reflection-based `Bind` ([Ch13](../13-Configuration/README.md)).
- **Minimal API / Regex / LoggerMessage** source generators.

Source-generated code is statically analyzable (the trimmer sees the references), so it needs no annotations and survives trimming/AOT cleanly. The modern .NET trajectory is toward source generators precisely to make trimming/AOT viable.

---

## Trimming granularity and libraries

- **Trim mode**: modern .NET trims at the **member** level (more aggressive, smaller) — earlier modes trimmed whole assemblies. Smaller output, but more reliance on correct annotations.
- **Trimmable libraries**: a library marks itself `<IsTrimmable>true</IsTrimmable>` to declare it's been analyzed/annotated for trimming. Libraries that aren't trim-compatible (heavy reflection, no annotations) produce warnings and risk breakage when trimmed — check that your dependencies support trimming before relying on it.

---

## Common gotchas

### Reflection silently trimmed

Code reached only via reflection is removed, failing at runtime (not at build/dev). Annotate (`[DynamicallyAccessedMembers]`/`[DynamicDependency]`) or use source generators — and **test the trimmed build**.

### Ignoring trim warnings

`IL2xxx` warnings predict the exact runtime failures trimming will cause. Don't suppress them blindly — resolve each (annotate or refactor). Treat them as errors for trimmed/AOT builds.

### Non-trimmable dependencies

A library full of unannotated reflection breaks when trimmed. Verify dependencies declare `IsTrimmable`/AOT support; if not, you may not be able to trim safely.

### Testing only the untrimmed build

Trimming bugs appear only in the trimmed artifact. Always **run the trimmed/published output** in a representative environment, not just `dotnet run`.

### Over-relying on `[DynamicDependency]`

Keeping everything via annotations defeats trimming's size benefit. Prefer **source generators** (no reflection to preserve) over annotating large reflection surfaces.

---

## Summary

- **Trimming** (`PublishTrimmed`, required by AOT) removes **statically-unreachable** code to shrink self-contained output; it works by **static reachability analysis** from the entry point.
- The core hazard: **reflection is invisible to static analysis**, so reflection-reached code gets trimmed and **fails at runtime** (not in dev) — `MissingMethod`/`TypeLoad` errors in the trimmed build.
- **Trim warnings (`IL2xxx`) predict these failures** — resolve them (treat as errors for trimmed/AOT). Use **`[DynamicallyAccessedMembers]`/`[DynamicDependency]`** to preserve necessary reflection, and **`[RequiresUnreferencedCode]`** to flag trim-unsafe APIs.
- The **better fix is source generators** (System.Text.Json, config binding, Minimal API) that emit statically-analyzable code — inherently trim/AOT-safe, no annotations needed.
- Check dependencies are **`IsTrimmable`**/AOT-ready, and **always test the trimmed published artifact** — trimming bugs don't appear in `dotnet run`.

→ Next: [08-ReadyToRun.md](08-ReadyToRun.md)
