# Globalization & Encoding

## Making software work across cultures and character sets

Globalization covers culture-specific formatting/parsing (numbers, dates, casing, sorting) and **encoding** (the bytes that represent text). Get it wrong and you corrupt data across locales, produce mojibake, or hit the Turkish-I bug.

> CSharpBook Chapter 13 §07–08 covers `Encoding`/BOM and `CultureInfo`/comparison in depth (the "invariant for machines, current for humans" rule, the Turkish-I problem, normalization). This file adds the **BCL/platform specifics**: ICU, region info, `Encoding` families, and globalization-invariant mode.

---

## The golden rule (recap)

| Scenario | Culture |
|---|---|
| Store / parse machine data (files, DB, JSON, wire) | **`CultureInfo.InvariantCulture`** |
| Display to a user | **`CultureInfo.CurrentCulture`** (their locale) |
| Compare identifiers / keys / paths | **`StringComparison.Ordinal[IgnoreCase]`** |
| Sort a user-visible list | culture-aware `StringComparer` |

The classic bug: persisting `1234.56` on a German machine writes `1234,56`, which parses wrong elsewhere. Always `InvariantCulture` for persistence. (Full treatment: CSharpBook Ch13 §08.)

---

## `CultureInfo` and the ambient cultures

```csharp
using System.Globalization;

CultureInfo.CurrentCulture;            // formatting/parsing (numbers, dates)
CultureInfo.CurrentUICulture;          // which localized resources load (translations)
CultureInfo.InvariantCulture;          // culture-neutral baseline (English-like, '.' decimals)

// Set app-wide (modern .NET):
CultureInfo.DefaultThreadCurrentCulture = new CultureInfo("en-US");
CultureInfo.DefaultThreadCurrentUICulture = new CultureInfo("en-US");

// Cached lookup (avoid `new CultureInfo` in hot paths):
var de = CultureInfo.GetCultureInfo("de-DE");
```

Two ambient cultures per thread: **`CurrentCulture`** drives formatting/parsing; **`CurrentUICulture`** drives resource (translation) lookup. ASP.NET Core sets these per-request via request localization middleware ([Ch04](../04-AspNetCore/README.md)).

---

## ICU — the engine underneath

Modern .NET uses **ICU** (International Components for Unicode) for culture data on **all platforms** (Windows 10+, Linux, macOS), replacing the old Windows-specific NLS. This means:
- **Consistent culture behavior across platforms** — the same sorting/formatting on Linux and Windows.
- ICU is loaded from the OS (Linux/macOS) or shipped (`icu.dll` / app-local ICU) on Windows.
- You can opt into **app-local ICU** for version stability (so a Linux distro's ICU upgrade doesn't silently change your sort order).

This cross-platform consistency is a major modern-.NET improvement — pre-Core, culture behavior diverged between OSes.

---

## `RegionInfo` — country/region data

Where `CultureInfo` is language+region, `RegionInfo` is region-specific data:

```csharp
var region = new RegionInfo("US");
region.CurrencySymbol;          // "$"
region.ISOCurrencySymbol;       // "USD"
region.IsMetric;                // false
region.TwoLetterISORegionName;  // "US"
```

Useful for currency, measurement system, and region metadata independent of language.

---

## Encoding — bytes ↔ text

A `string` is **UTF-16 in memory**; encoding matters only when crossing to bytes (files, network):

```csharp
byte[] utf8 = Encoding.UTF8.GetBytes("héllo");      // 6 bytes (é is 2 bytes in UTF-8)
string back = Encoding.UTF8.GetString(utf8);

Encoding.UTF8        // the universal default — ASCII-compatible, compact
Encoding.Unicode     // UTF-16 LE (.NET's in-memory format)
Encoding.Latin1      // ISO-8859-1 (.NET 5+)
Encoding.ASCII       // 0–127 only; loses non-ASCII
```

**Default to UTF-8 without a BOM** for files/interop:

```csharp
var noBom = new UTF8Encoding(encoderShouldEmitUTF8Identifier: false);
await File.WriteAllTextAsync(path, content, noBom);
```

The BOM (`EF BB BF` for UTF-8) is optional and breaks shell scripts, some JSON parsers, and file concatenation — write without it, read with detection (StreamReader detects/strips BOMs by default). (Full encoding + BOM + `Rune` discussion: CSharpBook Ch13 §07.)

