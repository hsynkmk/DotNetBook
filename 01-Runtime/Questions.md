# Chapter 01 — Runtime & CLR — Q & A

---

### Q1. What happens between launching an app and `Main` running?

The native **apphost** starts → **hostfxr/hostpolicy** resolve the runtime version and dependencies (from `runtimeconfig.json`/`deps.json`) → **CoreCLR initializes** (GC heap, thread pool, type system) → your assembly loads → the JIT compiles `Main` → it runs, with further methods JIT-compiled on first call.

---

### Q2. What is a GC safepoint and why does it exist?

A point in JIT'd code (method calls, loop back-edges, allocations) where a thread can be safely suspended for GC. The GC can't move objects while a thread is mid-operation on them, so threads run in **cooperative mode** and park at safepoints when the GC needs to collect. The JIT emits **GC info** describing live references at each safepoint.

---

### Q3. Why does .NET use a JIT instead of always compiling ahead of time?

Two wins: **portability** (IL is CPU-agnostic; the JIT targets the actual machine and its instruction sets) and **runtime knowledge** (the JIT sees which branches/types are hot and optimizes accordingly via PGO). The cost is startup warmup, which Native AOT avoids at the expense of runtime adaptation.

---

### Q4. What is tiered compilation?

Methods are first compiled at **Tier 0** (fast to compile, minimal optimization → quick startup), and hot methods (by call count) are recompiled at **Tier 1** (fully optimized → peak throughput). It gives both fast startup and fast steady state.

---

### Q5. What problem does On-Stack Replacement (OSR) solve?

