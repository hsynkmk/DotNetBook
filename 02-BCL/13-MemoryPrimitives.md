# Memory Primitives — Span, Memory, Buffers

## The high-performance I/O building blocks

`Span<T>`, `Memory<T>`, `ArrayPool<T>`, `ReadOnlySequence<T>`, and `ArrayBufferWriter<T>` are the BCL types that let you process data with **minimal allocations and zero copying**. They're why modern .NET I/O, JSON, and networking are fast. This file maps them and when to use which.

> CSharpBook Chapter 09 §05–07 covers `Span<T>`/`Memory<T>`/`ArrayPool<T>` mechanics in depth (stack-only semantics, slicing, pooling). This file is the **BCL toolkit overview** and the decision guide for high-perf buffer code.

---

## `Span<T>` / `ReadOnlySpan<T>` — zero-copy windows

A `Span<T>` is a **window** over contiguous memory — an array, a slice of one, a `stackalloc` buffer, or even unmanaged memory — with no allocation and no copy:

```csharp
int[] arr = [1, 2, 3, 4, 5];
Span<int> middle = arr.AsSpan(1, 3);     // {2,3,4} — a VIEW into arr, no copy
middle[0] = 99;                           // mutates arr[1]

ReadOnlySpan<char> text = "key=value".AsSpan();
ReadOnlySpan<char> key = text[..text.IndexOf('=')];   // slice without allocating a substring

Span<byte> stack = stackalloc byte[256];  // stack-allocated buffer, no heap, no GC
```

Spans are the **allocation-free path** for slicing, parsing, and buffer manipulation. Crucially, `Span<T>` is a **`ref struct`** — it can only live on the stack: **it can't be a field, can't be captured by a lambda, and can't cross an `await`**. That restriction (enforced by the compiler) is what makes it safe and free. For those cases, use `Memory<T>`.

---

## `Memory<T>` / `ReadOnlyMemory<T>` — heap-storable windows

`Memory<T>` is `Span<T>`'s heap-friendly sibling: same "window over memory" idea, but **can** be stored in fields, captured, and used across `await` (because it's not a `ref struct`):

```csharp
async Task ProcessAsync(ReadOnlyMemory<byte> buffer) {
    await stream.WriteAsync(buffer);          // Memory works across await; Span would not compile
    Memory<byte> slice = someArray.AsMemory(0, 100);
    DoSync(slice.Span);                        // get a Span when you need synchronous, zero-cost access
}
```

The rule: **`Span<T>` for synchronous, stack-local hot paths; `Memory<T>` for async APIs and when you must store the window.** Convert `Memory<T>` → `Span<T>` (`.Span`) at the point of synchronous use. Async I/O APIs (`Stream.ReadAsync`, `Channel`, pipelines) take `Memory<T>` for exactly this reason. (CSharpBook Ch09 §06.)

---

## `ArrayPool<T>` — reuse buffers instead of allocating

