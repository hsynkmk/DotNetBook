# Strings & Text

## The BCL's string toolkit

Text handling is one of the most-used and most-misused parts of the BCL. This file maps the platform's text types and the right tool for each job: immutable `string`, mutable `StringBuilder`, allocation-free `Span<char>`/`ReadOnlySpan<char>`, the formatting system, and `string.Create`.

> CSharpBook Chapter 01 §03 covers strings from the language angle (interpolation, raw strings, immutability) and Chapter 09 §12 covers interning. This file is the BCL/API reference and the performance decision guide.

---

## `string` — immutable UTF-16

A `string` is an **immutable** sequence of UTF-16 code units. Immutability means every "modification" allocates a new string:

```csharp
string s = "hello";
s = s + " world";        // allocates a NEW string; the original is unchanged
s = s.ToUpper();          // another allocation
s = s.Replace("L", "x");  // another
```

Immutability gives thread-safety and safe sharing (the JIT/runtime can intern and cache literals — see CSharpBook Ch09 §12), but it's a trap in loops: repeated concatenation is O(n²) allocations.

```csharp
// ✗ — O(n²): each += allocates a new string copying everything so far
string r = "";
foreach (var part in parts) r += part;

// ✓ — StringBuilder amortizes to O(n)
var sb = new StringBuilder();
foreach (var part in parts) sb.Append(part);
string result = sb.ToString();

// ✓ — for a known set, string.Join / string.Concat are simplest and fast
string joined = string.Join(", ", parts);
```

---

## `StringBuilder` — mutable building

`StringBuilder` maintains a growable buffer, so appends don't reallocate the whole string each time:

```csharp
var sb = new StringBuilder(capacity: 256);    // pre-size if you know the rough length
sb.Append("Name: ").Append(name).Append('\n');
sb.AppendLine($"Age: {age}");
sb.AppendFormat("{0:C}", price);
sb.Insert(0, "Header\n");
string result = sb.ToString();                 // single allocation of the final string
```

Use `StringBuilder` when building a string from **many pieces** or in a **loop**. For a small, fixed number of concatenations, `string.Concat`/interpolation/`Join` is clearer and just as fast (the compiler optimizes `a + b + c` into a single `Concat`).

Pre-sizing with `capacity` avoids intermediate buffer growth. `StringBuilder` is pooled-friendly via `Microsoft.Extensions.ObjectPool` for hot paths.

---

## `Span<char>` / `ReadOnlySpan<char>` — allocation-free text

A `ReadOnlySpan<char>` is a **window** over existing string/array/stack memory — slicing and parsing without allocating new strings. This is the modern high-performance text path.

```csharp
ReadOnlySpan<char> line = "key=value".AsSpan();
int eq = line.IndexOf('=');
ReadOnlySpan<char> key = line[..eq];        // no allocation — a view
ReadOnlySpan<char> val = line[(eq + 1)..];  // no allocation

// Parse without allocating substrings:
if (int.TryParse(line[(eq+1)..], out int number)) { ... }
```

