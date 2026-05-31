# Chapter 02 — BCL — Coding Problems

Exercises across the standard library: text/parsing, numerics, collections, I/O, serialization, time, diagnostics, crypto, and memory primitives.

---

## Problem 1: Allocation-free key=value parser

Parse `"key=value"` lines into a dictionary without allocating substrings during the split.

<details><summary>Solution</summary>

```csharp
Dictionary<string, string> Parse(ReadOnlySpan<char> input) {
    var result = new Dictionary<string, string>(StringComparer.Ordinal);
    foreach (Range lineRange in input.Split('\n')) {            // span Split → ranges, no string[]
        ReadOnlySpan<char> line = input[lineRange].Trim();
        int eq = line.IndexOf('=');
        if (eq < 0) continue;
        // Only the final stored keys/values allocate (dictionary needs strings):
        result[new string(line[..eq].Trim())] = new string(line[(eq + 1)..].Trim());
    }
    return result;
}
```

Slicing via spans avoids intermediate substring allocations; only the strings actually stored allocate. `StringComparer.Ordinal` for fast, stable key comparison. (Spans: [01-Strings.md](01-Strings.md).)

</details>

---

## Problem 2: Generic numeric average

Write `Average<T>` that works for any numeric type using generic math.

<details><summary>Solution</summary>

```csharp
using System.Numerics;

static T Average<T>(ReadOnlySpan<T> values) where T : INumber<T> {
    if (values.IsEmpty) return T.Zero;
    T sum = T.Zero;
    foreach (var v in values) sum += v;
    return sum / T.CreateChecked(values.Length);    // CreateChecked converts the int length to T
}

Average<double>([1.0, 2.0, 3.0]);   // 2.0
Average<int>([2, 4, 6]);            // 4 (integer division)
Average<decimal>([1m, 2m]);         // 1.5
```

`INumber<T>` gives `T.Zero`, `+`, `/`, and `T.CreateChecked` for cross-type conversion — one method for all numeric types, no boxing. ([02-Numerics.md](02-Numerics.md).)

</details>

---

## Problem 3: Thread-safe cache with exactly-once factory

Build a `ConcurrentDictionary` cache where the expensive load runs **once** per key even under contention.

<details><summary>Solution</summary>

```csharp
public class Cache<TKey, TValue> where TKey : notnull {
    private readonly ConcurrentDictionary<TKey, Lazy<TValue>> _store = new();

    public TValue GetOrAdd(TKey key, Func<TKey, TValue> factory) =>
        _store.GetOrAdd(key, k => new Lazy<TValue>(() => factory(k))).Value;
}
```

`GetOrAdd`'s factory isn't atomic, so multiple threads might create competing `Lazy<TValue>` wrappers — but only one wins, and `.Value` ensures the *expensive* factory runs exactly once (other threads block on the same `Lazy`). ([03-CollectionsDeep.md](03-CollectionsDeep.md).)

</details>

---

## Problem 4: Read a large file with bounded memory

Count error lines in a multi-GB log without loading it into memory, async.

<details><summary>Solution</summary>

```csharp
async Task<long> CountErrorsAsync(string path, CancellationToken ct) {
    long count = 0;
    await foreach (var line in File.ReadLinesAsync(path, ct))      // lazy, constant memory
        if (line.Contains("ERROR", StringComparison.Ordinal)) count++;
    return count;
}
```

`File.ReadLinesAsync` streams lazily — constant memory regardless of file size — and async frees the thread during disk waits. `Ordinal` comparison is fastest. ([04-IO.md](04-IO.md).)

</details>

---

## Problem 5: AOT-safe JSON round-trip with source generation

Set up source-generated serialization for a record so it works under Native AOT.

<details><summary>Solution</summary>

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

public record Order(int Id, string Customer, decimal Total, List<string> Items);

[JsonSourceGenerationOptions(PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase)]
[JsonSerializable(typeof(Order))]
public partial class AppJsonContext : JsonSerializerContext { }

