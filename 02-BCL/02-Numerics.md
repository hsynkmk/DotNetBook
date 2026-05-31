# Numerics

## The numeric toolbox

Beyond the primitive numeric types (CSharpBook Ch01 §02), the BCL provides math functions, arbitrary-precision integers, complex numbers, random generation, and bit manipulation. This file maps them and the correctness traps (especially around `Random` and floating point).

---

## `Math` and `MathF`

`Math` operates on `double`; `MathF` on `float` (avoids needless double-precision widening when you're working in `float`):

```csharp
Math.Sqrt(2.0);   Math.Pow(2, 10);   Math.Abs(-5);   Math.Clamp(x, 0, 100);
Math.Min(a, b);   Math.Max(a, b);    Math.Floor(x);  Math.Ceiling(x);  Math.Round(x, 2);
Math.BigMul(a, b);                    // full 64×64→128 multiply
Math.DivRem(a, b);                    // quotient + remainder in one op

MathF.Sqrt(2f);   MathF.Sin(angle);   // float versions
```

### Rounding modes — a common surprise

```csharp
Math.Round(2.5);                                    // 2 — banker's rounding (round half to even)!
Math.Round(2.5, MidpointRounding.AwayFromZero);     // 3 — the "school" rounding
Math.Round(2.345, 2, MidpointRounding.ToZero);
```

`Math.Round` defaults to **banker's rounding** (round-half-to-even) to reduce bias in aggregates. If you expect 2.5 → 3, pass `MidpointRounding.AwayFromZero`. This default trips up financial code constantly.

---

## Integer math helpers

`System.Numerics.BitOperations` and integer intrinsics for fast bit work:

```csharp
using System.Numerics;

BitOperations.PopCount(0b10110u);       // 3 (set bits)
BitOperations.LeadingZeroCount(1u);      // 31
BitOperations.TrailingZeroCount(8u);     // 3
BitOperations.IsPow2(64);                // true
BitOperations.RoundUpToPowerOf2(33);     // 64
BitOperations.Log2(64u);                 // 6
```

These compile to single CPU instructions where available — useful for hashing, bit-packing, and capacity calculations (e.g., `Dictionary` sizing uses them internally).

---

## `BigInteger` — arbitrary precision

When `long`/`ulong` (64-bit) isn't enough — cryptography prototypes, factorials, exact large arithmetic:

```csharp
using System.Numerics;

BigInteger factorial = 1;
for (int i = 1; i <= 100; i++) factorial *= i;     // exact 100! — hundreds of digits
BigInteger huge = BigInteger.Pow(2, 1000);          // 2^1000 exactly
BigInteger gcd = BigInteger.GreatestCommonDivisor(a, b);
```

`BigInteger` grows as needed (heap-allocated digit array), so it's far slower than fixed-width ints — use it only when you genuinely exceed 64 bits, not as a default.

For 128-bit fixed-width (faster than `BigInteger` for that range), .NET 7+ has **`Int128`/`UInt128`**.

---

## `Complex`

```csharp
using System.Numerics;
Complex z = new(3, 4);          // 3 + 4i
double mag = z.Magnitude;        // 5
Complex w = Complex.Sqrt(-1);    // i
```

Niche (signal processing, scientific computing), but built in.

---

## Generic math — `INumber<T>` and friends

The modern way to write numeric code that works across all numeric types (CSharpBook Ch04 §06): static abstract interface members let you constrain a generic to "anything numeric."

```csharp
using System.Numerics;

static T Sum<T>(ReadOnlySpan<T> values) where T : INumber<T> {
    T total = T.Zero;
    foreach (var v in values) total += v;
    return total;
}

int i   = Sum<int>([1, 2, 3]);          // 6
double d = Sum<double>([1.5, 2.5]);     // 4.0
decimal m = Sum<decimal>([1m, 2m, 3m]); // 6m
```

`INumber<T>`, `IAdditionOperators<,,>`, `IFloatingPoint<T>`, etc., expose `T.Zero`, `T.One`, operators, and functions generically — no boxing, fully specialized per type ([Ch01 §05](../01-Runtime/05-TypeSystem.md)). This replaced the old "write an overload per numeric type" drudgery.

---

## Random numbers — and the critical security distinction

There are **two** random generators, and confusing them is a security bug:

### `Random` — fast, NOT cryptographic

```csharp
// Use the shared instance — thread-safe, no seeding needed (.NET 6+)
int dice = Random.Shared.Next(1, 7);
double r = Random.Shared.NextDouble();
Random.Shared.Shuffle(array);              // .NET 8+
Random.Shared.GetItems(pool, 5);           // .NET 8+ — pick N

// Seeded (reproducible) instance for tests/simulations:
var rng = new Random(seed: 12345);
```

`Random` is fast and fine for games, simulations, sampling, and test data — but **predictable**. Never use it for tokens, passwords, keys, or anything security-sensitive.

### `RandomNumberGenerator` — cryptographically secure

```csharp
using System.Security.Cryptography;

byte[] key = RandomNumberGenerator.GetBytes(32);             // 256-bit secure key
int secure = RandomNumberGenerator.GetInt32(0, 1_000_000);    // unbiased secure int
string token = Convert.ToHexString(RandomNumberGenerator.GetBytes(16));
```

Use `RandomNumberGenerator` (a CSPRNG) for **anything secret**: API tokens, salts, keys, nonces, password-reset codes. It's slower but unpredictable. See [11-Cryptography.md](11-Cryptography.md).

> **The rule**: `Random` for non-security randomness, `RandomNumberGenerator` for security. Mixing them up is a classic vulnerability.

---

## Bit conversion & binary primitives

```csharp
// BitConverter — value ↔ bytes (respects machine endianness)
byte[] bytes = BitConverter.GetBytes(12345);
int back = BitConverter.ToInt32(bytes);

// BinaryPrimitives — explicit endianness (for protocols/files), span-based, no allocation
using System.Buffers.Binary;
Span<byte> buf = stackalloc byte[4];
BinaryPrimitives.WriteInt32BigEndian(buf, 12345);    // network byte order
int v = BinaryPrimitives.ReadInt32LittleEndian(buf);
```

For wire protocols and file formats, prefer **`BinaryPrimitives`** — it's explicit about endianness (don't rely on `BitConverter`'s machine-dependent byte order) and works on spans with zero allocation.

---

## Floating point — the eternal traps

```csharp
0.1 + 0.2 == 0.3;                       // FALSE — binary floating point can't represent these exactly
Math.Abs((0.1 + 0.2) - 0.3) < 1e-9;     // ✓ — compare with a tolerance

double.NaN == double.NaN;                // FALSE — NaN is never equal to anything
double.IsNaN(x);                         // ✓ — the correct check
1.0 / 0.0;                               // Infinity (not an exception for double/float)
```

- **Never `==` floats** — use a tolerance (or `decimal` for exact decimal math).
- **`decimal`** (128-bit, base-10) for money/financial values — exact decimal representation, no `0.1+0.2` surprise (but slower and smaller range than `double`).
- **`NaN`** propagates and is unequal to itself; check with `double.IsNaN`.
- `double`/`float` division by zero yields `Infinity`/`NaN`, not an exception (integer division by zero *does* throw).

(Floating-point representation: CSharpBook Ch01 §02.)

---

## Checked arithmetic & overflow

```csharp
int max = int.MaxValue;
int wrap = max + 1;              // -2147483648 (silent wraparound by default!)
checked {
    int boom = max + 1;          // throws OverflowException
}
// Or per-project: <CheckForOverflowUnderflow>true</CheckForOverflowUnderflow>
```

Integer overflow **wraps silently by default**. Use `checked` (or the project setting) where overflow would be a bug (financial counters, sizes). `unchecked` opts back out in a checked context. (CSharpBook Ch10 §13.)

---

## Common gotchas

### `Math.Round` banker's rounding

Defaults to round-half-to-even (2.5 → 2). Pass `MidpointRounding.AwayFromZero` for conventional rounding.

### `new Random()` per call / in a loop

Creating many `Random` instances rapidly can yield correlated sequences and wastes allocation. Use `Random.Shared`.

### `Random` for security

A vulnerability. Use `RandomNumberGenerator` for tokens/keys/salts.

### `==` on doubles, or `decimal` vs `double` for money

Compare floats with tolerance; use `decimal` for money (exact base-10).

### Silent integer overflow

Default arithmetic wraps. Use `checked` where overflow matters.

### `BitConverter` endianness assumptions

It uses machine byte order. For protocols/files, use `BinaryPrimitives` with explicit endianness.

---

## Summary

- **`Math`/`MathF`** for double/float math; mind **banker's rounding** (`Math.Round` defaults to round-half-to-even).
- **`BitOperations`** for fast bit work; **`BigInteger`** for arbitrary precision (slow — only when >64-bit); **`Int128`/`Complex`** for niche needs.
- **Generic math** (`INumber<T>`) writes type-agnostic numeric code with no boxing.
- **Two RNGs**: **`Random.Shared`** (fast, predictable — games/sims/tests) vs **`RandomNumberGenerator`** (CSPRNG — tokens/keys/salts). Don't mix them.
- **`BinaryPrimitives`** for explicit-endianness binary I/O (over `BitConverter`).
- Floating point: never `==` (use tolerance or `decimal` for money), `NaN` is unequal to itself, integer overflow **wraps silently** unless `checked`.

→ Next: [03-CollectionsDeep.md](03-CollectionsDeep.md)
