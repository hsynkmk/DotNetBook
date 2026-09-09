# 02 — Base Class Library Essentials

## ⚡ 30-second answer

The BCL is the standard library every .NET app builds on. The interview-relevant parts: `string`
is **immutable UTF-16** so loop concatenation is O(n²) — use `StringBuilder`, and
**`StringComparison.Ordinal`** for identifiers. **`Span<T>`/`Memory<T>`** are zero-copy windows
over memory and the allocation-free fast path for parsing. **System.Text.Json** is the default
serializer (cache `JsonSerializerOptions`, source-generate for AOT), and **`BinaryFormatter` is
removed — it was a remote-code-execution hole**. For time, default to **`DateTimeOffset`**, store
**UTC**, and inject **`TimeProvider`** so it's testable. For randomness, **`Random` for games,
`RandomNumberGenerator` for anything security-related** — never confuse them. And
`ActivitySource`/`Meter` are the vendor-neutral tracing and metrics primitives that cost nothing
when nobody's listening.

---

## Core mechanics

### Strings and text

```csharp
// ❌ O(n²) — each += allocates a new string and copies everything
foreach (var x in items) s += x;

// ✅
var sb = new StringBuilder(estimatedLength);      // pre-size when you can
foreach (var x in items) sb.Append(x);
var result = string.Join(", ", items);            // simpler still when it fits
```

- **`ReadOnlySpan<char>`** slices and parses without allocating — `AsSpan()`, span `TryParse`,
  span `IndexOf`. This is the high-performance text path.
- **`string.Create`** builds a string of known length in a single allocation.
- **`IParsable<T>`/`ISpanFormattable`** give uniform generic parsing and formatting for your own
  types.
- **Always pass culture for machine-readable output** — `InvariantCulture` for storage and the
  wire, current culture only for display.

### Comparison and globalization

| Comparison | Use for |
|---|---|
| **`Ordinal`** | identifiers, keys, file paths, protocol tokens — fast, exact |
| **`OrdinalIgnoreCase`** | case-insensitive identifiers (dodges the Turkish-I bug) |
| `CurrentCulture` | sorting text a human will read |
| `InvariantCulture` | rarely — legacy interop |

The **Turkish-I problem**: in `tr-TR`, `"I".ToLower()` is `"ı"` (dotless), so a culture-aware
comparison of `"FILE"` and `"file"` fails. This is why identifier comparison must be ordinal.
Modern .NET uses **ICU on all platforms** for consistent culture behavior;
`InvariantGlobalization` mode drops ICU entirely for smaller, faster non-localized services.

### Numerics

- **Never `==` on floating point** — use a tolerance, and use **`decimal` for money**. `NaN` is
  not equal to itself.
- **Integer overflow wraps silently** unless you opt into `checked`.
- `Math.Round` uses **banker's rounding** (half-to-even) by default — `Math.Round(2.5)` is `2`.
- **Generic math** (`INumber<T>`) writes type-agnostic numeric code with no boxing.
- Two random number generators, and the distinction is a security question:

| | `Random.Shared` | `RandomNumberGenerator` |
|---|---|---|
| Kind | fast PRNG, predictable | **CSPRNG** |
| Use for | games, simulations, sampling, tests | **tokens, keys, salts, nonces, passwords** |

### Collections beyond the basics

| Family | For |
|---|---|
| **Concurrent** (`ConcurrentDictionary`, …) | genuine multi-threaded mutation — *slower* single-threaded |
| **Immutable** | shared snapshots, safe publication |
| **Frozen** (`FrozenDictionary`) | build once, read many — fastest lookups |
| `OrderedDictionary<K,V>` (.NET 9+) | insertion order **and** O(1) lookup |
| `PriorityQueue` | binary heap — **not stable** |
| `BitArray` | packed bits, 8× denser than `bool[]` |

`ConcurrentDictionary.GetOrAdd`'s **factory is not atomic** — it can run more than once for the
same key. Wrap the value in `Lazy<T>` if the factory is expensive or has side effects.

Read-only views (`AsReadOnly()`, `IReadOnlyList<T>`) block *your* mutation but are **not
snapshots** — the underlying collection can still change under you. Copy for independence.

### Memory primitives

```csharp
ReadOnlySpan<char> span = text.AsSpan(5, 10);     // no allocation, no copy

var buffer = ArrayPool<byte>.Shared.Rent(4096);
try { … }
finally { ArrayPool<byte>.Shared.Return(buffer); }   // always in finally
```

- **`Span<T>`** is a `ref struct`: **stack-only**. No fields on a class, no capture in a lambda,
  **no `await` across it**. That restriction is the whole reason `Memory<T>` exists.
- **`Memory<T>`** is the heap-storable, async-capable counterpart — async I/O APIs take
  `Memory<T>`; call `.Span` for synchronous work.