Many BCL APIs accept/return spans (`string.AsSpan`, `Split` into spans via `MemoryExtensions`, `int.TryParse(ReadOnlySpan<char>)`). For parsing-heavy code (CSV, protocols, logs), spans eliminate the substring allocations that dominate naive string code. (Span deep dive: CSharpBook Ch09 §05; here it's the text-specific application.)

```csharp
// .NET span-based splitting (no string[] allocation)
foreach (Range r in "a,b,c".AsSpan().Split(','))   // newer Split overloads yield ranges
    Process(line[r]);
```

`Span<char>` is a `ref struct` — can't be stored in fields, captured by lambdas, or used across `await` (use `Memory<char>` there). See [13-MemoryPrimitives.md](13-MemoryPrimitives.md).

---

## The formatting system

### Composite & interpolated formatting

```csharp
string a = string.Format("{0:N2} on {1:d}", amount, date);   // composite
string b = $"{amount:N2} on {date:d}";                        // interpolated (compiler-built)
```

Interpolated strings (`$"..."`) are the modern default. In .NET 6+, the compiler uses an **interpolated string handler** (`DefaultInterpolatedStringHandler`) that builds the result efficiently (often on stack/pooled buffers), avoiding intermediate allocations — and APIs like `ILogger`/`StringBuilder.Append` use custom handlers to avoid formatting work entirely when not needed (CSharpBook Ch10 §11).

### Standard & custom format strings

```csharp
value.ToString("N2");      // 1,234.57   (number, 2 decimals)
value.ToString("C");       // $1,234.57  (currency, culture-dependent)
value.ToString("P1");      // 15.6%      (percent)
date.ToString("O");        // 2026-05-22T14:30:00.0000000Z (round-trip ISO 8601)
date.ToString("yyyy-MM-dd");
n.ToString("X");           // hex
```

**Always pass `CultureInfo` for machine-readable output** (`InvariantCulture` for storage/wire) — see [07-Globalization.md](07-Globalization.md) and CSharpBook Ch13 §08. The classic bug: persisting `1234.56` on a German machine writes `1234,56` and breaks parsing elsewhere.

### `IFormattable` / `ISpanFormattable` / `IUtf8SpanFormattable`

Your types can participate in formatting:
- **`IFormattable`** — `ToString(format, provider)` for custom format strings.
- **`ISpanFormattable`** — format directly into a `Span<char>` (no string allocation).
- **`IUtf8SpanFormattable`** (.NET 8+) — format directly into UTF-8 bytes (for JSON/HTTP without a UTF-16 round-trip).

These let high-performance code format your types straight into buffers.

---

## `string.Create` — build a string in one allocation

When you know the final length and how to fill it, `string.Create` writes directly into the new string's buffer — one allocation, no intermediate `StringBuilder`/`char[]`:

```csharp
// Build "ID-00042" with zero extra allocations beyond the final string
string id = string.Create(8, 42, (span, value) => {
    "ID-".CopyTo(span);
    value.TryFormat(span[3..], out _, "D5");
});
```

The delegate fills the provided `Span<char>`. This is the fastest way to construct a string of known shape (used internally by the BCL for things like `Guid.ToString`).

---

## Parsing — the modern generic way

```csharp
// IParsable<T> / ISpanParsable<T> (generic math era) — uniform parsing
int n = int.Parse("42", CultureInfo.InvariantCulture);
bool ok = double.TryParse(span, NumberStyles.Float, CultureInfo.InvariantCulture, out var d);

// Generic parsing (C# 11+):
static T ParseInvariant<T>(string s) where T : IParsable<T> =>
    T.Parse(s, CultureInfo.InvariantCulture);
```

`IParsable<T>`/`ISpanParsable<T>` (CSharpBook Ch04 §06) give a uniform, generic parsing contract across all numeric and many BCL types. Prefer `TryParse` for untrusted input and **always specify culture + styles** for robustness.

---

## Comparison & search

```csharp
a.Equals(b, StringComparison.Ordinal);            // byte-wise, fast, stable — for identifiers/keys
a.Equals(b, StringComparison.OrdinalIgnoreCase);
a.StartsWith("http", StringComparison.Ordinal);
string.Compare(a, b, StringComparison.CurrentCulture);  // linguistic — for user-facing sort
a.Contains('x');                                   // char overloads are fastest
a.AsSpan().IndexOf("needle");                       // span search, no allocation
```

**Use `Ordinal`/`OrdinalIgnoreCase` for non-linguistic comparisons** (keys, paths, protocol tokens) — faster and immune to the Turkish-I problem; reserve culture-aware comparison for user-visible sorting. See [07-Globalization.md](07-Globalization.md).

---

## Common gotchas

### Concatenation in loops

`+=` in a loop is O(n²) allocations. Use `StringBuilder` or `string.Join`.

### Culture-dependent formatting persisted to storage

`value.ToString()` uses the current culture; persisting it corrupts data across locales. Use `InvariantCulture` for storage/wire, current culture for display.

### Substring allocations in hot parsers

`Substring`/`Split` allocate. Use `AsSpan()` + span slicing/`int.TryParse(span)` for parsing-heavy paths.

### `ReferenceEquals` for string equality

Works only for interned strings; use `==`/`Equals` (CSharpBook Ch09 §12). And use `StringComparison.Ordinal` for identifier comparisons.

### Treating `string.Length` as character count

It's UTF-16 **code units**; an emoji is a surrogate pair (length 2). Use `Rune`/`StringInfo` for true characters (CSharpBook Ch13 §07).

---

## Summary

- `string` is **immutable UTF-16** — great for sharing, bad for loop concatenation (O(n²)); use **`StringBuilder`** (pre-sized) or `string.Join` for building.
- **`ReadOnlySpan<char>`** slices/parses without allocating — the high-performance text path (`AsSpan`, span `TryParse`, span `IndexOf`).
- The **formatting system**: interpolated strings (compiler-built handlers), standard/custom format strings, and `IFormattable`/`ISpanFormattable`/`IUtf8SpanFormattable` for your types — **always pass culture for machine output**.
- **`string.Create`** builds a known-shape string in a single allocation; **`IParsable<T>`** gives uniform generic parsing (`TryParse` + culture for untrusted input).
- Use **`StringComparison.Ordinal`** for keys/identifiers; culture-aware only for user-facing sort.

→ Next: [02-Numerics.md](02-Numerics.md)
