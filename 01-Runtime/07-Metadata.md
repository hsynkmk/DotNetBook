# Metadata & Reflection Internals

## What metadata is

Every assembly carries **metadata** — structured tables describing every type, method, field, parameter, and attribute it contains. This is what makes .NET self-describing: the runtime doesn't need separate header files to know an assembly's shape; it reads the metadata. Reflection, the debugger, IntelliSense, serializers, and the type loader all read these same tables.

```
Assembly = IL (the code)  +  Metadata (the description of the code)
```

This pairing is fundamental: native DLLs export symbols by name with no rich type info, but a .NET assembly fully describes its types — enabling reflection, late binding, and cross-language interop without extra files.

---

## The metadata tables

Metadata is a set of relational-style **tables** in the PE file. The main ones:

| Table | Holds |
|---|---|
| `TypeDef` | Types **defined** in this assembly |
| `TypeRef` | Types **referenced** from other assemblies |
| `MethodDef` | Methods defined here (with a pointer to their IL) |
| `Field` | Fields |
| `Param` | Parameters |
| `MemberRef` | References to members in other assemblies |
| `AssemblyRef` | Referenced assemblies (the dependency list) |
| `CustomAttribute` | Attributes applied to elements |
| `TypeSpec` / `MethodSpec` | Generic instantiations |

Each row has a **token** — a 4-byte handle (top byte = table id, lower 3 = row index) that uniquely identifies a metadata element. IL instructions reference metadata by token (e.g., `call 0x0A000003` means "call the member at MemberRef row 3").

```
Token 0x02000002  →  TypeDef table, row 2  →  e.g., "class OrderService"
Token 0x06000010  →  MethodDef table, row 16 → e.g., "OrderService.Place(...)"
```

---

## Signatures

A method's parameter and return types aren't stored as text — they're **binary signatures**: compact blobs encoding the type shape (e.g., "instance method, returns `Task<int>`, takes a `string` and an `int`"). The same blob encoding describes field types, property types, and generic constraints. Tools decode signatures to reconstruct the human-readable method shape you see in a decompiler.

This binary encoding keeps metadata compact and fast to parse — the type loader reads signatures to build method tables ([05-TypeSystem.md](05-TypeSystem.md)).

---

## Attributes are metadata

Custom attributes ([CSharpBook Ch12 §03]) are stored in the `CustomAttribute` table: each row links a *target* element (a type/method/etc. token) to the attribute's constructor (a token) and a blob of the constructor arguments + named property values.