- **`ArrayPool<T>`** eliminates per-operation allocations. It may return a **larger** array than
  requested, and the contents are **not zeroed**.
- `ReadOnlySequence<T>` + `SequenceReader<T>` handle multi-segment buffers (Pipelines);
  `IBufferWriter<T>` is the write side. Together these are the allocation-free pipeline behind
  Kestrel and System.Text.Json.

### Serialization

```csharp
private static readonly JsonSerializerOptions Options = new(JsonSerializerDefaults.Web);
// ↑ cache it — constructing options per call rebuilds the whole metadata cache
```

Four API levels: `JsonSerializer` (objects) → `JsonNode` (mutable DOM) → `JsonDocument`
(read-only DOM) → `Utf8JsonReader/Writer` (streaming, fastest). **Source generation**
(`JsonSerializerContext`) removes reflection — required for AOT, faster everywhere.

**`BinaryFormatter` is removed from .NET** — it deserialized arbitrary types and was a reliable
RCE vector. For binary use MessagePack, protobuf, or MemoryPack. **Treat all deserialized data as
untrusted**: allow-list polymorphic types, bound depth and size.

### Date and time

```csharp
public sealed class OrderService(TimeProvider clock)      // ✅ injectable, fakeable
{
    public Order Create() => new(clock.GetUtcNow());
}
```

| Type | For |
|---|---|
| **`DateTimeOffset`** | **timestamps** — unambiguous, carries the offset |
| `TimeSpan` | durations |
| `DateOnly` / `TimeOnly` | a date or a time with no other half |
| `DateTime` | legacy; if you must, keep it **UTC** |

**Store UTC**, serialize ISO 8601 (`"O"` + `InvariantCulture`), convert to a user's zone with
`TimeZoneInfo` **only for display**. Use `UtcNow`, never `Now`, for logic — `Now` does a timezone
lookup and lands you in DST ambiguity.

### Diagnostics primitives

```csharp
private static readonly ActivitySource Source = new("MyApp.Orders");   // static, created once
using var activity = Source.StartActivity("PlaceOrder");
```

`ActivitySource`/`Activity` (spans), `Meter` (counters/histograms/gauges), and `EventSource`
(low-overhead structured events) are **vendor-neutral and OpenTelemetry-compatible**. They cost
essentially **nothing when no listener subscribes** — `StartActivity` returns `null` — so
instrument freely. Activities nest via `Activity.Current` (an `AsyncLocal`) and propagate across
services via **W3C Trace Context** ([12](12-Observability.md)).

Use **`Stopwatch`** (monotonic) for elapsed time — never subtract `DateTime`s.

### Threading primitives

- **`Channel<T>`** — the modern async producer/consumer; bounded channels give you back-pressure.
  Prefer it over `BlockingCollection<T>` ([08](08-BackgroundProcessing.md)).
- **`AsyncLocal<T>`** flows ambient data across `await` (via `ExecutionContext`) — this is how
  `Activity.Current`, logging scopes and culture propagate. **`ThreadStatic`/`ThreadLocal` do
  not flow across awaits.**
- **`Interlocked`** for atomic operations and CAS; **`Volatile`** to prevent reordering of a
  single field access. For anything non-trivial, take a `lock`.
