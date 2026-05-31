# Chapter 01 — Runtime & CLR

> The Common Language Runtime: what loads your assembly, JITs your IL, manages memory, dispatches your virtual calls. Deep internals — pair with CSharpBook Chapter 09 (Memory & Performance) for the language-side complement.

**Prerequisites**: comfortable with C# basics; ideally CSharpBook chapter 09 read.

**Time to read**: ~6-8 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-CoreCLRArchitecture.md](01-CoreCLRArchitecture.md) | The pieces: type loader, JIT, GC, exception handler, threading. How they cooperate. |
| [02-JIT.md](02-JIT.md) | Just-in-time compilation, tiered compilation, OSR, PGO, ReadyToRun, what the JIT can and can't do. |
| [03-NativeAOT.md](03-NativeAOT.md) | Native AOT in detail — when, limits, what it disables, how to make code AOT-friendly. |
| [04-GCDeepDive.md](04-GCDeepDive.md) | Generations, LOH, POH, write barriers, server vs workstation GC, DATAS (.NET 10 default), background GC. |
| [05-TypeSystem.md](05-TypeSystem.md) | Method tables, vtables, generic instantiation, devirtualization, value-type specialization. |
| [06-Assemblies.md](06-Assemblies.md) | Assembly loading, AssemblyLoadContext, lazy load, isolated extensibility. |
| [07-Metadata.md](07-Metadata.md) | Tokens, signatures, attributes, the metadata reader, reflection internals. |
| [08-Threading.md](08-Threading.md) | Thread pool internals, hill climbing, work stealing, IO completion ports, modern Task scheduling. |
| [09-PInvokeInternals.md](09-PInvokeInternals.md) | Marshaling stub generation, LibraryImport, SuppressGCTransition, blittable types. |
| [Questions.md](Questions.md) | ~30 drilling questions. |
| [Coding.md](Coding.md) | Inspect IL, JIT'd code, GC, AssemblyLoadContext, marshaling. |

---

## Learning objectives

After this chapter you should be able to:
- Explain how IL becomes machine code, including tiered compilation.
- Describe GC's generational design, when each collection happens, and what triggers them.
- Read a method table dump (`dotnet-dump`).
- Choose between server/workstation/DATAS GC modes.
- Author Native-AOT-friendly code, knowing what's disabled.
- Use AssemblyLoadContext for plugin isolation.

→ Begin: [01-CoreCLRArchitecture.md](01-CoreCLRArchitecture.md)