```csharp
[Obsolete("Use NewApi")]
public void OldApi() { }
```
becomes a `CustomAttribute` row: target = `OldApi`'s MethodDef token, ctor = `ObsoleteAttribute(string)`'s token, blob = `"Use NewApi"`. At runtime, `GetCustomAttribute<ObsoleteAttribute>()` reads this row, **constructs** the attribute, and returns it — which is why attribute args must be compile-time constants (they're baked into the blob) and why each `GetCustomAttribute` call instantiates a fresh attribute.

---

## How reflection reads metadata

Reflection (`System.Reflection`) is, at bottom, a **metadata reader plus the type loader**:

```csharp
Type t = typeof(OrderService);            // resolves the TypeDef token → MethodTable
MethodInfo[] methods = t.GetMethods();    // reads the MethodDef rows for this type
ParameterInfo[] ps = methods[0].GetParameters();  // decodes the method's signature blob
var attr = t.GetCustomAttribute<MyAttr>(); // reads CustomAttribute rows
object inst = Activator.CreateInstance(t); // finds a .ctor MethodDef, allocates, runs it
methods[0].Invoke(inst, args);            // sets up a call frame and dispatches
```

So `typeof`/`GetType` map to method tables; `GetMethods`/`GetFields`/`GetCustomAttributes` enumerate metadata tables; `Invoke` builds a managed call. This is why reflection is **correct but slow** — it's table lookups + signature decoding + dynamic invocation overhead per call, versus a direct call's single jump. (Reflection performance and how to cache/compile it away: CSharpBook Ch12 §07.)

---

## `typeof` / `nameof` / tokens at the IL level

- `typeof(T)` compiles to `ldtoken` (push the type's metadata token) + `Type.GetTypeFromHandle` — cheap, resolved from a token, no name lookup.
- `nameof(x)` is **pure compile-time** — the compiler emits a string literal; no metadata read at runtime.
- The `metadata token` of a member is accessible via `MemberInfo.MetadataToken` — occasionally useful for tooling that correlates IL and metadata.

(These compile-time reflection features: CSharpBook Ch12 §06.)

---

## Reading metadata without running code

Tools (analyzers, linkers, doc generators) often need metadata **without executing or fully loading** an assembly:
- **`System.Reflection.Metadata`** — a low-level, high-performance, read-only metadata reader (used by the SDK, ILLink/trimmer, and source-gen tooling). It parses the tables directly without the type loader.
- **`MetadataLoadContext`** — a reflection API surface over assemblies loaded for inspection only (no execution) — see [06-Assemblies.md](06-Assemblies.md).

These power the trimmer's reachability analysis, the AOT compiler's whole-program walk ([03-NativeAOT.md](03-NativeAOT.md)), and Roslyn analyzers — all of which reason about code via metadata, not by running it.

---

## Why this matters in practice

- **Reflection cost** — every reflective operation is a metadata read + dynamic dispatch. Cache `MemberInfo`, and compile to delegates (`Expression`/`Delegate.CreateDelegate`) or use **source generators** for hot paths (CSharpBook Ch12 §05, §07).
- **Trimming/AOT** — the trimmer removes metadata for unreachable types/members; reflection by runtime string can't be traced, so types get trimmed (the `IL2xxx` warnings). Source generators read metadata at **compile time** and emit direct code — the AOT-safe alternative.
- **Serialization** — reflection-based serializers (old STJ, Newtonsoft) walk metadata at runtime; source-gen serializers bake the metadata-derived code at build time.

The throughline: **metadata is what makes .NET introspectable**, and the modern trend is to read it at *compile time* (source generators) rather than *runtime* (reflection) for speed and AOT-compatibility.

---

## Common gotchas

### Assuming reflection is cheap

Each `GetMethod`/`GetCustomAttribute`/`Invoke` is metadata lookup + dynamic dispatch (~hundreds of ns). Cache results; compile or source-gen hot paths.

### Attribute args must be constants

They're stored as a blob in metadata at compile time — only compile-time constants (primitives, strings, `typeof`, enums, arrays of those) are allowed.

### Reflecting on trimmed members

Under trimming/AOT, a member reached only via string-based reflection may have been removed → `null`/missing-member at runtime. Annotate (`[DynamicallyAccessedMembers]`) or use source generators.

### Confusing IL tokens with hash codes

A metadata token identifies a row in *this* assembly's tables; it's not stable across builds or a hash. Don't persist tokens across compilations.

---

## Summary

- Every assembly pairs **IL** with **metadata** — relational tables (`TypeDef`, `MethodDef`, `Field`, `CustomAttribute`, …) describing its types, addressed by 4-byte **tokens** that IL instructions reference.
- Type shapes are stored as compact **binary signatures**; attributes live in the `CustomAttribute` table (hence compile-time-constant args).
- **Reflection** is a metadata reader + type loader + dynamic invoker — correct but slow (cache it, or use source generators).
- `typeof` → `ldtoken` (cheap); `nameof` → compile-time string (free).
- **`System.Reflection.Metadata`** / **`MetadataLoadContext`** read metadata without executing code — powering the trimmer, AOT compiler, and analyzers.
- Modern best practice: read metadata at **compile time** (source generators) rather than runtime (reflection) for performance and AOT-safety.

→ Next: [08-Threading.md](08-Threading.md)
