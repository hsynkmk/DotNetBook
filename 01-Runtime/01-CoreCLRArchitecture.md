# CoreCLR Architecture

## What the runtime is

CoreCLR is the engine that turns your compiled IL into running code and keeps it alive. When you launch a .NET app, a small native host (`dotnet` / your `apphost`) bootstraps the runtime, which then loads your assembly, finds `Main`, JIT-compiles it, and runs it — providing memory management, type loading, exception handling, threading, and interop along the way.

```
Your process
┌──────────────────────────────────────────────────────────┐
│  apphost / dotnet (native bootstrap)                        │
│        ↓ hostfxr → hostpolicy resolve runtime + deps        │
│  ┌────────────────────────────────────────────────────┐    │
│  │  CoreCLR (coreclr.dll / libcoreclr.so)              │    │
│  │  ┌───────────┐ ┌──────────┐ ┌──────────┐           │    │
│  │  │ Type      │ │  JIT      │ │   GC      │           │    │
│  │  │ loader    │ │ (RyuJIT)  │ │           │           │    │
│  │  └───────────┘ └──────────┘ └──────────┘           │    │
│  │  ┌───────────┐ ┌──────────┐ ┌──────────┐           │    │
│  │  │ Exception │ │ Threading │ │ Interop / │           │    │
│  │  │ handling  │ │ + threadpl│ │ P/Invoke  │           │    │
│  │  └───────────┘ └──────────┘ └──────────┘           │    │
│  └────────────────────────────────────────────────────┘    │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Your assemblies + BCL (managed IL)                 │    │
│  └────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

This file is the map; the rest of the chapter zooms into each component.

---

## Startup sequence

What happens between double-clicking your app and `Main` running:

1. **apphost** — the native launcher (your `MyApp.exe`/`MyApp`) starts. It finds the runtime.
2. **hostfxr / hostpolicy** — resolve which runtime version to use (from `MyApp.runtimeconfig.json`) and which dependencies (`MyApp.deps.json`), locating shared frameworks or self-contained copies.
3. **CoreCLR initializes** — sets up the GC heap, thread pool, type system.
4. **Assembly load** — your entry assembly is loaded; its metadata is read.
5. **JIT `Main`** — the entry point's IL is compiled to native code.
6. **Run** — `Main` executes; methods are JIT-compiled on first call as the program runs.

The `runtimeconfig.json` (generated from your `.csproj`) controls runtime knobs: target framework version, GC mode, thread pool settings, feature switches. Worth knowing it exists when you need to tune behavior.

---

## The components and how they cooperate

### Type loader

Loads types on demand from assembly metadata. When code first references a type, the loader builds its **method table** (the runtime representation: layout, vtable, static fields, type metadata). Generics are instantiated here. Detailed in [05-TypeSystem.md](05-TypeSystem.md).

### JIT compiler (RyuJIT)

Compiles IL → native machine code, **method by method, on first call**. Uses tiered compilation (quick Tier 0, then optimized Tier 1 for hot methods) and Dynamic PGO. Detailed in [02-JIT.md](02-JIT.md). (Native AOT replaces the JIT with build-time compilation — [03-NativeAOT.md](03-NativeAOT.md).)

### Garbage collector

Manages the managed heap: allocates objects, tracks references, reclaims unreachable memory across generations. It cooperates tightly with the JIT (which emits **GC info** describing where object references live in each method) and threads (which must reach **safepoints** for collection). Detailed in [04-GCDeepDive.md](04-GCDeepDive.md).

### Exception handling

Implements try/catch/finally over the platform's native exception machinery, walking the managed stack to find handlers and running `finally` blocks during unwind. Two-pass model (find handler, then unwind).

### Threading and the thread pool

Manages threads and the **thread pool** that powers `Task`, `async`/`await`, timers, and I/O callbacks. The pool auto-tunes its worker count via a hill-climbing algorithm and uses work-stealing queues. Detailed in [08-Threading.md](08-Threading.md).

### Interop / P/Invoke

Bridges managed and native code: generating marshaling stubs, transitioning the GC mode across the boundary, and pinning objects. Detailed in [09-PInvokeInternals.md](09-PInvokeInternals.md).

---

## The cooperative contract: GC safepoints

The components aren't independent — they cooperate through shared contracts. The most important: **the GC can't move/collect objects while a thread is mid-operation on them.** So threads run in **cooperative mode** and periodically hit **safepoints** (method calls, loop back-edges, allocations) where they can be suspended for GC.

- The **JIT** emits **GC info** for every method: at each safepoint, which registers/stack slots hold live object references. This lets the GC find and update roots when it moves objects (compaction).
- When the GC needs to run, it requests all managed threads to **suspend at their next safepoint** (a "GC suspension"). Once all are parked, it collects, then resumes them.
- Threads doing P/Invoke run in **preemptive mode** (the GC doesn't need to suspend them because they're not touching managed objects), which is why the P/Invoke boundary has a GC-mode transition (see [09-PInvokeInternals.md](09-PInvokeInternals.md)).

This JIT↔GC↔threading cooperation is the backbone of managed execution. You rarely see it, but it explains why GC pauses exist and why tight, allocation-free loops without safepoints can briefly delay a collection.

---

## Self-contained vs framework-dependent (runtime resolution)

How CoreCLR is found at startup:
- **Framework-dependent** — `runtimeconfig.json` names a shared framework (`Microsoft.NETCore.App` / `Microsoft.AspNetCore.App`); hostfxr locates the installed copy.
- **Self-contained** — the runtime ships alongside the app; hostpolicy loads the local copy.
- **Native AOT** — there's no separate CoreCLR; the runtime (GC, type system) is statically linked into the single native binary.

`deps.json` lists the app's assemblies and their locations so the loader can resolve them. These two JSON files are the contract between the host and the runtime.

---

## Where to observe it

The runtime is introspectable with the diagnostic tools (also covered in CSharpBook Ch15 and DotNetBook [Ch21](../21-Performance/README.md)):

```bash
dotnet-dump collect -p <pid>     # capture process state
dotnet-dump analyze <dump>
  > clrstack       # managed call stack
  > clrthreads     # all managed threads
  > dumpheap -stat # heap by type
  > eeheap -gc     # GC heap segments/generations
  > dumpmt <addr>  # a method table (type loader output)
```

These SOS commands expose the very structures described in this chapter — method tables, GC heap, threads. The Coding exercises put them to use.

---

## Summary

- CoreCLR is the execution engine: a native host (apphost → hostfxr/hostpolicy) bootstraps it using `runtimeconfig.json`/`deps.json`, then it loads your assembly and runs `Main`.
- Core components: **type loader** (method tables), **JIT** (IL→native), **GC** (memory), **exception handling**, **threading/thread pool**, **interop**.
- They cooperate via **GC safepoints**: the JIT emits GC info, threads suspend at safepoints, the GC collects/compacts and updates roots — the foundation of managed execution.
- Runtime is resolved as framework-dependent, self-contained, or statically linked (Native AOT).
- Inspect it all with `dotnet-dump` + SOS commands.
- The next files deep-dive each component, starting with the JIT.

→ Next: [02-JIT.md](02-JIT.md)
