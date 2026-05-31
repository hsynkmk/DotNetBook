# dotnet-dump

## Inspecting the heap and call stacks

**`dotnet-dump`** captures a **memory dump** of a .NET process — a snapshot of everything in memory at one instant: the managed heap (every object), thread stacks, and runtime state — and then lets you analyze it with **SOS-style commands** (`dumpheap`, `dumpobj`, `gcroot`, `clrstack`). It's the tool for **memory problems** (what's on the heap, why isn't it collected — i.e., **leaks**) and for **hangs/crashes** (what were the threads doing). Where `dotnet-trace` gives a *time* profile ([04-DotnetTrace.md](04-DotnetTrace.md)), `dotnet-dump` gives a *snapshot* — the contents of memory, frozen.

```bash
dotnet-dump collect --process-id 1234          # capture a dump (creates core_xxxx)
dotnet-dump analyze core_1234                  # open the interactive analysis prompt
```

---

## Capturing and analyzing

```bash
dotnet-dump collect -p 1234           # snapshot the process (it briefly pauses)
dotnet-dump analyze core_1234         # SOS-style REPL
```

Inside `analyze`, you run **SOS commands** (the same diagnostics historically used in WinDbg, now cross-platform). The workflow for a memory investigation:

1. **`dumpheap -stat`** — summarize the heap by type: count and total bytes per type, sorted. This shows *what's eating memory* — e.g., "2.1 GB of `byte[]`" or "800,000 `OrderViewModel` instances."
2. **`dumpheap -type Foo`** — list the actual instances of a suspicious type, with their addresses.
3. **`gcroot <address>`** — **why is this object still alive?** It traces the chain of references **rooting** the object (what's keeping it from being collected — a static field, an event subscription, a captured closure). This is the key to finding leaks.
4. **`dumpobj <address>`** — inspect an object's fields.
5. **`clrstack` / `clrthreads`** — thread call stacks (for hangs/deadlocks — [CSharpBook Ch08](../../CSharpBook/08-Concurrency/README.md)).

---

## Finding a memory leak

The canonical use. `dotnet-counters` ([03-DotnetCounters.md](03-DotnetCounters.md)) shows heap growing without GC reclaiming it → capture a dump (ideally two, over time, to compare growth) and:

```
> dumpheap -stat
Statistics:
  MT    Count    TotalSize   Class Name
  ...   850,000  340,000,000 MyApp.CachedItem      ← suspiciously many
> dumpheap -type MyApp.CachedItem      (pick one address)
> gcroot 00007f...
  -> MyApp.CacheService              (static field _cache)   ← the root!
  -> System.Collections.Dictionary...
  -> MyApp.CachedItem
```

`gcroot` reveals the **retention path** — here, a static `Dictionary` cache that never evicts, keeping every `CachedItem` alive forever. The most common .NET "leaks" are exactly this: **objects rooted by something long-lived** — static collections, **un-unsubscribed events** (the publisher holds subscribers — [CSharpBook Ch05](../../CSharpBook/05-DelegatesEvents/README.md)), captured closures, or a cache without bounds. (.NET is garbage-collected, so a "leak" is always *unintended reachability*, not unfreed memory — [CSharpBook Ch09 §13](../../CSharpBook/09-MemoryPerformance/README.md).)

---

## Analyzing hangs and crashes

For a hung or deadlocked process, capture a dump and inspect threads:

- **`clrthreads`** — list all managed threads.
- **`clrstack`** (per thread, or `~*e !clrstack`-style) — what each thread is executing.
- A **deadlock** shows as threads blocked on locks held by each other; **thread-pool starvation** shows many threads blocked on `.Result`/`.Wait()` (sync-over-async — [10-CommonWins.md](10-CommonWins.md)).

For crashes, a dump captured at the fault (or a post-mortem core dump) shows the faulting thread's stack and state.

---

## When to use a dump vs a trace

| Question | Tool |
|---|---|
| Where does CPU/allocation time go? | **dotnet-trace** (time profile — [04](04-DotnetTrace.md)) |
| What's on the heap / why isn't it collected (leak)? | **dotnet-dump** (heap snapshot + `gcroot`) |
| Why is the app hung / what are threads doing? | **dotnet-dump** (`clrthreads`/`clrstack`) |
| What's happening live (triage)? | **dotnet-counters** ([03](03-DotnetCounters.md)) |

A dump is a **point-in-time snapshot** — perfect for "what is the state of memory/threads right now," not for "where does time go over a period" (that's a trace). For interactive heap analysis with a GUI, **dotMemory** ([08-JetBrainsTools.md](08-JetBrainsTools.md)) is friendlier; `dotnet-dump` is the scriptable, cross-platform, no-GUI option (great for servers/containers).

---

## Common gotchas

### Dump size and pause

A full dump captures the whole heap — it can be **large** (gigabytes for a big process) and briefly **pauses** the process. Plan for the size/pause on production captures.

### Expecting a dump to show "where time goes"

A dump is a *snapshot*, not a time profile. For CPU/allocation hot spots over time, use **dotnet-trace** ([04](04-DotnetTrace.md)); use a dump for heap state and hangs.

### Misunderstanding "leak"

In a GC'd runtime, a leak is **unintended reachability** — something long-lived (static, event, cache) roots objects. `gcroot` finds the retention path; the fix is removing that root (unsubscribe, bound/evict the cache), not "freeing" memory.

### One dump for growth analysis

A single dump shows current state, not what's *growing*. Capture **two dumps over time** and compare `dumpheap -stat` to see which types are accumulating.

### Symbol/version mismatch

Analysis needs the right runtime/symbols. Analyze with a compatible SOS/runtime version (the `dotnet-dump` tooling generally handles this, but mismatches cause unreadable stacks).

---

## Summary

- **`dotnet-dump`** captures a **memory snapshot** (managed heap, thread stacks, runtime state) and analyzes it with **SOS commands** (`dumpheap -stat`, `dumpobj`, **`gcroot`**, `clrstack`) — the tool for **leaks** and **hangs/crashes** (a snapshot, vs `dotnet-trace`'s time profile).
- **Find leaks**: `dumpheap -stat` shows what's eating the heap, then **`gcroot`** reveals the **retention path** — the long-lived root (static collection, un-unsubscribed event, unbounded cache) keeping objects alive (a .NET "leak" = unintended reachability — [CSharpBook Ch09 §13](../../CSharpBook/09-MemoryPerformance/README.md)).
- **Diagnose hangs**: `clrthreads`/`clrstack` show what threads are doing — deadlocks and **thread-pool starvation** (threads blocked on `.Result`) are visible.
- Use a **dump** for heap/thread *state*, a **trace** for *time*, **counters** for live triage; capture **two dumps over time** to find growth; mind dump **size/pause**. **dotMemory** ([08](08-JetBrainsTools.md)) is the GUI alternative.

→ Next: [06-PerfView.md](06-PerfView.md)