A method with a long-running loop called only once would be stuck in slow Tier 0 forever (it's never *re-called* to trigger Tier 1). OSR swaps the running method's Tier 0 frame for optimized Tier 1 code **mid-loop**, without restarting it.

---

### Q6. What is Dynamic PGO and what does it enable?

Profile-Guided Optimization: the JIT instruments Tier 0 to record runtime behavior (hot branches, concrete types at virtual call sites), then uses that profile at Tier 1. It enables **devirtualization** (inlining the common concrete type behind a type check), hot/cold block splitting, and better inlining. Default since .NET 8.

---

### Q7. What is ReadyToRun and how does it relate to the JIT?

R2R pre-compiles assemblies to native code at **publish** time, so methods run from precompiled code at startup (no Tier 0 cost) — faster cold start. The IL is still shipped, and the JIT can re-optimize hot methods to Tier 1 at runtime (with PGO). It's a middle ground between pure JIT and Native AOT.

---

### Q8. Why can't Native AOT use `Reflection.Emit` or `Expression.Compile`?

There's no JIT in the AOT binary, and those features **generate and execute new code at runtime** (which needs a JIT). They throw `PlatformNotSupportedException`. The fix is source generators (compile-time codegen). Flagged by `IL3050`.

---

### Q9. Why does Native AOT require trimming and break some reflection?

AOT does **whole-program analysis**, keeping only provably-reachable code. Reflection by runtime string isn't traceable, so those types/members may be trimmed → null/missing at runtime (`IL2xxx` warnings). Annotate with `[DynamicallyAccessedMembers]` or use source generators.

---

### Q10. What stays the same under Native AOT vs JIT?

The **GC, type system (method tables, vtables), BCL, async/await, exceptions, and threading** are unchanged — most code just works. Only runtime codegen and string-based reflection break.

---

### Q11. Explain the generational hypothesis.

Most objects die young (temporaries, DTOs, iterators), while a few live long (caches, singletons). So the GC partitions the heap into generations (0/1/2) and collects the young ones often and cheaply, the old ones rarely and expensively — minimizing total GC work.

---

### Q12. Walk through a GC collection.

**Suspend** threads at safepoints → **mark** by tracing from roots (statics, stack/registers, GC handles) to find live objects → **compact** the collected generation (move survivors together, update all references via GC info) → **resume**. Survivors are promoted to the next generation; allocation afterward is a cheap pointer bump.

---

### Q13. What are GC roots?

References that keep objects alive: static fields, stack locals/arguments and registers of running methods, and GC handles (incl. the finalization queue) — plus everything transitively reachable from them. Unintended roots (static events, never-evicting caches, captured closures) cause leaks.

---

### Q14. Workstation vs Server GC?

**Workstation**: one heap, collects mostly on the app thread, tuned for low latency/memory (client apps). **Server**: one heap per logical core, dedicated parallel GC threads, tuned for throughput (servers). ASP.NET Core defaults to Server GC.

---

### Q15. What is background (concurrent) GC?

It performs most of the expensive Gen 2 marking **concurrently** with the application running, shrinking the stop-the-world pause. On by default; the reason modern .NET avoids long GC pauses.

---

### Q16. What is DATAS?

Dynamically Adapting To Application Sizes — modern Server GC sizes the heap to the app's **actual working set** (using dynamic regions) instead of core count, cutting memory 20–40% for small/medium apps. Default in modern .NET; disable only if a large-allocator workload measures worse.

---

### Q17. What are write barriers and card tables for?

A young (Gen 0) collection only scans Gen 0, but an old (Gen 2) object may reference a young object. On every reference-field write, the JIT emits a **write barrier** that records the modified region in a **card table**; the young collection also scans marked cards so it doesn't miss those references. Reference writes thus carry a small cost.

---

### Q18. What's in an object header, and why does an empty object cost 16 bytes?

On x64: an 8-byte sync-block index (lock/hashcode) + an 8-byte **method table pointer** = 16 bytes before any fields. Every reference object pays this, which is why millions of tiny objects are wasteful and dense data prefers structs/arrays.

---

### Q19. How does virtual dispatch work?

The object's method-table pointer leads to the **vtable**; each virtual method has a fixed **slot index**; a call loads the MT, indexes the slot, and calls through the pointer. Overrides reuse the base slot pointing at the derived method. Interface dispatch uses a dispatch map + cached stub; PGO often devirtualizes both.

---

### Q20. How do generics share vs specialize code?

**Reference-type** instantiations (`List<string>`, `List<object>`) **share** one JIT-compiled body (a reference is just a pointer). **Value-type** instantiations (`List<int>`, `List<MyStruct>`) get **specialized** native code with inline layout — giving type safety *and* allocation-free value-type performance (no boxing).

---

### Q21. What is an assembly and what's in it?

The unit of deployment/versioning: a PE file containing **IL**, **metadata**, and a **manifest** (name, version, culture, public key, referenced assemblies). Identity = name+version+culture+optional public key token. Assemblies load lazily on first type reference.

---

### Q22. What replaced AppDomains, and what can it do?

**`AssemblyLoadContext`**. It loads assemblies into isolated contexts that can hold **multiple versions** of the same assembly, support custom resolution, and (when `isCollectible`) be **unloaded** — enabling plugin reload. AppDomains don't exist in modern .NET.

---

### Q23. What's the #1 gotcha when building plugins with AssemblyLoadContext?

The **shared contract** (e.g., `IPlugin`) must load in the **Default** context so host and plugin see the *same* type — otherwise the cast fails (two `IPlugin` types). Return `null` from the ALC's `Load` for shared contracts so they fall back to Default.

---

### Q24. Why might `Unload()` not free a plugin?

It's a request, completed only when **no references** to the context's assemblies/types remain. A lingering host field, cached `Type`/delegate, event subscription, or running thread roots the context. Verify collection with a `WeakReference`.

---

### Q25. What is metadata and why does it make .NET self-describing?

Relational tables (`TypeDef`, `MethodDef`, `Field`, `CustomAttribute`, …) in each assembly describing every type/member, addressed by 4-byte **tokens** that IL references. The runtime reads metadata to load types and dispatch — no separate headers needed — which is what enables reflection, the debugger, and cross-language interop.

---

### Q26. Why is reflection slow, and how do you avoid the cost?

Each operation is a metadata table lookup + signature decode + dynamic dispatch (hundreds of ns) vs a direct call's single jump. Cache `MemberInfo`, compile to delegates (`Expression`/`Delegate.CreateDelegate`), or use **source generators** to read metadata at compile time and emit direct code (also AOT-safe).

---

### Q27. Why does blocking a thread-pool thread cause starvation?

The pool injects new threads slowly (hill climbing, ~1–2/sec when starved). Blocking workers (`.Result`, sync I/O) removes them from a small shared pool while work queues up; throughput collapses until threads are slowly added. Hence "never block on async."

---

### Q28. Why does async I/O scale to thousands of connections on few threads?

`await`ing I/O registers the operation with the OS (IOCP/epoll/kqueue) and **returns the thread to the pool** — no thread is parked waiting. When the OS signals completion, the pool runs the brief continuation. N concurrent I/Os need only enough threads to run continuations, not N parked threads.

---

### Q29. What is work-stealing in the thread pool?

Work submitted by a pool thread goes to its **local LIFO queue** (cache-friendly); external work goes to the **global queue**. An idle worker **steals** from the tail of another worker's local queue (or the global queue), balancing load across cores without central coordination.

---

### Q30. What does the GC-mode transition in P/Invoke do, and why?

Before calling native code, the marshaling stub switches the thread from **cooperative** to **preemptive** mode, so the GC can collect without waiting for the thread (native code never reaches a managed safepoint). It switches back on return. This ~tens-of-ns transition is P/Invoke's fixed cost; `SuppressGCTransition` skips it for trivial, fast, non-blocking calls only.

---

### Q31. Blittable vs non-blittable types in P/Invoke?

**Blittable** types have identical managed/native representations (primitives, pointers, blittable structs) and pass with no conversion (fast path, pinned). **Non-blittable** types (`string`, `bool`, classes) require conversion + allocation/copy per call. Design blittable signatures for hot interop paths.

---

### Q32. `[LibraryImport]` vs `[DllImport]` at the runtime level?

`[LibraryImport]` generates the marshaling stub at **compile time** (a source generator → real C#), so it's AOT/trim-safe. `[DllImport]` generates the stub at **runtime** via the JIT (an IL stub), which doesn't fit Native AOT well. Prefer `[LibraryImport]` for new code.

---

→ Next: [Coding.md](Coding.md)
