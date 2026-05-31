# Chapter 02 — BCL — Q & A

---

### Q1. Why is repeated string concatenation in a loop O(n²)?

`string` is immutable, so each `+=` allocates a new string copying everything accumulated so far. Over n appends that's quadratic. Use `StringBuilder` (amortized O(n)) or `string.Join`/`Concat` for a fixed set.

---

### Q2. When use `Span<char>`/`ReadOnlySpan<char>` for text?

For slicing and parsing without allocating substrings — the high-performance path (`AsSpan`, span `IndexOf`, `int.TryParse(span)`). But `Span<char>` is stack-only (`ref struct`): no fields, no capture, no `await` — use `Memory<char>` there.

---

### Q3. What does `string.Create` do?

Builds a string of known length in a **single allocation** by writing directly into its buffer via a delegate that fills a `Span<char>` — no intermediate `StringBuilder`/`char[]`. The fastest way to construct a known-shape string.

---

### Q4. Why does `Math.Round(2.5)` return 2?

It defaults to **banker's rounding** (round half to even) to reduce bias in aggregates. For conventional "round half up," pass `MidpointRounding.AwayFromZero`.

---

### Q5. `Random` vs `RandomNumberGenerator` — when each?

`Random` (use `Random.Shared`) is fast but **predictable** — fine for games, simulations, sampling, test data. `RandomNumberGenerator` is a CSPRNG — required for **anything secret** (keys, salts, tokens, nonces). Using `Random` for secrets is a vulnerability.

---

### Q6. How do you compare floats correctly?

