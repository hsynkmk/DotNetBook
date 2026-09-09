# 01 — Runtime: CLR, JIT & GC

## ⚡ 30-second answer

**CoreCLR** is the execution engine. A native host bootstraps it from `runtimeconfig.json`, it
loads your assembly, and the **JIT (RyuJIT)** compiles IL → native code **per method, on first
call**, targeting the actual CPU. **Tiered compilation** compiles quickly at Tier 0 for fast
startup, then recompiles hot methods at Tier 1 with full optimization — guided by **dynamic
PGO**, which profiles real behavior to devirtualize and inline. Memory is managed by a
**generational mark-compact GC**: Gen 0/1/2 plus the Large Object Heap, exploiting "most objects
die young" so young collections stay cheap and full Gen 2 collections stay rare. Threads suspend
at **GC safepoints** so the collector can move objects and fix up references. **Native AOT**
trades the JIT's runtime adaptation for instant startup and a small self-contained binary.

---

## Core mechanics

### Startup path

```text
apphost → hostfxr → hostpolicy → CoreCLR → load assembly → JIT Main → run
                     ↑ reads runtimeconfig.json (framework version) + deps.json (assembly list)
```

Core components: **type loader** (builds method tables), **JIT**, **GC**, **exception handling**,
**threading/thread pool**, **interop**. They cooperate through **GC safepoints** — the JIT emits
GC info describing which registers and stack slots hold references at each point, so the runtime
can suspend threads and the GC can move objects safely.

### The JIT

- **Tier 0** — compile fast, optimize little. Startup is dominated by compile time, so this wins.
- **Tier 1** — after ~30 calls, recompile with full optimization.
- **OSR (On-Stack Replacement)** — upgrades a method *already running* a long loop, so a hot loop
  entered once doesn't stay stuck at Tier 0.
- **Dynamic PGO** (on by default since .NET 8) — Tier 0 code collects profile data (which types
  actually show up at a call site, which branches are taken), and Tier 1 uses it for
  **guarded devirtualization**, inlining, and hot/cold splitting. This is why interface-heavy
  idiomatic C# can hit near-`static`-call speed on hot paths.
