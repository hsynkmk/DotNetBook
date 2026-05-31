# Chapter 02 — Base Class Library

> The library that ships with .NET — `System.*`, `System.IO`, `System.Text`, `System.Collections`, `System.Threading`, `System.Net`, and the dozen other namespaces that make up "what you have without NuGet."

**Prerequisites**: comfortable with C# generics, collections, async.

**Time to read**: ~12-16 hours (deep dive across many namespaces).

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-Strings.md](01-Strings.md) | `String`, `StringBuilder`, `Span<char>`, formatting (composite, interpolated, custom), `string.Create`, interning. |
| [02-Numerics.md](02-Numerics.md) | `Math`, `MathF`, `BigInteger`, `Complex`, `Random`, `RandomNumberGenerator`, `BitConverter`. |
| [03-CollectionsDeep.md](03-CollectionsDeep.md) | Specialized collections beyond CSharpBook: `Concurrent*`, `Immutable*`, `Frozen*`, `OrderedDictionary`, `BitArray`, `BlockingCollection`. |
| [04-IO.md](04-IO.md) | File, Directory, Path, Stream hierarchy, FileStream, MemoryStream, BufferedStream, pipelines (System.IO.Pipelines). |
| [05-Serialization.md](05-Serialization.md) | System.Text.Json (deep), XmlSerializer, DataContract, binary alternatives. |
| [06-DateTimeAndTime.md](06-DateTimeAndTime.md) | DateTime, DateTimeOffset, TimeSpan, TimeZoneInfo, TimeProvider (.NET 8+), TimeOnly, DateOnly. |
| [07-Globalization.md](07-Globalization.md) | CultureInfo, Encoding, normalization, casing, region-aware ops. |
| [08-Diagnostics.md](08-Diagnostics.md) | Activity, ActivitySource, Meter (System.Diagnostics.Metrics), Process, Stopwatch, EventSource, Debug, Trace. |
| [09-Reflection.md](09-Reflection.md) | Type, MemberInfo, Activator, dynamic invocation, attribute reading. |
| [10-Net.md](10-Net.md) | `System.Net.*` — HttpClient (briefly; ch 08 covers in depth), DNS, IPAddress, NetworkInterface, Sockets. |
| [11-Cryptography.md](11-Cryptography.md) | System.Security.Cryptography — RNG, hashing, symmetric, asymmetric, HMAC, KDF. |
| [12-Threading.md](12-Threading.md) | Beyond Tasks: Channel<T>, ThreadPool, AsyncLocal, ExecutionContext, Volatile, Interlocked. |
| [13-MemoryPrimitives.md](13-MemoryPrimitives.md) | Span<T>, Memory<T>, ArrayPool, ReadOnlySequence, ArrayBufferWriter — building blocks for high-perf I/O. |
| [Questions.md](Questions.md) | ~30 drilling questions. |
| [Coding.md](Coding.md) | ~15 problems exercising the BCL. |

---

## Learning objectives

After this chapter you should be able to:
- Pick the right collection for any workload.
- Stream large files with `Pipelines`.
- Configure System.Text.Json for any serialization scenario.
- Generate cryptographically strong random + secure hashes.
- Use `Activity` and `Meter` for OpenTelemetry-compatible observability.
- Reflect (and source-gen-emit) to bridge dynamic and static code.

→ Begin: [01-Strings.md](01-Strings.md)