string json = JsonSerializer.Serialize(order, AppJsonContext.Default.Order);
Order? back = JsonSerializer.Deserialize(json, AppJsonContext.Default.Order);
```

No reflection — works under Native AOT, faster startup. The reflection-based `Serialize(order)` overload would warn (`IL2026`/`IL3050`) under AOT. ([05-Serialization.md](05-Serialization.md).)

</details>

---

## Problem 6: Store and display a timestamp correctly across zones

Persist an event time, then display it in the user's zone.

<details><summary>Solution</summary>

```csharp
// Store — UTC, ISO 8601, invariant culture
DateTimeOffset now = DateTimeOffset.UtcNow;
string forStorage = now.ToString("O", CultureInfo.InvariantCulture);

// Read back — round-trips exactly
var loaded = DateTimeOffset.Parse(forStorage, CultureInfo.InvariantCulture, DateTimeStyles.RoundtripKind);

// Display — convert to the user's zone at the presentation layer only
var userZone = TimeZoneInfo.FindSystemTimeZoneById("America/New_York");
DateTimeOffset local = TimeZoneInfo.ConvertTime(loaded, userZone);
Console.WriteLine(local.ToString("f", new CultureInfo("en-US")));
```

Store UTC + ISO 8601 + invariant; convert to a named zone only for display. UTC has no DST gaps/ambiguity. ([06-DateTimeAndTime.md](06-DateTimeAndTime.md).)

</details>

---

## Problem 7: Make a time-dependent class testable

Refactor `IsExpired` to be unit-testable, then test it.

```csharp
public class Token { public DateTimeOffset Expiry { get; init; } }
public bool IsExpired(Token t) => DateTimeOffset.UtcNow > t.Expiry;   // untestable
```

<details><summary>Solution</summary>

```csharp
public class TokenService(TimeProvider clock) {
    public bool IsExpired(Token t) => clock.GetUtcNow() > t.Expiry;
}

// Test (Microsoft.Extensions.TimeProvider.Testing)
[Fact]
public void IsExpired_AfterExpiry_True() {
    var fake = new FakeTimeProvider();
    fake.SetUtcNow(new DateTimeOffset(2030, 1, 1, 0, 0, 0, TimeSpan.Zero));
    var svc = new TokenService(fake);
    Assert.True(svc.IsExpired(new Token { Expiry = new DateTimeOffset(2029, 1, 1, 0, 0, 0, TimeSpan.Zero) }));
    fake.Advance(TimeSpan.FromDays(365));   // advance time deterministically
}
```

Inject `TimeProvider`; use `FakeTimeProvider` to control "now" in tests. ([06-DateTimeAndTime.md](06-DateTimeAndTime.md).)

</details>

---

## Problem 8: Instrument an operation with a trace and a metric

Add a span and a duration histogram to an operation.

<details><summary>Solution</summary>

```csharp
using System.Diagnostics;
using System.Diagnostics.Metrics;

public class OrderProcessor {
    private static readonly ActivitySource Source = new("Shop.Orders", "1.0");
    private static readonly Meter Meter = new("Shop.Orders", "1.0");
    private static readonly Histogram<double> Duration = Meter.CreateHistogram<double>("orders.duration", "ms");
    private static readonly Counter<long> Placed = Meter.CreateCounter<long>("orders.placed");

    public async Task<Order> PlaceAsync(OrderRequest req) {
        using var activity = Source.StartActivity("PlaceOrder");
        activity?.SetTag("customer", req.CustomerId);
        var sw = Stopwatch.StartNew();
        try {
            var order = await ProcessAsync(req);
            Placed.Add(1, new KeyValuePair<string, object?>("region", req.Region));
            activity?.SetStatus(ActivityStatusCode.Ok);
            return order;
        } catch (Exception ex) {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            throw;
        } finally {
            Duration.Record(sw.Elapsed.TotalMilliseconds);
        }
    }
}
```

`ActivitySource`/`Meter` are static (created once), instrumentation is near-free when no listener subscribes, and a low-cardinality `region` tag slices the metric. OpenTelemetry-compatible. ([08-Diagnostics.md](08-Diagnostics.md).)

</details>

---

## Problem 9: Generate a secure token and verify an HMAC safely

Create a secure token, then verify a signature in constant time.

<details><summary>Solution</summary>

```csharp
using System.Security.Cryptography;

