# Native AOT (Runtime Internals)

## What it is, from the runtime's view

Native AOT compiles your **whole program plus a minimal runtime** to a single native executable at **publish** time. There's no IL shipped, no JIT, and no separate CoreCLR to load — the GC, type system, and exception machinery are statically linked into your binary, like a C++ program that happens to have a garbage collector.

```
JIT model:   ship IL  →  CoreCLR loads it  →  JIT compiles per method at runtime
AOT model:   compile everything to native at publish  →  run the native binary directly
```

> The **usage, limitations checklist, and trimming workflow** are covered in CSharpBook Chapter 14. This file focuses on **how the runtime differs** under AOT and *why* the limitations exist.

---

## How the AOT toolchain works

```bash
dotnet publish -r linux-x64 -c Release   # with <PublishAot>true</PublishAot>
```

1. **Whole-program analysis** — the ILC (IL Compiler) starts from your entry point and walks every reachable method/type. Anything unreachable is **trimmed** (removed).
2. **Ahead-of-time codegen** — reachable IL is compiled to native object code (ILC uses the same RyuJIT backend, but at build time).
3. **Static runtime linking** — a cut-down runtime (GC, type system, exception handling, but no JIT) is linked in.
4. **Native link** — the platform linker (clang/MSVC) produces the final executable.

The output is self-contained native machine code for one specific RID. No `runtimeconfig.json` runtime resolution, no IL, no JIT.

---

## Why the limitations exist (the runtime reasons)

The AOT restrictions aren't arbitrary — they follow directly from "no JIT" and "whole-program trimming":

### No runtime code generation → no JIT-dependent features

There's no JIT in the final binary, so anything that **generates and executes new code at runtime** can't work:
- `System.Reflection.Emit` / `DynamicMethod` — emit IL to be JIT'd. No JIT → throws.
- `Expression.Compile()` — compiles an expression tree to a delegate via the JIT. Throws `PlatformNotSupportedException`.
- `dynamic` (the DLR) — generates call sites at runtime.
- Loading and running *new* assemblies (`Assembly.LoadFile` of code not seen at build) — that code was never AOT-compiled.

These are flagged at build with `IL3050` (`RequiresDynamicCode`).

### Whole-program trimming → reflection must be statically visible

The ILC keeps only what it can *prove* is reachable. Reflection over types named by runtime strings is invisible to that analysis:
```csharp
Type? t = Type.GetType(userSuppliedName);   // ILC can't see what 't' will be → that type may be trimmed
Activator.CreateInstance(t!);                 // → null / missing-member at runtime
```
Hence the trimming annotations (`[DynamicallyAccessedMembers]`) and warnings (`IL2xxx`). The fix is **source generators** — compile-time codegen that *is* visible to the analysis (STJ source-gen, `[GeneratedRegex]`, `[LibraryImport]`, `LoggerMessage`).

### Generic instantiation must be known at build

The JIT normally creates generic instantiations on demand (`List<SomeType>` the first time it's used). AOT must generate them at build. `MakeGenericType`/`MakeGenericMethod` over type combinations not seen at compile time can fail at runtime.

---

## What stays the same

Crucially, the **execution model above codegen is unchanged**:
- **Same GC** — generational, with the same heap behavior (though some modes/config differ). Your allocation patterns behave the same.
- **Same type system** — method tables, vtables, virtual dispatch all work (just resolved at build).
- **Same BCL** — collections, LINQ, `Span<T>`, async, networking — all work.
- **Same exceptions, threading, async/await.**

So most code "just works." What breaks is the reflection-heavy, runtime-codegen tail — which modern libraries increasingly avoid via source generators.

---

## The runtime characteristics

| Aspect | JIT (CoreCLR) | Native AOT |
|---|---|---|
| Startup | warmup (Tier 0 → Tier 1) | **instant** (native code from byte 0) |
| Steady-state peak | **best** (PGO re-optimizes) | good (build-time optimization only) |
| Memory footprint | runtime + heap | **smaller** (no JIT, trimmed) |
| Binary | needs runtime / self-contained | **single native file** |
| Reflection/dynamic | full | limited |
| Adapts to runtime profile | yes (PGO) | no (fixed at build) |

The fundamental trade: AOT gives up the JIT's **runtime adaptation** (PGO, OSR, CPU-specific re-JIT) in exchange for **zero warmup and a small self-contained binary**. So AOT wins for **short-lived / startup-sensitive** workloads (CLI, serverless, scale-to-zero containers); the JIT can win **steady-state throughput** on long-running servers.

---

## Diagnostics under AOT

You lose some runtime introspection (no JIT to query, limited reflection), but:
- Build-time **trim/AOT analyzers** (`IL2xxx`/`IL3xxx`) catch incompatibilities *before* you ship — treat them as errors.
- Enable analyzers even on non-AOT builds with `<IsAotCompatible>true</IsAotCompatible>` on libraries, so you catch issues early.

```xml
<PropertyGroup>
  <PublishAot>true</PublishAot>          <!-- apps -->
  <!-- or, for libraries: -->
  <IsAotCompatible>true</IsAotCompatible>
</PropertyGroup>
```

---

## When the runtime team recommends AOT

- CLI tools (instant start, single binary, no install).
- Serverless functions (cold-start cost dominates).
- Microservices/containers that scale to zero or scale out frequently.
- Constrained environments (small images, low memory).

And when **not** to:
- Reflection/`Expression`-heavy frameworks (some ORMs, plugin hosts, `dynamic`).
- Long-running high-throughput servers where JIT+PGO peak throughput matters more than startup.

---

## Summary

- Native AOT compiles your whole program + a JIT-less runtime (GC, type system) to one native binary at publish, via whole-program analysis + trimming + the RyuJIT backend at build time.
- The limitations follow from the model: **no JIT** → no `Reflection.Emit`/`Expression.Compile`/`dynamic` (`IL3050`); **whole-program trimming** → reflection must be statically visible (`IL2xxx`), fixed by source generators.
- The **GC, type system, BCL, async, and exceptions are unchanged** — most code just works.
- Trade-off: gives up JIT runtime adaptation (PGO/OSR) for **instant startup + small self-contained binary** — ideal for CLI/serverless, less so for long-running throughput servers.
- Practical usage and the full trimming workflow: CSharpBook Chapter 14; deployment in [Chapter 19](../19-Deployment/README.md).

→ Next: [04-GCDeepDive.md](04-GCDeepDive.md)
