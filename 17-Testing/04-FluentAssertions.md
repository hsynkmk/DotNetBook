# Assertion Libraries

## Readable assertions, better failure messages

Built-in assertions (`Assert.Equal(expected, actual)`) work, but they read awkwardly and give terse failure messages. **Assertion libraries** make assertions read like English and produce **rich, specific failure output** that tells you exactly what went wrong. The most popular is **FluentAssertions** (`actual.Should().Be(expected)`); **Shouldly** is a lighter alternative (`actual.ShouldBe(expected)`). They work with any test framework ([02-NUnitMSTest.md](02-NUnitMSTest.md)) and dramatically improve the diagnostic experience when a test fails.

```csharp
// Built-in:                          Assert.Equal(5, result);
// FluentAssertions:                  result.Should().Be(5);
// On failure FluentAssertions says:  "Expected result to be 5, but found 3."
```

> **Licensing note**: FluentAssertions changed to a commercial license from v8 (free for non-commercial; paid for commercial use). v7 remains free under the old license. This has pushed some teams toward **Shouldly** (free/OSS) or **AwesomeAssertions** (an OSS fork). The *concepts* below apply to all of them.

---

## Why fluent assertions

Two concrete benefits:

1. **Readability** — `order.Total.Should().BeGreaterThan(0)` reads as a sentence; chained assertions express intent clearly.
2. **Failure messages** — they automatically include the subject name, the expectation, and the actual value (and for collections/objects, *which element/property* differed) — far more useful than "Assert.True failed."

```csharp
result.Should().Be(5);
name.Should().StartWith("Jo").And.EndWith("hn").And.HaveLength(4);
collection.Should().HaveCount(3).And.Contain(x => x.IsActive);
action.Should().Throw<ArgumentException>().WithMessage("*required*");
```

---

## Collection and object assertions

The biggest wins are on collections and object graphs, where built-in asserts force tedious element-by-element checks:

```csharp
// Collections
items.Should().HaveCount(3);
items.Should().ContainSingle(x => x.Id == 5);
items.Should().BeInAscendingOrder(x => x.Date);
items.Should().OnlyContain(x => x.IsValid);
items.Should().BeEquivalentTo(expectedItems);     // order-insensitive structural compare

// Object graphs — compares by structure, not reference
actual.Should().BeEquivalentTo(expected);          // deep, member-by-member equality
actual.Should().BeEquivalentTo(expected, opts => opts.Excluding(x => x.Timestamp));
```

**`BeEquivalentTo`** is the standout: it compares two objects/collections **by structure** (member-by-member, recursively) rather than by reference or `Equals`, with options to exclude/include members. This turns "assert these two DTOs match" from a dozen property asserts into one line — invaluable for API/mapping tests.

---

## Exception assertions

Asserting exceptions reads cleanly and lets you inspect the thrown exception:

```csharp
Action act = () => sut.Process(null!);
act.Should().Throw<ArgumentNullException>()
   .WithParameterName("input");

await sut.Invoking(s => s.RunAsync())
        .Should().ThrowAsync<InvalidOperationException>()
        .WithMessage("*not initialized*");

act.Should().NotThrow();
```

---

## Shouldly (the lighter alternative)

Shouldly takes a different but similarly readable approach, and is fully open-source:

```csharp
result.ShouldBe(5);
name.ShouldStartWith("Jo");
collection.ShouldContain(x => x.IsActive);
Should.Throw<ArgumentException>(() => sut.Process(null!));
```

Shouldly's signature feature is that its **failure messages include the asserted expression** (e.g., it prints `result.ShouldBe(5)` with the actual value), without you writing a message. It's lighter than FluentAssertions and license-free — a common choice for teams avoiding FluentAssertions' commercial licensing.

---

## Custom assertions and scopes

- **Assertion scopes** (FluentAssertions `using new AssertionScope()`) collect *multiple* assertion failures and report them all at once, instead of stopping at the first — useful when you want to see every mismatch in one run.
- **Extension methods** let you build domain-specific assertions (`order.Should().BeShipped()`) for repeated, readable checks.

```csharp
using (new AssertionScope()) {
    order.Total.Should().Be(100);
    order.Status.Should().Be(Status.Paid);   // both checked even if the first fails
}
```

---

## Common gotchas

### Comparing object graphs with `Be` instead of `BeEquivalentTo`

`Should().Be()` uses reference/`Equals` equality; for two distinct objects with equal *contents*, use **`BeEquivalentTo`** (structural). Using `Be` fails on equal-but-not-same instances.

### Ignoring the licensing change

FluentAssertions v8+ requires a commercial license for commercial use. Pin v7, or switch to Shouldly/AwesomeAssertions, to avoid licensing surprises — decide deliberately.

### One assertion stopping the run

Without an `AssertionScope`, the first failed assertion stops the test, hiding later failures. Use a scope to see all mismatches at once when that's helpful.

### Over-asserting with `BeEquivalentTo` on volatile fields

Structural comparison includes *all* members, so timestamps/generated ids cause spurious failures. Exclude volatile members (`opts => opts.Excluding(...)`).

### Mixing assertion styles inconsistently

Half the suite using `Assert.Equal` and half `Should().Be()` is jarring. Standardize on one assertion library/style across the project.

---

## Summary

- **Assertion libraries** make tests read like English and produce **rich failure messages** (subject, expectation, actual, and *which* element/member differed) — a big diagnostic upgrade over built-in `Assert`.
- **FluentAssertions** (`x.Should().Be(...)`) is most popular; its standout is **`BeEquivalentTo`** — **structural**, member-by-member comparison of objects/collections (with include/exclude options) — ideal for DTO/mapping/API tests.
- Assert exceptions fluently (`act.Should().Throw<T>().WithMessage(...)`); use **`AssertionScope`** to report multiple failures at once and extension methods for domain-specific assertions.
- **Shouldly** is a lighter, fully-OSS alternative (`x.ShouldBe(...)` with expression-aware messages); note FluentAssertions' **v8+ commercial licensing** — pin v7 or use Shouldly/AwesomeAssertions if that's a concern. Standardize on one style.

→ Next: [05-IntegrationTests.md](05-IntegrationTests.md)
