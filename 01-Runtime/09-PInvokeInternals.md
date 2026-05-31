# P/Invoke Internals

## What happens at the boundary

When managed code calls a native function, the runtime can't just `call` the address — managed and native code disagree about calling conventions, data layout, string encoding, error reporting, and (critically) how the GC interacts with the thread. The runtime bridges this with a **marshaling stub**: generated glue code that converts arguments, transitions the GC mode, calls the native function, and converts results back.

> CSharpBook Chapter 14 covers **how to write** P/Invoke (`[LibraryImport]`, `[DllImport]`, marshaling attributes, `SafeHandle`, callbacks). This file covers **what the runtime does** under the hood and *why* the rules exist.

```
Managed caller
   ↓
[ Marshaling stub ]   ← generated glue (source-gen at compile time, or JIT at runtime)
   • marshal arguments (convert/copy/pin)
   • transition GC: cooperative → preemptive
   • call the native function
   • transition GC: preemptive → cooperative
   • marshal return value + out params
   • capture last error (if requested)
   ↓
Native function (in a .dll/.so/.dylib)
```

---

## The GC-mode transition (the heart of P/Invoke cost)

This is the most important runtime detail. Managed threads normally run in **cooperative mode**: they cooperate with the GC by reaching safepoints where they can be suspended so the GC can move objects ([01-CoreCLRArchitecture.md](01-CoreCLRArchitecture.md)). But native code knows nothing about the GC and will never reach a managed safepoint — if a thread sat in native code in cooperative mode, it would **block the GC indefinitely** (the GC can't suspend a thread that's off in native land, but it also can't proceed without it).

The fix: before calling native code, the stub switches the thread to **preemptive mode** — telling the GC "I'm leaving managed code; collect without waiting for me, just don't move objects I've pinned." On return, it switches back to cooperative.

```
Cooperative mode:  GC must suspend this thread at a safepoint before collecting.
Preemptive mode:   GC may collect freely; this thread isn't touching managed objects.
```

This transition is the **fixed overhead** of every P/Invoke (~tens of nanoseconds). It's why you batch native calls rather than calling a native function millions of times in a tight loop.

### `SuppressGCTransition` — skipping the transition

For tiny, fast, **non-blocking** native functions that don't call back into managed code and won't run long, you can skip the transition:

```csharp
[LibraryImport("mylib")]
[SuppressGCTransition]                  // omit the cooperative↔preemptive switch
internal static partial int FastAdd(int a, int b);
```

This removes the per-call overhead, approaching a direct native call. But it's **dangerous if misused**: if the function blocks or runs long, it *delays GC for the whole process* (the thread stays cooperative-mode-but-in-native, stalling collections). Use only for trivial, fast, non-blocking calls — and measure.

---

## Marshaling — converting data across the boundary

Native code expects C-style data layouts; managed objects have headers, GC-managed layouts, and UTF-16 strings. The stub **marshals** (converts) arguments:

### Blittable types — no conversion

A type is **blittable** if its managed and native representations are **identical** — primitives (`int`, `double`, `byte`, `IntPtr`), pointers, and structs/arrays composed only of blittable types. Blittable args need **no conversion**: the stub can pass them directly (pinning arrays so the GC won't move them during the call). This is the fast path — keep interop signatures blittable.

```csharp
[LibraryImport("mylib")]
internal static partial int Sum(int* data, int count);   // int* and int are blittable → cheap
```

### Non-blittable types — conversion + copy

- **`string`** — managed strings are UTF-16; the native side may want UTF-8 or ANSI. The stub allocates a native buffer and converts (a copy). `StringMarshalling.Utf8/Utf16` controls it.
- **`bool`** — managed `bool` is 1 byte but Win32 `BOOL` is 4 bytes; the stub converts (a classic mismatch bug if you don't specify `[MarshalAs]`).
- **arrays of non-blittable types, classes, delegates** — require element-by-element conversion or special handling.

Non-blittable marshaling means **allocation + copy** per call — more reason to design blittable APIs for hot paths.

### Pinning

For blittable arrays/buffers passed by pointer, the stub **pins** the managed object — telling the GC "don't move this during the native call" — so the native pointer stays valid. Pinning is cheap but, if held long (e.g., a long-running native call holding a pinned buffer), can fragment the GC heap (which is why the **Pinned Object Heap** exists — [04-GCDeepDive.md](04-GCDeepDive.md)).

---

## `[LibraryImport]` (source-gen) vs `[DllImport]` (runtime stub)

The two ways to declare P/Invoke differ in *when* the marshaling stub is created:

| | `[DllImport]` (classic) | `[LibraryImport]` (modern, .NET 7+) |
|---|---|---|
| Stub generation | **at runtime**, by the JIT (IL stub) | **at compile time**, by a source generator |
| AOT-compatible | partial (runtime stub needs codegen) | **yes** (stub is real, visible C#) |
| Trimming-safe | weaker | **yes** |
| Declaration | `static extern` | `static partial` |

`[LibraryImport]` is preferred: the source generator emits the marshaling code as ordinary C# at build time — so it works under Native AOT (no runtime codegen needed) and the trimmer can see it. `[DllImport]` generates the stub via the JIT at runtime, which is why heavy reflection/AOT scenarios prefer `[LibraryImport]`. (Practical usage: CSharpBook Ch14 §01.)

---

## Callbacks (native → managed)

When native code calls back into managed code (e.g., a comparator passed to native `qsort`), the runtime needs a native function pointer that, when invoked, transitions **back** into managed mode (preemptive → cooperative) and dispatches to your method.

- The classic approach passes a **delegate** (marshaled to a native function pointer), but the delegate must be **kept alive** — if the GC collects it while native code still holds the pointer, the callback crashes.
- The modern, AOT-friendly approach is **`[UnmanagedCallersOnly]`** + a function pointer (`delegate* unmanaged<...>`): a static method directly callable from native code with no delegate object to keep alive.

```csharp
[UnmanagedCallersOnly(CallConvs = [typeof(CallConvCdecl)])]
static int Compare(int a, int b) => a.CompareTo(b);
// pass &Compare as a delegate* unmanaged[Cdecl]<int,int,int> — no GC-rooting needed
```

(Details: CSharpBook Ch14 §01.)

---

## Error handling across the boundary

Native functions report errors via a thread-local "last error" (`GetLastError` on Windows, `errno` on Unix). Managed code between the call and reading the error could clobber it, so you must opt in to capturing it at the right moment:

```csharp
[LibraryImport("kernel32.dll", SetLastError = true)]
[return: MarshalAs(UnmanagedType.Bool)]
internal static partial bool CloseHandle(IntPtr h);

if (!CloseHandle(handle)) {
    int err = Marshal.GetLastPInvokeError();   // captured by the stub right after the call
    throw new Win32Exception(err);
}
```

`SetLastError = true` makes the stub capture the error immediately after the native call (before any managed code runs), and `GetLastPInvokeError()` reads it.

---

## Common gotchas (runtime view)

### P/Invoke in a tight loop

Each call pays the GC-mode transition (~tens of ns) plus any marshaling. Calling a native function millions of times per second is dominated by transition overhead — batch the work into fewer native calls, or use `SuppressGCTransition` for trivial functions.

### Non-blittable signatures on hot paths

Strings/bools/classes marshal with allocation + copy each call. Design blittable signatures (pointers + lengths, `Span<byte>`) for performance-critical interop.

### `SuppressGCTransition` on a blocking call

Skipping the transition on a function that blocks or runs long stalls GC for the whole process. Only for fast, non-blocking calls.

### Long-held pinned buffers

Pinning for a long native call fragments the heap. Keep native calls short or use the POH / pre-pinned buffers.

### Forgetting to keep a callback delegate alive

A GC'd delegate behind a native function pointer → crash on callback. Store it, or use `[UnmanagedCallersOnly]`.

---

## Summary

- A P/Invoke call goes through a **marshaling stub** that converts arguments, transitions the GC mode, calls native code, and converts results back.
- The **cooperative→preemptive GC-mode transition** is the core fixed cost (~tens of ns) and the reason native code can't block the GC — it's why you batch native calls. **`SuppressGCTransition`** skips it for trivial, fast, non-blocking functions only.
- **Blittable** types (primitives, pointers, blittable structs) pass with no conversion (fast path, pinned); **non-blittable** types (`string`, `bool`, classes) marshal with allocation + copy.
- **`[LibraryImport]`** generates the stub at **compile time** (AOT/trim-safe); **`[DllImport]`** generates it at **runtime** via the JIT.
- Callbacks need the delegate kept alive (or use **`[UnmanagedCallersOnly]`** + function pointers, AOT-friendly); errors are captured via `SetLastError` + `GetLastPInvokeError`.
- Practical syntax and `SafeHandle`: CSharpBook Chapter 14.

→ Next: [Questions.md](Questions.md)