// Secure token (NOT Random)
string token = Convert.ToHexString(RandomNumberGenerator.GetBytes(32));   // 256-bit

// HMAC-sign a message and verify safely
byte[] key = RandomNumberGenerator.GetBytes(32);
byte[] Sign(byte[] msg) => HMACSHA256.HashData(key, msg);

bool Verify(byte[] msg, byte[] providedMac) {
    byte[] expected = HMACSHA256.HashData(key, msg);
    return CryptographicOperations.FixedTimeEquals(expected, providedMac);   // constant-time!
}
```

`RandomNumberGenerator` (not `Random`) for the token; `FixedTimeEquals` (not `==`) for the MAC comparison to prevent timing attacks. ([11-Cryptography.md](11-Cryptography.md).)

</details>

---

## Problem 10: Async producer/consumer with back-pressure

Build a bounded `Channel<T>` pipeline: one producer, multiple consumers.

<details><summary>Solution</summary>

```csharp
using System.Threading.Channels;

async Task RunPipelineAsync(IEnumerable<int> source, CancellationToken ct) {
    var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(100) {
        FullMode = BoundedChannelFullMode.Wait
    });

    // Producer
    var producer = Task.Run(async () => {
        foreach (var item in source) await channel.Writer.WriteAsync(item, ct);  // back-pressure when full
        channel.Writer.Complete();
    }, ct);

    // 4 consumers draining concurrently
    var consumers = Enumerable.Range(0, 4).Select(_ => Task.Run(async () => {
        await foreach (var item in channel.Reader.ReadAllAsync(ct))
            await ProcessAsync(item);
    }, ct)).ToArray();

    await Task.WhenAll(consumers.Append(producer));
}
```

Bounded channel gives back-pressure (producer awaits when full — no unbounded memory), all async (no blocked threads), multiple consumers share the reader. ([12-Threading.md](12-Threading.md).)

</details>

---

## Problem 11: Zero-allocation buffer processing with ArrayPool

Read a stream in chunks using a pooled buffer.

<details><summary>Solution</summary>

```csharp
using System.Buffers;

async Task<long> ProcessStreamAsync(Stream stream, CancellationToken ct) {
    byte[] buffer = ArrayPool<byte>.Shared.Rent(81920);
    try {
        long total = 0;
        int read;
        while ((read = await stream.ReadAsync(buffer.AsMemory(0, 81920), ct)) > 0) {  // loop! Read may be partial
            ProcessChunk(buffer.AsSpan(0, read));    // use only the bytes read
            total += read;
        }
        return total;
    } finally {
        ArrayPool<byte>.Shared.Return(buffer);       // always return
    }
}
```

Pooled buffer (no per-call allocation), `Memory` for the async read, `Span` for synchronous processing, the `Read`-may-be-partial loop, and `Return` in `finally`. ([13-MemoryPrimitives.md](13-MemoryPrimitives.md).)

</details>

---

## Problem 12: Fix the culture-corruption bug

This breaks when run on a non-US machine. Fix it.

```csharp
void Save(decimal price) => File.WriteAllText("p.txt", price.ToString());
decimal Load() => decimal.Parse(File.ReadAllText("p.txt"));
```

<details><summary>Solution</summary>

```csharp
void Save(decimal price) =>
    File.WriteAllText("p.txt", price.ToString(CultureInfo.InvariantCulture));
decimal Load() =>
    decimal.Parse(File.ReadAllText("p.txt"), CultureInfo.InvariantCulture);
```

On a German machine `1234.56.ToString()` writes `"1234,56"`; parsing elsewhere misreads it. Always `InvariantCulture` for persistence; current culture only for display. ([07-Globalization.md](07-Globalization.md).)

</details>

---

These exercises span the BCL: span-based parsing, generic math, concurrent caching, streaming I/O, source-gen JSON, correct time handling and testability, observability instrumentation, secure crypto, async channels, pooled buffers, and the culture trap. Next, the platform's application backbone.

→ Back to [Chapter 02 README](README.md) · Next chapter: [Chapter 03 — Hosting & DI](../03-HostingAndDI/README.md)
