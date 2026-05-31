# The Type System (Method Tables, vtables, Generics)

## What it is

Every object on the heap carries, at its very start, a pointer to a **method table** (MT) — the runtime's representation of its type. The MT is how the runtime knows an object's size, how to dispatch its virtual methods, where its static fields live, and how to find its metadata. Understanding the MT demystifies virtual dispatch, generics, casts, and why `object` has a header.

```
Heap object layout (reference type):
┌──────────────────┬──────────────────┬─────────────────────┐
│ object header     │ method table ptr  │ instance fields...   │
│ (sync block /     │ (→ MethodTable)   │                      │
│  lock + hashcode) │                   │                      │
└──────────────────┴──────────────────┴─────────────────────┘
        8 bytes            8 bytes (x64)
```

That two-word header (on x64: 8-byte sync-block index + 8-byte MT pointer = 16 bytes) is why even an empty `object` costs 16+ bytes, and why value types boxed onto the heap gain that overhead.

---

## The method table

The MT is the type loader's output (built on first use of a type — see [01-CoreCLRArchitecture.md](01-CoreCLRArchitecture.md)). It contains:
- **Instance size** and GC layout (where reference fields are, for the GC to trace).
- The **vtable** — an array of pointers to the type's virtual method implementations.
- A pointer to **EEClass** (rarely-accessed type metadata: field/method lists, etc. — split out so the hot MT stays cache-friendly).
- The **base type** MT and implemented **interface** map.
- **Static field** storage location.
- Module/metadata references.

You can dump it:
```
dumpmt -md <MethodTable address>   # in dotnet-dump / SOS — lists methods, slots, EEClass
dumpobj <object address>           # shows an object's MT pointer + fields
```

---

## Virtual dispatch via the vtable

A non-virtual call jumps to a fixed address — the compiler/JIT knows the target. A **virtual** call must pick the implementation based on the object's *runtime* type. The MT's **vtable** makes this O(1):

```
Animal a = new Dog();
a.Speak();    // virtual

// At runtime:
//  1. load a's method table pointer (from the object header)
//  2. index into the vtable at Speak's fixed slot
//  3. call through that pointer  → Dog.Speak
```

Each virtual method has a fixed **slot index** assigned at type load. An override reuses the base method's slot, pointing it at the derived implementation. So dispatch is just "load MT, index slot, call" — a couple of indirections.

**Interface dispatch** is slightly more involved (interfaces don't share a single contiguous slot layout across all implementers): the runtime uses an **interface dispatch map** plus a per-call-site cache (the "dispatch stub") that memoizes the resolved target. Dynamic PGO and **devirtualization** ([02-JIT.md](02-JIT.md)) often eliminate this entirely by proving the concrete type and inlining.

---

## Value types: no MT pointer when unboxed

A **value type** stored inline (a local, a field, an array element) has **no header and no MT pointer** — it's just its raw fields. That's why `int` is 4 bytes, not 16, and why a `struct[]` is densely packed.

But when you **box** a value type (assign to `object`/an interface), the runtime allocates a heap object *with* the header + MT pointer and copies the value in:

```csharp
int x = 42;          // 4 bytes on the stack, no MT
object o = x;        // BOXING: heap alloc, header + MT(Int32) + the 42
int y = (int)o;      // UNBOXING: read the value back out
```

Boxing is a heap allocation + copy — a classic hidden cost. Generics avoid it (the JIT specializes value-type instantiations — below). See CSharpBook Ch03 §07.

---

## Generics — sharing and specialization

A generic type/method is a *template*; the runtime creates a concrete **instantiation** per type argument. The clever part is how the JIT shares code:

- **Reference-type arguments share one instantiation.** `List<string>`, `List<object>`, `List<Customer>` all use the **same** JIT-compiled code (a `T` that's a reference is just a pointer regardless of which reference type). The MT differs per instantiation, but the native code is shared. This keeps code size down.
- **Value-type arguments get specialized code.** `List<int>`, `List<double>`, `List<MyStruct>` each get their **own** JIT-compiled, type-specific native code — because their sizes and layouts differ. This is what makes generic collections of value types **allocation-free and as fast as hand-written** (no boxing): `List<int>` stores raw `int`s inline.

```csharp
List<int> ints = [];        // specialized: stores 4-byte ints inline, no boxing
List<string> strs = [];     // shared code with List<object> etc. (T is a pointer)
```

This shared-vs-specialized model is why .NET generics give you **both** type safety and value-type performance — unlike Java's type-erased generics (which box) or C++ templates (which specialize everything, bloating code). Generic math (`INumber<T>`, CSharpBook Ch04) leans on value-type specialization for zero-overhead numeric generics.

`MakeGenericType`/`MakeGenericMethod` create instantiations at runtime via reflection — which is why Native AOT (no JIT) struggles with instantiations it didn't see at build ([03-NativeAOT.md](03-NativeAOT.md)).

---

## Casts and type checks

`is`, `as`, and explicit casts consult the MT:
- **Exact type check** (`o.GetType() == typeof(T)`) — compare MT pointers (one load + compare).
- **`is`/`as`/cast to a base or interface** — walk the MT's base-type chain or check its interface map; cached for speed.

```csharp
if (animal is Dog d) { ... }    // checks animal's MT against Dog's (and derived)
```

Sealed types let the JIT turn some checks into a single MT-pointer comparison (no chain walk) and enable devirtualization — one reason `sealed` can help hot paths.

---

## Statics

Static fields aren't in instances — they live in storage referenced by the MT (per closed generic type: `Counter<int>` and `Counter<string>` have *separate* statics). Static constructors (`static Foo() {}`) run lazily, exactly once, before first use — the runtime guards this with a flag checked on access (a tiny cost the JIT often optimizes after first run).

---

## Common gotchas

### Boxing in disguise

Passing a struct to an `object`/non-generic-interface parameter, putting it in a non-generic collection, or string-formatting it can box. Use generics and `Span<T>` to stay unboxed. (CSharpBook Ch03 §07, Ch17 §03.)

### Interface calls on structs box (sometimes)

Calling an interface method on a struct **through the interface** boxes it; calling it directly (or via a generic constrained to the interface) doesn't. `where T : IComparable<T>` keeps it unboxed.

### Assuming `List<int>` boxes

It doesn't — value-type generic instantiations are specialized and store values inline. (The old non-generic `ArrayList` *did* box; that's why it's obsolete.)

### Object header overhead for tiny objects

Millions of tiny reference objects each pay the 16-byte header. For dense data, prefer `struct`s/arrays of structs.

---

## Summary

- Every reference object starts with an **object header + method table pointer**; the MT defines size, GC layout, the **vtable**, base/interface maps, and statics.
- **Virtual dispatch** = load MT, index a fixed **vtable slot**, call; **interface dispatch** uses a dispatch map + cached stub. PGO/devirtualization often inline these away.
- **Value types unboxed** carry no header/MT (raw fields, dense, fast); **boxing** wraps them in a heap object with a header — a hidden alloc + copy.
- **Generics**: reference-type args **share** one native code body; value-type args get **specialized** code — giving type safety *and* allocation-free value-type performance.
- Casts/`is`/`as` consult the MT (pointer compare or base/interface walk); `sealed` enables faster checks and devirtualization.
- Inspect with `dumpmt`/`dumpobj` in `dotnet-dump`.

→ Next: [06-Assemblies.md](06-Assemblies.md)