- **`PeriodicTimer`** — async, non-overlapping periodic work.
- Use async synchronization in async code (`SemaphoreSlim.WaitAsync`), never `lock` around an
  `await` (it won't compile) or `Monitor` held across one.

### Cryptography

**Don't invent crypto.** Prefer high-level APIs (ASP.NET Core Data Protection, Identity) over
assembling primitives ([10](10-Identity-Security.md)).

| Need | Use |
|---|---|
| Random for security | `RandomNumberGenerator` — **never `Random`** |
| Integrity | SHA-256 or better |
| Keyed authentication | HMAC — compare with **`CryptographicOperations.FixedTimeEquals`** |
| **Passwords** | a slow salted KDF: **PBKDF2, bcrypt, Argon2** — never a fast hash |
| Symmetric encryption | **AEAD**: AES-GCM or ChaCha20-Poly1305, **unique nonce per message** |
| Signatures / key exchange | ECDSA, RSA |

---

## 🪤 Traps & gotchas

- **String `+=` in a loop** — O(n²) allocations and copies. `StringBuilder` or `string.Join`.
- **Culture-sensitive comparison for identifiers** — the Turkish-I bug makes `"FILE"` and
  `"file"` compare unequal under `tr-TR`. Use `StringComparison.Ordinal`.
- **Formatting numbers or dates without a culture** for storage or the wire — `1.5` becomes
  `"1,5"` in a German locale and your CSV or JSON breaks in production only.
- **`==` on `double`** — accumulated representation error means `0.1 + 0.2 != 0.3`. Money is
  `decimal`.
- **`NaN != NaN`** — so a `NaN` in a sort comparator or a `Distinct()` behaves bizarrely.
- **Silent integer overflow** — unchecked by default, so a counter wraps to negative rather than
  throwing.
- **`Math.Round(2.5)` returns 2** — banker's rounding. Pass `MidpointRounding.AwayFromZero` if you
  meant the schoolbook rule.
- **Using `Random` for tokens, session ids or password resets** — predictable output, a real
  vulnerability. `RandomNumberGenerator`.
- **A fast hash (SHA-256) for passwords** — GPUs do billions per second. Use a deliberately slow
  KDF with a per-user salt.
- **Comparing HMACs or tokens with `==`** — short-circuits on the first differing byte, leaking
  the answer through timing. `FixedTimeEquals`.
- **Reusing an AES-GCM nonce** with the same key catastrophically breaks confidentiality *and*
  authenticity.
- **`ConcurrentDictionary.GetOrAdd` factory running twice** — it isn't atomic. Wrap in `Lazy<T>`
  when the factory is expensive or has side effects.
- **Using concurrent collections single-threaded** — you pay synchronization for nothing; the
  plain collections are faster.
- **Relying on `Dictionary` enumeration order** — undefined and it has changed between versions.
  `OrderedDictionary<K,V>` if you need insertion order.
- **Treating `AsReadOnly()` as a snapshot** — it's a live view; the source can still mutate.
- **Capturing a `Span<T>` in a lambda or across an `await`** — it won't compile, and the reason is
  worth stating: it's a `ref struct` that may point at the stack.
- **Not returning a pooled array in `finally`** — an exception leaks it out of the pool forever.
  And pooled arrays are **larger than requested and not zeroed**, so `buffer.Length` is a bug.
- **Constructing `JsonSerializerOptions` per call** — it rebuilds the reflection metadata cache
  every time. Cache a static instance.
- **`BinaryFormatter`** — removed, and it was remote code execution by design.
- **Deserializing untrusted polymorphic JSON** without an allow-list of types.
- **`DateTime.Now` in business logic** — server timezone dependent, ambiguous across DST
  transitions. `UtcNow`, or better, `TimeProvider`.
- **`DateTime.Kind == Unspecified`** — the value silently means different instants depending on
  which code touches it next. `DateTimeOffset`.
- **Computing durations by subtracting `DateTime`s** — NTP adjustments and DST can make elapsed
  time negative. `Stopwatch` is monotonic.
- **`ThreadStatic` for request context** — it doesn't flow across `await`, so it's empty after the
  first continuation. `AsyncLocal<T>`.
- **Ignoring the `Stream.Read` contract** — it may return fewer bytes than asked for. Loop, or use
  `ReadExactlyAsync`/`CopyToAsync`.
- **`new HttpClient()` per request** — socket exhaustion ([09](09-Http-gRPC-SignalR.md)).
- **Reflection in a hot loop** — cache the `MemberInfo`, compile a delegate, or use a source
  generator.
- **High-cardinality metric tags** (user id, request id) — a cardinality explosion that will take
  down your metrics backend, not your app.

---

## ❓ Likely questions

**Q: Why is string concatenation in a loop bad, and what do you use instead?**
A: `string` is immutable, so each `+=` allocates a new string and copies the whole thing — O(n²)
overall. `StringBuilder` (pre-sized if you know the length) or `string.Join`.

**Q: When do you use `StringComparison.Ordinal`?**
A: For identifiers, keys, paths and protocol tokens — anything not being shown to a human. It's
faster and exact, and it avoids culture bugs like the Turkish-I, where `"I".ToLower()` isn't `"i"`
in `tr-TR`. Culture-aware comparison is only for sorting text a user reads.

**Q: What is `Span<T>` and why can't you `await` across one?**
A: A zero-copy, zero-allocation window over contiguous memory. It's a `ref struct`, so it lives on
the stack only — it can't be a field, captured in a closure, or held across an `await`, because
the continuation may resume on a different stack. `Memory<T>` is the heap-storable version for
exactly that case.

**Q: What does `ArrayPool<T>` give you, and what are its sharp edges?**
A: Reuse of buffers instead of allocating per operation — a large win on hot paths and for
avoiding LOH churn. The edges: return in a `finally`, the returned array may be **bigger** than you
asked for (so track your own length), and it is **not cleared**, so it may contain another
operation's data.

**Q: Why cache `JsonSerializerOptions`?**
A: Each instance builds and owns a metadata cache of your types via reflection. Creating one per
call rebuilds it every time and destroys serialization throughput.

**Q: Why was `BinaryFormatter` removed?**
A: It deserialized whatever type the payload named, which made any untrusted input a remote-code-
execution vector. There was no safe way to use it on untrusted data. Use System.Text.Json, or
MessagePack/protobuf for binary.

**Q: `DateTime` vs `DateTimeOffset` — which and why?**
A: `DateTimeOffset` for timestamps, because it carries the UTC offset and is unambiguous.
`DateTime` has a `Kind` that is frequently `Unspecified`, which means the same value silently
denotes different instants to different code. Store UTC, convert to local only for display.

**Q: How do you make time testable?**
A: Inject **`TimeProvider`** rather than calling `DateTime.UtcNow` directly, and use
`FakeTimeProvider` in tests. It also fakes timers and delays, so you can test a scheduler without
sleeping.

**Q: `Random` vs `RandomNumberGenerator`?**
A: `Random` is a fast, seeded, **predictable** PRNG — fine for games, simulation and sampling.
`RandomNumberGenerator` is a cryptographically secure generator, and it's the only acceptable
source for tokens, keys, salts, nonces and password reset codes.

**Q: How do you store passwords?**
A: A deliberately slow key-derivation function with a per-user salt — PBKDF2, bcrypt, or Argon2 —
with a work factor tuned to your hardware. Never a fast hash like SHA-256, which a GPU computes
billions of times per second.

**Q: Why compare secrets with `FixedTimeEquals`?**
A: An ordinary comparison returns as soon as bytes differ, so how long it took reveals how many
leading bytes were correct. An attacker can recover a token byte by byte. Constant-time
comparison removes that channel.

**Q: What's the difference between `AsyncLocal` and `ThreadStatic`?**
A: `AsyncLocal<T>` flows with the `ExecutionContext` across `await` boundaries — which is how
`Activity.Current` and logging scopes work. `ThreadStatic`/`ThreadLocal` are per-thread, so after
the first `await` resumes on a different pool thread, the value is gone.

**Q: When is `ConcurrentDictionary` the wrong choice?**
A: When only one thread touches it — you pay for synchronization you don't need. Also note
`GetOrAdd`'s factory isn't atomic and may run more than once concurrently, so wrap expensive or
side-effecting factories in `Lazy<T>`.

**Q: What do `ActivitySource` and `Meter` cost when unused?**
A: Essentially nothing — `StartActivity` returns `null` when no listener has subscribed, and
metric instruments no-op. That's deliberate, so libraries can be instrumented unconditionally.

**Q: Why `Stopwatch` instead of subtracting `DateTime`s?**
A: `Stopwatch` is monotonic. Wall-clock time can jump backwards from an NTP correction or a DST
change, which makes a measured duration negative.

---

## 🎓 Senior Extra

- **`string.Create` and `ISpanFormattable`** let you build a string in exactly one allocation when
  you know the length — the pattern behind most of the BCL's own fast formatting.
- **UTF-8 all the way down**: `u8` literals, `IUtf8SpanFormattable`, and `Utf8JsonWriter` avoid the
  UTF-16 round-trip entirely. On a JSON API that transcoding is a measurable share of CPU.
- **`FrozenDictionary`/`FrozenSet`** pay a higher build cost to produce a faster read-only lookup —
  ideal for configuration-derived maps built once at startup and read on every request.
- **`SearchValues<T>`** (.NET 8+) vectorizes "does this span contain any of these characters",
  which is the hot loop in most tokenizers and validators.
- **`ExecutionContext` is what makes `AsyncLocal` work**, and it's captured on every async
  operation — which is also why an unbounded number of `AsyncLocal` values is a real cost, and why
  `ExecutionContext.SuppressFlow` exists.
- **Pipelines (`System.IO.Pipelines`)** solves the problem `Stream` doesn't: buffer management and
  partial-message parsing for protocols. It's what Kestrel is built on, and the right tool when
  you're framing your own wire protocol.
- **`TimeProvider` also fakes timers and `Task.Delay`**, so it's not only about "now" — it turns a
  test of retry backoff or a scheduler from a sleepy integration test into a fast deterministic
  unit test.
- **`InvariantGlobalization=true`** drops ICU: smaller container, faster startup, and every
  culture-aware operation silently becomes invariant. Excellent for a non-localized backend
  service, quietly wrong for anything that formats for humans.
- **`CryptographicOperations.ZeroMemory`** clears key material, though the honest caveat is that
  the GC may have copied it during a compaction — which is why `SecureString` was abandoned.
  .NET 10 adds post-quantum algorithms (ML-KEM, ML-DSA).
- **`EventSource` + EventPipe** is the mechanism behind `dotnet-counters` and `dotnet-trace`;
  writing your own `EventSource` gives you production diagnostics at near-zero cost when nobody is
  listening ([21](21-Performance-Tooling.md)).

→ Deeper: [`../02-BCL/`](../02-BCL/README.md)
