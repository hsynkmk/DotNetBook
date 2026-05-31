# Property-Based Testing and Test Data

## Testing properties, not examples

Example-based tests check specific inputs (`Add(2,3) == 5`). **Property-based testing** instead asserts **properties that should hold for *all* inputs**, then generates hundreds of random inputs to try to falsify them. The framework (**FsCheck** in .NET) finds edge cases you'd never think to write — empty collections, negative numbers, huge values, Unicode oddities — and when it finds a failure, it **shrinks** the input to the minimal failing case. It complements example tests: examples document specific behavior; properties catch the cases you didn't imagine. Separately, **Bogus** generates realistic fake data for tests/seeding.

```csharp
// Property: reversing a list twice yields the original
[Property]
public bool Reversing_twice_is_identity(int[] xs)
    => xs.Reverse().Reverse().SequenceEqual(xs);   // FsCheck runs this for ~100 random arrays
```

---

## What makes a good property

The hard part is *finding* properties. Common patterns:

- **Round-trip / inverse**: `decode(encode(x)) == x` (serialization, parsing, compression) — a serializer should reproduce what it serialized.
- **Invariants**: `sorted.Count == original.Count` and `IsSorted(sort(x))` — sorting preserves length and produces order.
- **Idempotence**: `f(f(x)) == f(x)` (normalization, deduplication).
- **Commutativity / associativity**: `Add(a,b) == Add(b,a)`.
- **Equivalence ("oracle")**: a fast/new implementation matches a simple/known-correct one for all inputs.
- **Metamorphic**: relationships between outputs (`sort(x).First() == min(x)`).

```csharp
[Property]
public Property Roundtrips_through_json(Order order) {
    var json = JsonSerializer.Serialize(order);
    var back = JsonSerializer.Deserialize<Order>(json);
    return (back == order).ToProperty();
}
```

---

## FsCheck — generation and shrinking

FsCheck's two superpowers:

1. **Generation** — it produces random inputs of the parameter types (ints, strings, lists, your own types via generators), including boundary/degenerate values (0, empty, null where allowed, max/min) that example tests routinely miss.
2. **Shrinking** — when a property fails for some large random input, FsCheck **automatically reduces** it to the *smallest* input that still fails — so instead of "failed on a 500-element array of random numbers," you get "failed on `[0]`" or "failed on the empty string." This minimal counterexample makes debugging dramatically easier.

```csharp
// If this property is wrong, FsCheck reports the smallest failing input, e.g. n = 0
[Property]
public bool Abs_is_non_negative(int n) => Math.Abs(n) >= 0;   // FAILS and shrinks to int.MinValue!
```

(That example actually surfaces a real edge case: `Math.Abs(int.MinValue)` overflows — exactly the kind of bug property testing catches and example tests usually don't.)

You can constrain generators (`Prop.ForAll` with custom `Arbitrary`/`Gen`) when inputs must satisfy preconditions (e.g., only positive numbers, valid emails).

---

## Bogus — realistic fake data

**Bogus** generates realistic synthetic data (names, emails, addresses, dates, prices) — for seeding test databases, populating DTOs in tests, or demos — far better than `"test1"`/`"test2"`:

```csharp
var faker = new Faker<Customer>()
    .RuleFor(c => c.Id, f => f.Random.Guid())
    .RuleFor(c => c.Name, f => f.Name.FullName())
    .RuleFor(c => c.Email, (f, c) => f.Internet.Email(c.Name))
    .RuleFor(c => c.CreatedAt, f => f.Date.Past());

var customers = faker.Generate(100);   // 100 realistic, varied customers
```

Seed the `Faker` for **deterministic** data (`new Faker<T>().UseSeed(123)`) when tests must be reproducible. Bogus is about *realistic, varied* data; FsCheck is about *adversarial, edge-finding* data — different tools for different jobs.

---

## When to use which

- **Example tests** (most of your suite) — specific, documented behaviors and known edge cases.
- **Property tests** — algorithms, parsers, serializers, data structures, anything with clear invariants/round-trips, where the input space is large and you want edge-case coverage.
- **Bogus** — generating realistic test/seed data (not for finding edge cases).

Property testing shines for *logic with mathematical structure*; it's less useful for thin CRUD/glue code where there's no interesting invariant. Use it where properties are natural, alongside example tests.

---

## Common gotchas

### Trivial or wrong properties

A property that's tautologically true (or that just restates the implementation) tests nothing. Good properties are independent of the implementation (round-trip, invariant, oracle). Think about what *must* be true for all inputs.

### Ignoring the shrunk counterexample

FsCheck hands you the minimal failing input — that's the gift. Read it; it usually points straight at the bug (like `int.MinValue` for `Math.Abs`).

### Unconstrained generators producing invalid inputs

If your function has preconditions (positive numbers, valid dates), an unconstrained generator feeds it invalid data and the property fails spuriously. Constrain the generator (custom `Gen`/`Arbitrary`) to valid inputs.

### Non-deterministic Bogus data breaking tests

Unseeded Bogus produces different data each run; if a test asserts on specifics, it'll flake. Seed the `Faker` for reproducibility when needed.

### Using property tests for everything

Not all code has meaningful properties. For thin glue/CRUD, example tests are clearer. Apply property testing where invariants exist.

---

## Summary

- **Property-based testing** (FsCheck) asserts **properties true for all inputs**, generating hundreds of random cases to falsify them — catching edge cases example tests miss (empty/zero/boundary/overflow).
- Good properties: **round-trip/inverse** (`decode(encode(x))==x`), **invariants** (length preserved, output sorted), **idempotence**, **commutativity**, and **oracle** (match a known-correct impl).
- FsCheck's killer features are **generation** (boundary-heavy random inputs) and **shrinking** (auto-reduce a failure to the **minimal** counterexample — e.g., surfacing `Math.Abs(int.MinValue)` overflow); constrain generators when inputs have preconditions.
- **Bogus** generates **realistic** fake data (names/emails/dates) for seeding/test DTOs — seed it for determinism. Use property tests for algorithm/parser/serializer logic with clear invariants, example tests for the rest.

→ Next: [10-TestStrategy.md](10-TestStrategy.md)
