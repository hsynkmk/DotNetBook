# System.IO.Pipelines (for Networking)

## High-throughput protocol parsing

`System.IO.Pipelines` is the high-performance, low-allocation I/O library for parsing streaming data — the engine inside Kestrel, SignalR, and gRPC. In networking, it solves the painful parts of reading from a socket: buffer management, **partial reads** (TCP gives you a byte stream, not messages — [06-Sockets.md](06-Sockets.md)), and back-pressure.

> The full mechanics — `PipeReader`/`PipeWriter`, `ReadOnlySequence<byte>`, `AdvanceTo(consumed, examined)`, `SequenceReader<byte>`, back-pressure thresholds — are in **CSharpBook Chapter 13 §03**. This file frames *why* networking code reaches for pipelines and the typical socket-parsing shape.

```csharp
// Wrap a network stream in a PipeReader and parse messages
PipeReader reader = PipeReader.Create(networkStream);
while (true) {
    ReadResult result = await reader.ReadAsync(ct);
    ReadOnlySequence<byte> buffer = result.Buffer;

    while (TryParseMessage(ref buffer, out Message msg))   // extract complete messages
        await HandleAsync(msg);

    reader.AdvanceTo(buffer.Start, buffer.End);            // consumed vs examined
    if (result.IsCompleted) break;
}
await reader.CompleteAsync();
```

---

## Why pipelines for socket code

Reading a protocol from a raw socket forces you to handle, by hand:
- **Partial messages** — a `ReceiveAsync` may return half a message, one message, or several. You must buffer the remainder and prepend it to the next read.
- **Buffer management** — allocating and reusing `byte[]` buffers without churning the GC.
- **Buffer copying** — shuffling leftover bytes to the front on every read (a memmove per read).
- **Back-pressure** — slowing the reader when the consumer can't keep up.

Pipelines handle all four: it pools buffers (near-zero allocation), exposes a multi-segment `ReadOnlySequence<byte>` (so leftover bytes aren't copied — a new segment is chained), tracks **consumed vs examined** via `AdvanceTo` (so partial messages wait for more data without busy-looping), and provides built-in back-pressure (the writer pauses when the reader lags). This is why every high-throughput .NET server (Kestrel) uses pipelines for socket parsing.

---

## The framing problem, solved

The TCP framing problem from [06-Sockets.md](06-Sockets.md) (recovering message boundaries from a byte stream) is exactly what pipelines + `SequenceReader<byte>` make clean:

```csharp
// Parse length-prefixed or delimited messages from the pipe's buffer
bool TryParseMessage(ref ReadOnlySequence<byte> buffer, out Message message) {
    var reader = new SequenceReader<byte>(buffer);
    if (reader.TryReadBigEndian(out int length) && reader.Remaining >= length) {
        var payload = buffer.Slice(reader.Position, length);
        message = Deserialize(payload);
        buffer = buffer.Slice(reader.Position).Slice(length);   // advance past this message
        return true;
    }
    message = default;
    return false;   // incomplete — wait for more data (AdvanceTo with examined = buffer.End)
}
```

`SequenceReader<byte>` reads across the (possibly multi-segment) buffer without copying it into one contiguous array, and `AdvanceTo(consumed, examined)` tells the pipe "I consumed up to here, but examined to here" — so if a message is incomplete, the next `ReadAsync` waits for *more* data rather than handing back the same bytes. This eliminates the manual buffer-juggling that makes raw socket parsing error-prone.

---

## When to use pipelines

- **Writing a custom network protocol parser** at scale (a binary protocol over TCP).
- **High-throughput servers** where allocation and copying dominate (you've measured it).
- Working **below** the framework — implementing the transport itself.

When **not** to: ordinary HTTP (use `HttpClient`/ASP.NET Core — they already use pipelines internally), simple/low-volume socket code (the manual approach in [06-Sockets.md](06-Sockets.md) is fine), or anything a higher-level protocol (gRPC/SignalR) handles. Pipelines add real complexity — reach for them only for high-performance protocol code you've measured a need for, and learn the mechanics in CSharpBook Ch13 §03 first.

---

## Common gotchas

(Detailed in CSharpBook Ch13 §03 — the highlights for networking:)

- **Wrong `AdvanceTo(consumed, examined)`** — claiming you consumed more than you parsed loses data; examining too little busy-loops. Set `consumed` to the end of complete messages, `examined` to the end of the buffer when a message is incomplete.
- **Treating `ReadOnlySequence` as contiguous** — it may have multiple segments. Check `IsSingleSegment` or use `SequenceReader`.
- **Forgetting to `Complete`** — both ends must complete the pipe, or the counterpart waits forever.
- **Using pipelines for simple cases** — overkill for ordinary HTTP or low-volume sockets; use the framework or basic socket reads.

---

## Summary

- **`System.IO.Pipelines`** is the high-throughput, low-allocation library for parsing streaming data — the engine inside Kestrel/SignalR/gRPC — and the right tool for **custom network protocol parsers** at scale.
- For socket code it solves **partial reads** (TCP framing), **buffer pooling** (no GC churn), **no leftover-copying** (multi-segment `ReadOnlySequence`), and **back-pressure** — the painful parts of raw socket parsing ([06-Sockets.md](06-Sockets.md)).
- Wrap a network stream in a **`PipeReader`**, extract complete messages with **`SequenceReader<byte>`**, and signal progress with **`AdvanceTo(consumed, examined)`** so incomplete messages wait for more data.
- Use it only for **measured high-performance protocol code**; ordinary HTTP/gRPC/SignalR already use it internally. Full mechanics: **CSharpBook Chapter 13 §03**.

→ Next: [Questions.md](Questions.md)