Allocating large temporary buffers repeatedly churns the GC (and large ones hit the LOH — [Ch01 §04](../01-Runtime/04-GCDeepDive.md)). `ArrayPool<T>` rents and returns reusable arrays:

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(8192);    // may return a LARGER array than asked
try {
    int read = await stream.ReadAsync(buffer.AsMemory(0, 8192), ct);
    Process(buffer.AsSpan(0, read));                   // use only the length you requested
}
finally {
    ArrayPool<byte>.Shared.Return(buffer);             // always return (in finally)
}
```

Rules:
- `Rent(n)` may return an array **larger** than `n` — use the length you asked for, not `buffer.Length`.
- **Always `Return`** (in `finally`), or the pool degrades to plain allocation.
- The returned buffer's contents are **not cleared** by default — pass `clearArray: true` to `Return` for sensitive data, or don't assume it's zeroed on rent.
- Don't use the buffer after returning it (use-after-return bug).

`ArrayPool` is the standard way to eliminate per-operation buffer allocations in I/O, serialization, and parsing loops. (CSharpBook Ch09 §07.)

---

## `ReadOnlySequence<T>` — multi-segment buffers

Sometimes data isn't contiguous — it arrives in chunks (network packets, pipeline segments). `ReadOnlySequence<T>` represents a sequence that may span **multiple discontiguous memory segments** without copying them into one buffer:

```csharp
void Parse(ReadOnlySequence<byte> seq) {
    if (seq.IsSingleSegment) {
        ParseSpan(seq.FirstSpan);            // fast path: contiguous
    } else {
        var reader = new SequenceReader<byte>(seq);   // parse across segments
        while (reader.TryReadTo(out ReadOnlySequence<byte> line, (byte)'\n'))
            ProcessLine(line);
    }
}
```

This is the buffer type **`System.IO.Pipelines`** (`PipeReader`) hands you — it avoids copying leftover bytes between reads by chaining segments. `SequenceReader<byte>` parses across the segment boundaries. (Pipelines: CSharpBook Ch13 §03, [04-IO.md](04-IO.md).)

---

## `ArrayBufferWriter<T>` / `IBufferWriter<T>` — efficient building

When *writing* a variable amount of data, `IBufferWriter<T>` lets a producer request spans to write into and commit how much it used — the write-side counterpart to spans:

```csharp
var writer = new ArrayBufferWriter<byte>();
Span<byte> span = writer.GetSpan(sizeHint: 256);     // get a buffer to write into
int written = Encoding.UTF8.GetBytes("hello", span);
writer.Advance(written);                              // commit the bytes
ReadOnlyMemory<byte> result = writer.WrittenMemory;   // the accumulated output
```

`Utf8JsonWriter`, `PipeWriter`, and many BCL serializers write into an `IBufferWriter<T>` — so your types can serialize efficiently into pooled buffers without intermediate allocations (`IUtf8SpanFormattable` — [01-Strings.md](01-Strings.md)).

---

## How they fit together (the high-perf I/O pipeline)

```
Network/file bytes
   → PipeReader gives a ReadOnlySequence<byte>   (multi-segment, pooled, no copy)
   → SequenceReader<byte> parses across segments
   → parse values with ReadOnlySpan<T> + span TryParse  (no substring allocations)
   → build output into an IBufferWriter<T> / ArrayBufferWriter<T>  (pooled)
   → Utf8JsonWriter / PipeWriter writes UTF-8 directly  (no UTF-16 round-trip)
```

This allocation-free, copy-free chain is how Kestrel and System.Text.Json achieve their throughput. You compose the same primitives in your own hot paths. (Performance idioms: CSharpBook Ch17 §03.)

---

## When to reach for these (and when not)

- **Use** spans/pooling on **measured hot paths** — high-throughput parsing, serialization, I/O loops, network servers.
- **Don't** rewrite ordinary business code with `Span`/`ArrayPool` — it adds complexity and `ref struct` constraints for no benefit where allocations aren't the bottleneck.
- The .NET 10 JIT also does **escape analysis** ([Ch01 §02](../01-Runtime/02-JIT.md)) that stack-allocates some short-lived objects automatically — so write clean code first, then optimize the proven-hot 5% with these primitives. **Measure first** ([Ch21](../21-Performance/README.md)).

---

## Common gotchas

### Storing a `Span<T>` in a field or using it across `await`

Won't compile — `Span<T>` is a `ref struct` (stack-only). Use `Memory<T>` for those cases, convert to `Span` at synchronous use.

### Using `buffer.Length` after `ArrayPool.Rent`

`Rent` may return a larger array. Track the length you requested; use that, not `.Length`.

### Forgetting to `Return` a rented array

Defeats the pool (falls back to allocation) and can leak. Return in `finally`; never use after returning.

### Assuming rented buffers are zeroed

They're not cleared by default. Don't read uninitialized regions; clear sensitive data on return (`clearArray: true`).

### Treating `ReadOnlySequence<T>` as contiguous

It may have multiple segments. Check `IsSingleSegment` or use `SequenceReader<T>`.

### Premature span-ifying

Adding `Span`/pooling to cold code adds risk for no gain. Profile; optimize hot paths only.

---

## Summary

- **`Span<T>`/`ReadOnlySpan<T>`** are zero-copy, zero-alloc windows over memory — the fast path for slicing/parsing — but are **stack-only** (`ref struct`: no fields, no capture, no `await`).
- **`Memory<T>`/`ReadOnlyMemory<T>`** are the heap-storable, async-capable counterpart; convert to `Span` (`.Span`) for synchronous use. Async I/O APIs take `Memory<T>`.
- **`ArrayPool<T>`** reuses buffers (rent/return in `finally`; may return larger; not zeroed) to eliminate per-op allocations.
- **`ReadOnlySequence<T>`** + `SequenceReader<T>` handle multi-segment buffers (pipelines); **`IBufferWriter<T>`/`ArrayBufferWriter<T>`** are the efficient write side.
- Together they form the allocation-free I/O pipeline behind Kestrel and STJ — use on **measured hot paths**, not everywhere. Mechanics: CSharpBook Chapter 09.

→ Next: [Questions.md](Questions.md)