Never with `==` (binary FP can't represent 0.1+0.2 exactly). Use a tolerance (`Math.Abs(a-b) < epsilon`) or `decimal` for exact base-10 (money). `NaN` is never equal to itself — check `double.IsNaN`.

---

### Q7. Does default integer arithmetic detect overflow?

No — it **wraps silently** (`int.MaxValue + 1` → negative). Use `checked` (or `<CheckForOverflowUnderflow>`) where overflow is a bug.

---

### Q8. When use a concurrent collection?

Only when **multiple threads** access the same collection instance. Single-threaded or per-request-isolated code should use the plain (faster) collections — don't pay for concurrency you don't have.

---

### Q9. Why might `ConcurrentDictionary.GetOrAdd`'s factory run more than once?

The factory isn't atomic under contention — two threads may both run it, though only one result is stored. If the factory is expensive or has side effects, wrap it in `Lazy<T>` so it runs exactly once per key.

---

### Q10. `BlockingCollection<T>` vs `Channel<T>`?

Both are producer/consumer with bounding. `BlockingCollection` **blocks threads** (synchronous, thread-based). `Channel<T>` is **async** (`await WriteAsync/ReadAsync`, no thread blocking), lower-overhead, and integrates with cancellation — prefer it for async code to avoid thread-pool starvation.

---

### Q11. Can you rely on `Dictionary` preserving insertion order?

No — it happens to today but is **not guaranteed**. Use `OrderedDictionary<TKey,TValue>` (.NET 9+) when insertion order is a requirement (it also gives O(1) lookup + indexed access).

---

### Q12. Immutable vs Frozen vs Concurrent collections?

**Concurrent** — mutable, safe concurrent mutation (shared state changed by many threads). **Immutable** — immutable, "changes" return a new instance (shared, mostly-read, snapshots). **Frozen** — immutable, read-optimized, expensive to build (build once, read very often, like lookup tables).

---

### Q13. What's the cardinal `Stream.Read` rule?

`Read` may return **fewer bytes than requested**; 0 means end of stream. You must loop until you've read enough, or use `ReadExactly`/`CopyToAsync`. Assuming one read returns everything is the #1 stream bug.

---

### Q14. What cross-platform filesystem differences matter?

Path separators (`\` vs `/` — use `Path.Combine`), **case sensitivity** (Linux is case-sensitive), reserved names/chars (Windows), line endings (`\r\n` vs `\n`), and permissions (Unix modes vs Windows ACLs).

---

### Q15. What does `RandomAccess` give you?

Thread-safe, offset-based file I/O **without** a `Stream.Position` — multiple threads can read different offsets of the same file concurrently. Ideal for databases and page-oriented formats.

---

### Q16. Why cache `JsonSerializerOptions`?

It builds and caches per-type metadata internally. Creating a new instance per call rebuilds that cache every time — a major performance bug. Use one static instance.

---

### Q17. Why use System.Text.Json source generation?

It emits serialization code at **compile time** — no reflection, AOT-safe, trimming-safe, faster startup, often faster serialize. Required for Native AOT; declared via a `JsonSerializerContext` partial class.

---

### Q18. Why must you never use `BinaryFormatter`?

It's a **critical security vulnerability** — deserializing untrusted data with it enables remote code execution. It's removed/disabled in modern .NET. Use STJ or a modern binary serializer (MessagePack, protobuf, MemoryPack).

---

### Q19. Why store `DateTimeOffset` over `DateTime` for timestamps?

`DateTimeOffset` always carries the UTC offset (unambiguous instant), while `DateTime`'s `Kind` (especially `Unspecified`) is easy to lose, causing wrong conversions/comparisons. Store UTC (ISO 8601 "O" + invariant culture); convert to a user's zone only for display.

---

### Q20. How do you make time testable?

Inject **`TimeProvider`** instead of calling `DateTime.Now`/`UtcNow` directly. Use `TimeProvider.System` in production and `FakeTimeProvider` (controllable, advanceable) in tests. It also abstracts timers and `Task.Delay`.

---

### Q21. What's the invariant-vs-current-culture rule?

**Invariant culture** for machine-readable data (storage, files, JSON, wire) so it round-trips across locales; **current culture** for user-facing display. Persisting with current culture (e.g., German `1234,56`) is the classic data-corruption bug.

---

### Q22. What is ICU and why does it matter?

International Components for Unicode — modern .NET uses it for culture data on **all platforms**, giving consistent sorting/formatting on Windows, Linux, and macOS (pre-Core, behavior diverged by OS). You can use app-local ICU for version stability.

---

### Q23. Why compare identifiers with `Ordinal`?

It's byte-wise (fast, stable) and immune to the **Turkish-I problem** (where `"i".ToUpper()` is `"İ"` in tr-TR culture). Use `Ordinal`/`OrdinalIgnoreCase` for keys, paths, and extensions; culture-aware comparison only for user-facing sorting.

---

### Q24. What are `Activity`/`ActivitySource` for?

Distributed tracing: an `Activity` is a span (operation with timing, tags, parent link); `ActivitySource` creates them. They nest automatically and propagate across services via W3C Trace Context, forming a distributed trace. This *is* the OpenTelemetry tracing API — `StartActivity` returns null (near-zero cost) when no listener subscribes.

---

### Q25. What's `Meter` for, and what's a cardinality pitfall?

`Meter` creates metric instruments (Counter, Histogram, Gauge) — the metrics half of observability, OpenTelemetry-compatible. Pitfall: tagging metrics with **high-cardinality** values (user/request IDs) explodes the series count and overwhelms backends. Use low-cardinality dimensions (region, status, route template).

---

### Q26. Why use `Stopwatch` instead of `DateTime` subtraction for elapsed time?

`Stopwatch` uses a high-resolution **monotonic** timer, unaffected by clock changes (NTP sync, DST). `DateTime.Now` subtraction can yield negative or wrong durations when the clock jumps.

---

### Q27. Why is reflection slow and how do you mitigate it?

Each operation is metadata lookup + dynamic dispatch + boxing (~hundreds of ns vs ~1 ns direct). Cache `MemberInfo`, compile to typed delegates (`Expression`/`Delegate.CreateDelegate`), or — best — use **source generators** (compile-time, AOT-safe). See CSharpBook Ch12.

---

### Q28. Why never create `HttpClient` per request?

It pools TCP connections and is meant to be long-lived/shared. Per-request creation leaves sockets in `TIME_WAIT` and exhausts the port range under load. Reuse a shared instance, or use `IHttpClientFactory` (which also handles DNS rotation via `PooledConnectionLifetime`).

---

### Q29. Why compare MACs/secrets with `FixedTimeEquals`?

A normal `==`/`SequenceEqual` short-circuits on the first mismatched byte, leaking timing that lets an attacker forge the value byte-by-byte. `CryptographicOperations.FixedTimeEquals` compares in constant time, preventing timing attacks.

---

### Q30. Why can't you hash passwords with SHA-256, and what should you use?

SHA-256 is **fast**, so attackers brute-force billions of guesses/sec. Passwords need a deliberately **slow, salted, per-user KDF** (PBKDF2/Argon2) — or better, use ASP.NET Core Identity which handles it correctly.

---

### Q31. Why prefer AES-GCM over AES-CBC?

AES-GCM is **authenticated encryption (AEAD)** — it encrypts *and* detects tampering. Plain AES-CBC without a MAC is vulnerable to tampering/padding-oracle attacks. With GCM, the **nonce must be unique per message** (reuse is catastrophic).

---

### Q32. What does `AsyncLocal<T>` solve that `ThreadStatic` doesn't?

`ThreadStatic`/`ThreadLocal` don't survive an `await` (the continuation may resume on a different thread). `AsyncLocal<T>` flows ambient data along the logical async call chain (via `ExecutionContext`) — how `Activity.Current`, logging scopes, and culture propagate across awaits.

---

### Q33. `Volatile` vs `Interlocked` vs `lock`?

`Volatile.Read/Write` prevents reordering of a **single field** access (e.g., a stop flag). `Interlocked` does **atomic** read-modify-write (increment, CAS) without a lock. `lock` provides general mutual exclusion. For anything beyond a flag/counter, prefer `lock` — the memory model is subtle.

---

### Q34. Why `PeriodicTimer` over the callback `Timer`?

`PeriodicTimer.WaitForNextTickAsync` is async (no thread blocked), naturally prevents **overlapping** ticks (you control the loop), and respects cancellation. The callback `Timer` can re-enter if work outlasts the period.

---

### Q35. `Span<T>` vs `Memory<T>` — when each?

`Span<T>` for **synchronous, stack-local** hot paths — zero-copy, zero-alloc, but stack-only (no fields/capture/`await`). `Memory<T>` when you must **store** the window or use it **across `await`** (async I/O APIs take `Memory<T>`); call `.Span` for synchronous access.

---

### Q36. What are the rules for `ArrayPool<T>`?

`Rent(n)` may return a **larger** array (use the length you asked for, not `.Length`); **always `Return`** in `finally` (or it degrades to allocation); contents are **not cleared** by default (clear sensitive data); never use a buffer after returning it.

---

→ Next: [Coding.md](Coding.md)