### Legacy code pages

.NET Core dropped most legacy code pages to shrink the runtime. To read Windows-1252 etc., register the provider:

```csharp
Encoding.RegisterProvider(CodePagesEncodingProvider.Instance);   // System.Text.Encoding.CodePages
var win1252 = Encoding.GetEncoding(1252);
```

### UTF-8 string literals (`u8`)

For constant byte sequences (HTTP headers, protocol tokens), bake UTF-8 bytes at compile time:

```csharp
ReadOnlySpan<byte> contentType = "application/json"u8;   // no runtime GetBytes, zero alloc
```

(CSharpBook Ch10 §10.)

---

## Casing & comparison — the Turkish-I trap

```csharp
// ✗ — culture-sensitive casing breaks on tr-TR ("i".ToUpper() → "İ")
if (ext.ToUpper() == "TXT") { }

// ✓ — invariant/ordinal for non-linguistic comparisons
if (ext.Equals("txt", StringComparison.OrdinalIgnoreCase)) { }
"file".ToUpperInvariant();                  // stable casing
```

Use **`Ordinal`/`OrdinalIgnoreCase`** (or `ToUpperInvariant`) for identifiers, keys, file extensions, and protocol strings — faster and immune to locale quirks. Reserve culture-aware comparison for user-facing sorting/display. (CSharpBook Ch13 §08.)

---

## Normalization

The same visible string can have different code-point sequences (precomposed `é` vs `e` + combining accent) — they're `!=` despite looking identical:

```csharp
string a = "café";              // U+00E9
string b = "café";        // e + combining acute
a == b;                          // false!
a.Normalize() == b.Normalize();  // true (both → NFC)
```

`Normalize(NormalizationForm.FormC)` composes (default); `FormD` decomposes. **Normalize user input** before security-sensitive comparisons (usernames, paths) to avoid homograph/spoofing issues.

---

## Globalization-invariant mode

For backend services with **no localization needs**, run in invariant mode to drop the ICU dependency, shrink the image, and speed startup:

```xml
<PropertyGroup>
  <InvariantGlobalization>true</InvariantGlobalization>
</PropertyGroup>
```

In this mode all cultures behave like `InvariantCulture` (no ICU). Great for containers/microservices that only handle machine data. **Don't** use it if you display localized content or do culture-aware sorting. (Common in minimal container images alongside Native AOT — [Ch19](../19-Deployment/README.md).)

---

## Common gotchas

### Persisting with current culture

The #1 globalization bug — corrupts numbers/dates across locales. Use `InvariantCulture` for storage/wire.

### Culture-sensitive comparison of identifiers

The Turkish-I problem and performance both argue for `Ordinal`/`OrdinalIgnoreCase` on keys/extensions.

### `DateTime.Parse` without format/culture

`"01/02/2026"` is Jan 2 (US) or Feb 1 (EU)? Use `ParseExact` + culture, or ISO 8601 ([06-DateTimeAndTime.md](06-DateTimeAndTime.md)).

### Wrong encoding → mojibake

Reading a Windows-1252 file as UTF-8 garbles non-ASCII. Know your source encoding; specify it explicitly.

### BOM breaking consumers

A UTF-8 BOM breaks shell scripts and some parsers. Write without it (`new UTF8Encoding(false)`).

### Forgetting normalization

Visually identical strings with different code points compare unequal. `Normalize()` before sensitive comparisons.

---

## Summary

- **Invariant culture for machines** (storage/wire), **current culture for humans** (display); **`Ordinal`/`OrdinalIgnoreCase`** for identifiers (and to dodge the Turkish-I bug).
- Modern .NET uses **ICU on all platforms** for consistent culture behavior; opt into app-local ICU for stability. **`RegionInfo`** gives region/currency data.
- A `string` is UTF-16 in memory; **default to UTF-8 without BOM** for I/O; register a provider for legacy code pages; use `u8` literals for constant byte sequences.
- **Normalize** Unicode before security-sensitive comparisons.
- **`InvariantGlobalization`** mode drops ICU for non-localized services (smaller, faster startup).
- Deep mechanics: CSharpBook Chapter 13 §07–08.

→ Next: [08-Diagnostics.md](08-Diagnostics.md)