- Optimizations worth naming: inlining, **bounds-check elimination**, constant folding, escape
  analysis (stack-allocating objects that don't escape), loop hoisting, auto-vectorization.
- **ReadyToRun (R2R)** pre-JITs at publish time for faster startup, while still allowing the JIT
  to re-optimize hot code at runtime.

**Interview-relevant consequence:** benchmark at **steady state**. A microbenchmark that measures
the first few calls is measuring Tier 0 — BenchmarkDotNet warms up precisely for this reason.

### The GC

**Generational hypothesis**: most objects die young. So the heap is split:

| Generation | Holds | Collected |
|---|---|---|
| **Gen 0** | fresh allocations | very often, very cheap |
| **Gen 1** | Gen 0 survivors — a buffer | often |
| **Gen 2** | long-lived objects | rarely; a **full GC**, expensive |
| **LOH** | objects ≥ 85,000 bytes | with Gen 2; **not compacted by default** |
| **POH** | pinned objects (.NET 5+) | separately, keeping pinning out of the main heap |

Collection is **mark-compact**: trace from **roots** (statics, stack slots, registers, GC handles),
mark reachable objects, move survivors together, then update every reference to point at the new
addresses. Because the heap stays compact, **allocation is a pointer bump** — nearly free, which
is why allocation-heavy .NET code performs better than people expect. The cost lands at
collection time.

**Write barriers + card tables**: when you assign a reference into an object's field, the runtime
records that the containing region is dirty, so a Gen 0 collection can find old→young references
without scanning all of Gen 2. Every reference write pays a small cost; writes to value-type
fields don't.

**Flavors**:

| Mode | Heaps | Best for |
|---|---|---|
| **Workstation** | one | desktop apps, low-latency small services |
| **Server** | one per core, with dedicated GC threads | **throughput servers** — the ASP.NET Core default |
| **Background GC** | Gen 2 marking runs concurrently | shrinking pause times (on by default) |
| **DATAS** | dynamically sizes heap to the real working set | modern Server GC default — big memory savings for typical services |

### Type system

Every reference object starts with an **object header** (sync block / hash) plus a **method table
pointer**. The method table defines size, GC layout, the **vtable**, base and interface maps, and
statics.

- **Virtual dispatch**: load MT → index a fixed vtable slot → call. **Interface dispatch** uses a
  dispatch map plus a cached stub. PGO frequently devirtualizes and inlines both away.
- **Value types unboxed** carry no header or MT — just the raw fields, dense and cache-friendly.
  **Boxing** wraps one in a heap object: a hidden allocation plus a copy.
- **Generics**: reference-type arguments **share one native code body** (all references are the
  same size); value-type arguments get **specialized code**. That's how you get type safety *and*
  allocation-free `List<int>`.
- `sealed` enables cheaper cast checks and easier devirtualization.

### Assemblies and metadata

An assembly is a PE file of **IL + metadata + a manifest**, loaded **lazily** on first type
reference and resolved via `deps.json`. Modern .NET has **no AppDomains** — isolation is
**`AssemblyLoadContext`**, which supports loading multiple versions side by side, custom
resolution, and (when `isCollectible: true`) **unloading**.

The **plugin pattern**: a collectible ALC plus `AssemblyDependencyResolver`, with the shared
contract assembly deliberately left in the Default context (return `null` from `Load` for it) so
casts across the boundary work. **Unload is a request** — it completes only once every reference
is gone.

Metadata is a set of relational tables (`TypeDef`, `MethodDef`, `CustomAttribute`, …) addressed
by 4-byte **tokens**. **Reflection is a metadata reader plus dynamic invoker** — correct but slow;
cache `MemberInfo`s or, better, move the work to **compile time with source generators**.
`typeof` compiles to a cheap `ldtoken`; `nameof` is free (a compile-time string).

### Threading

Most concurrency runs on the **thread pool**: a global queue plus **per-thread local queues with
work-stealing**, and **hill-climbing** that tunes worker count by measured throughput.

**Async I/O registers with the OS** (IOCP on Windows, epoll/kqueue on Linux/macOS) and **returns
the thread**. That is the whole reason async scales: 10,000 concurrent requests waiting on I/O
need a handful of threads, not 10,000.

Conversely, **blocking a pool thread causes starvation** — the pool only injects new threads
slowly (roughly one or two per second once past the core count), so a burst of blocking calls
collapses throughput. This is the runtime-level reason behind "never block on async."

### P/Invoke

A managed→native call goes through a **marshaling stub**: convert arguments, transition the GC
mode from **cooperative to preemptive**, call, convert back. That transition is the core fixed
cost (tens of nanoseconds) and the reason native code can't block the GC — and the reason to
batch native calls rather than making many tiny ones.

- **Blittable** types (primitives, pointers, structs of them) pass with no conversion — pinned,
  fast path. **Non-blittable** (`string`, `bool`, classes) marshal with allocation and copying.
- **`[LibraryImport]`** generates the stub at **compile time** (AOT/trim-safe);
  **`[DllImport]`** generates it at runtime via the JIT.
- `SuppressGCTransition` skips the mode switch — only for trivial, fast, non-blocking native
  functions, or you stall the GC.

---

## Comparison tables

| | JIT (default) | ReadyToRun | Native AOT |
|---|---|---|---|
| Compiled | at runtime, per method | at publish + rejit hot code | fully at publish |
| Startup | slowest | fast | **instant** |
| Peak throughput | **best** (PGO, OSR) | good | good, no runtime adaptation |
| Binary size | small | larger | single native file |
| Reflection/`Emit` | full | full | **restricted** |
| Best for | long-running servers | balanced | CLI tools, serverless, containers |

| Tier | When | Optimization |
|---|---|---|
| **Tier 0** | first calls | minimal — compile speed wins |
| **OSR** | long-running loop at Tier 0 | promotes a method mid-execution |
| **Tier 1** | hot methods (~30 calls) | full, PGO-guided |

| | Value type (unboxed) | Reference type |
|---|---|---|
| Header + MT pointer | none | yes (~16 bytes overhead on x64) |
| Lives | inline / stack | heap |
| Copy semantics | by value | by reference |
| Generic code | **specialized** per `T` | shared across all references |

---

## 🪤 Traps & gotchas

- **Benchmarking cold code** — the first calls run unoptimized Tier 0. Numbers from a hand-rolled
  `Stopwatch` loop with no warmup are measuring the JIT, not your algorithm. Use BenchmarkDotNet
  ([21](21-Performance-Tooling.md)).
- **"The GC is slow, I'll call `GC.Collect()`"** — forcing a collection promotes surviving
  objects to older generations, making the *next* full GC worse. Effectively never call it in
  production code.
- **Assuming a finalizer runs promptly, or at all.** Finalizable objects survive an extra
  collection (they're resurrected onto the finalizer queue), so they're strictly more expensive.
  `IDisposable` is the deterministic mechanism; a finalizer is a safety net for native handles.
- **Large Object Heap fragmentation** — anything ≥ 85,000 bytes goes to the LOH, which isn't
  compacted by default. Repeatedly allocating large arrays fragments it and grows the process
  even though "memory is free". Pool them (`ArrayPool<T>`) instead.
- **A `byte[]` of 85,000 elements is LOH; 84,000 isn't** — the threshold is bytes, not elements,
  so the boundary moves with element size.
- **Boxing in a hot loop** — every `object o = 42;`, every `IComparable` on a struct, every
  non-generic collection is a hidden allocation plus copy. Gen 0 collections rise and nobody can
  see why in the source.
- **Blocking on async** (`.Result`, `.Wait()`, `.GetAwaiter().GetResult()`) burns a pool thread
  while it waits, and the pool injects replacements slowly — throughput collapses under load
  long before CPU is saturated.
- **`Task.Run` for long-running work** — you've occupied a pool thread meant for short items. Use
  a dedicated thread (`TaskCreationOptions.LongRunning`) or a `BackgroundService`
  ([08](08-BackgroundProcessing.md)).
- **Server GC on a small container** — per-core heaps in a 1-CPU-limit pod can commit far more
  memory than you expect. DATAS mitigates this and is the modern default; know the knob exists.
- **Reflection in a hot path** — `MethodInfo.Invoke` is orders of magnitude slower than a direct
  call. Cache the `MemberInfo`, build a delegate, or use a source generator.
- **Reflection under Native AOT** — `Reflection.Emit`, `Expression.Compile` and `dynamic` don't
  work (`IL3050`), and anything not statically visible gets trimmed (`IL2xxx`). The fix is source
  generators, not suppressing the warnings.
- **`AssemblyLoadContext.Unload()` "not working"** — unload is a *request*. One lingering
  reference (an event handler, a cached delegate, a static) pins the whole context.
- **Forgetting the shared contract must stay in the Default context** — load it into the plugin
  ALC too and you get `InvalidCastException` between two identical-looking types.
- **Chatty P/Invoke** — each transition costs, so a per-element native call in a loop is dominated
  by marshaling. Batch into one call over a span.
- **`SuppressGCTransition` on a blocking native call** — the GC can't suspend that thread, so
  collections stall across the whole process.
- **Assuming `string` marshals for free** — it's non-blittable: allocation plus copy plus encoding
  conversion on every call.

---

## ❓ Likely questions

**Q: What does the JIT do, and what is tiered compilation?**
A: The JIT compiles IL to native code per method, on first call, targeting the actual CPU. Tiered
compilation compiles quickly and unoptimized at Tier 0 so startup is fast, then recompiles
methods that prove hot at Tier 1 with full optimization. OSR handles the case where a method
entered once is stuck in a long loop.

**Q: What is dynamic PGO and why does it matter?**
A: Tier 0 code collects a profile — which concrete types show up at each virtual call site, which
branches are taken — and Tier 1 compiles using it. That enables guarded devirtualization and
inlining of interface calls, so idiomatic abstraction-heavy C# can perform close to hand-written
static calls. It's on by default from .NET 8.

**Q: How does the GC work?**
A: It's generational and mark-compact. Objects are allocated in Gen 0; survivors are promoted to
Gen 1, then Gen 2. Collection traces from roots, marks what's reachable, compacts survivors
together, and updates all references. Because the heap stays compact, allocation is just a
pointer bump.

**Q: Why is generational GC a good idea?**
A: The generational hypothesis — most objects die young. Collecting only the young generation is
cheap because there are few survivors to move, and it's where nearly all the garbage is. Full
Gen 2 collections are expensive but rare.

**Q: What is the LOH and why does it matter?**
A: The Large Object Heap holds objects ≥ 85,000 bytes. It's collected with Gen 2 and **not
compacted by default**, so heavy large-array churn fragments it and grows the process. Pool large
buffers with `ArrayPool<T>` instead of reallocating.

**Q: Workstation vs Server GC?**
A: Workstation uses a single heap and is tuned for latency on client machines. Server GC uses one
heap per core with dedicated GC threads for throughput — the ASP.NET Core default. Background GC
does Gen 2 marking concurrently to cut pauses. DATAS then right-sizes the heap to the actual
working set.

**Q: What are write barriers and card tables for?**
A: To find references from old objects into young ones without scanning all of Gen 2. Every
reference write marks its region dirty in the card table, and a Gen 0 collection scans only the
dirty regions.

**Q: What is boxing, at the runtime level?**
A: Allocating a heap object with an object header and method-table pointer, copying the value
type's bytes into it, and handing back a reference. It's a hidden allocation and copy — which is
why it shows up as Gen 0 pressure that isn't visible in the source.

**Q: How do generics work with value types vs reference types?**
A: Reference-type arguments all share one native code body, because every reference is the same
size. Value-type arguments get specialized native code per type. That's what makes `List<int>`
allocation-free per element while keeping one implementation for all reference types.

**Q: Why does async scale?**
A: Async I/O registers the operation with the OS (IOCP/epoll) and **returns the thread to the
pool**. So thousands of concurrent I/O operations need only a handful of threads, instead of one
blocked thread each.

**Q: What is thread-pool starvation?**
A: Pool threads are blocked (usually by sync-over-async) rather than doing work. The pool injects
replacements only slowly, so throughput collapses and latency spikes while CPU sits idle — the
diagnostic signature that distinguishes it from a CPU bottleneck.

**Q: What replaced AppDomains?**
A: `AssemblyLoadContext`. It gives you isolation, side-by-side versions, custom resolution, and —
when collectible — unloading. Unload is a request that completes only when all references drop.

**Q: What are Native AOT's limits, and why?**
A: They follow from the model. No JIT means no `Reflection.Emit`, `Expression.Compile` or
`dynamic`. Whole-program trimming means reflection targets must be statically visible or they're
removed. The GC, type system, BCL, async and exceptions are all unchanged, so most code just
works. Source generators are the standard fix.

**Q: When would you choose Native AOT over the JIT?**
A: When startup time and footprint dominate — CLI tools, serverless functions, short-lived
containers. You give up the JIT's runtime adaptation (PGO, OSR), so for a long-running throughput
server the JIT usually wins on peak performance.

**Q: `[LibraryImport]` vs `[DllImport]`?**
A: `[LibraryImport]` is a source generator that emits the marshaling stub at compile time, so it's
trim- and AOT-safe and the marshaling is visible code. `[DllImport]` generates the stub at runtime
via the JIT. New code should use `[LibraryImport]`.

**Q: What's the fixed cost of a P/Invoke call?**
A: The GC-mode transition from cooperative to preemptive and back — tens of nanoseconds. It exists
so native code can't block a collection. It's why you batch native calls rather than calling per
element.

---

## 🎓 Senior Extra

- **GC info is a JIT output.** The JIT records, for every safepoint, which registers and stack
  slots hold live references — which is precisely what lets the collector move objects and fix
  references. This is the coupling that makes precise, compacting GC possible at all.
- **Allocation is a pointer bump; the cost is at collection.** So the useful metric is not
  "allocations" but **survivors** — an object that dies in Gen 0 is nearly free, while one that
  gets promoted costs you a copy and eventually a Gen 2 trace.
- **Watch `% time in GC` and Gen 2 collection rate**, not just working set. A rising Gen 2
  frequency is the signal that objects are being promoted that shouldn't be — usually a cache
  without a bound or a static collection that only grows ([21](21-Performance-Tooling.md)).
- **`GCSettings.LatencyMode` / `SustainedLowLatency`** exist, but the honest senior answer is that
  tuning GC modes is a last resort after fixing allocation patterns.
- **Escape analysis** can stack-allocate objects that provably don't escape a method, which is why
  "it allocates" is increasingly a claim to verify rather than assume.
- **`ArrayPool<T>` / `MemoryPool<T>` / `RecyclableMemoryStream`** are the standard answers to LOH
  churn; `Span<T>` and `stackalloc` avoid the heap entirely for short-lived buffers
  ([02](02-BCL-Essentials.md)).
- **Struct tearing**: a multi-field struct written without synchronization can be observed
  half-updated, because only naturally-aligned word-sized writes are atomic. Worth knowing before
  you "avoid locks" with a struct.
- **The plugin ALC leak checklist** — event subscriptions back to the host, static caches,
  `Type`/`MethodInfo` held anywhere, and running tasks. Any of them pins the context and unload
  silently never completes.
- **`MetadataLoadContext`** inspects assemblies without executing them — the mechanism behind the
  trimmer, the AOT compiler, and most analyzers, and the right tool when you need to reflect over
  something you don't want to load.
- **Compile-time over runtime is the through-line of modern .NET**: source generators for
  serialization, logging, regex, P/Invoke, and DI-adjacent glue. Every one of them replaces
  reflection with generated code, and every one is what makes AOT viable.

→ Deeper: [`../01-Runtime/`](../01-Runtime/README.md)
