# Test Strategy

## How much of what kind of test?

Knowing the tools (xUnit, mocks, `WebApplicationFactory`, Testcontainers, bUnit, BenchmarkDotNet) is half the battle; the other half is **strategy** — what to test, at which level, and how much. The classic guide is the **test pyramid**; a modern refinement is the **test trophy**. Both answer: maximize confidence per unit of test-suite cost (speed + maintenance), avoiding both under-testing (bugs ship) and over-testing (a brittle, slow suite that resists change).

---

## The test pyramid

The traditional shape: **many fast unit tests at the base, fewer integration tests, very few slow E2E tests at the top.**

```
        /\        E2E (few)         — slow, brittle, high-fidelity (real browser/app)
       /  \       Integration       — medium speed, real pipeline/dependencies
      /    \      (some)
     /______\     Unit (many)        — fast, isolated, cheap
```

The rationale: unit tests are fast and cheap, so write lots; E2E tests are slow and flaky, so write few but cover critical journeys. The pyramid warns against the **inverted pyramid / ice-cream cone** (mostly E2E tests) — a slow, flaky suite that's painful to maintain.

---

## The test trophy

A modern refinement (popularized for systems with rich integration concerns) shifts weight toward **integration tests**:

```
        ▽         E2E (few)
      ▭▭▭▭▭       Integration (most weight)  ← "tests that resemble how the software is used"
       ▭▭▭        Unit (some)
        ▬         Static analysis (compiler, nullable, analyzers — the base)
```

The argument: **integration tests give the most confidence per test** because they exercise real wiring (the bugs that actually bite are often in the seams between units, not within them), and modern tooling (`WebApplicationFactory` + Testcontainers — [05-IntegrationTests.md](05-IntegrationTests.md), [06-TestContainers.md](06-TestContainers.md)) makes them fast enough to write many. Static analysis (the compiler, **nullable reference types**, Roslyn analyzers) forms the free base layer — it catches whole bug classes before any test runs.

Pyramid vs trophy isn't a contradiction — both say "few E2E," "static analysis is free," and "balance fast-isolated against realistic." The trophy just weights integration higher for app/service code where the seams matter most.

---

## What to unit-test vs integration-test

- **Unit test** (isolated, mocked) — **logic-rich** code: domain rules, calculations, algorithms, parsers, validation, state machines. These have many branches/edge cases best covered cheaply in isolation ([01-xUnitDeep.md](01-xUnitDeep.md), [03-Mocking.md](03-Mocking.md)).
- **Integration test** (real wiring) — **the seams**: does the endpoint route, bind, validate, authorize, hit the database, and serialize correctly end-to-end? Does the EF query actually run? Does DI resolve? These catch wiring bugs unit tests can't ([05-IntegrationTests.md](05-IntegrationTests.md)).
- **E2E** (real browser/app) — a few **critical user journeys** (login → checkout → confirmation) in a real environment, via Playwright/Selenium.

A useful heuristic: **test behavior, not implementation.** Tests coupled to *how* code works (over-mocking, asserting internal calls) break on refactors; tests coupled to *what* it does (inputs → outputs/effects) survive refactors and document intent.

---

## What to mock vs use real

Match this to the level ([03-Mocking.md](03-Mocking.md)):

- **Unit tests**: mock all collaborators to isolate the unit — but only at meaningful boundaries; don't mock value objects or over-specify interactions.
- **Integration tests**: use **real** in-process components (the pipeline, DI, often a real DB via Testcontainers); substitute only **external/nondeterministic** dependencies (third-party APIs, email/SMS, time via `TimeProvider`, randomness).
- **Determinism**: inject `TimeProvider` (not `DateTime.Now`), seed randomness, and fix external inputs so tests are reproducible — flakiness destroys trust in a suite.

---

## Coverage and what it does (and doesn't) mean

- **Code coverage** measures which lines/branches tests execute — useful for finding *untested* areas, but a **poor goal** in itself: 100% coverage with weak assertions tests nothing, and chasing a coverage number produces low-value tests. Use coverage to spot gaps, not as a target.
- **Mutation testing** (e.g., Stryker.NET) is a stronger signal: it mutates your code and checks whether tests *catch* the change — measuring assertion quality, not just execution. High mutation score means tests actually verify behavior.

---

## Making the suite trustworthy and fast

- **Fast**: keep the broad base (unit + fast integration) quick so developers run it constantly; isolate slow container/E2E suites into a separate CI stage.
- **Reliable**: no flaky tests — eliminate nondeterminism (time, randomness, ordering, shared state — [01-xUnitDeep.md](01-xUnitDeep.md)). A flaky suite gets ignored, defeating its purpose.
- **Independent**: tests shouldn't depend on order or shared mutable state; reset data between integration tests ([05-IntegrationTests.md](05-IntegrationTests.md)).
- **Maintainable**: test behavior over implementation; use builders/Bogus ([09-PropertyBased.md](09-PropertyBased.md)) for test data; keep tests readable (FluentAssertions — [04-FluentAssertions.md](04-FluentAssertions.md)).

---

## Common gotchas

### Inverted pyramid (too many E2E)

A suite dominated by slow, flaky E2E tests is painful and unreliable. Keep E2E few (critical journeys); push coverage down to faster unit/integration layers.

### Over-mocking unit tests

Mocking everything tests implementation, not behavior — brittle to refactors. Mock at meaningful boundaries; assert outcomes ([03-Mocking.md](03-Mocking.md)).

### Chasing coverage percentage

High coverage with weak assertions is false confidence. Use coverage to find gaps; use mutation testing to measure assertion strength.

### Flaky tests tolerated

One flaky test erodes trust in the whole suite (people stop believing red builds). Fix or quarantine flakiness immediately — eliminate the nondeterminism.

### Ignoring static analysis

The compiler, nullable reference types, and analyzers catch whole bug classes for free — skipping them (disabling nullable, ignoring warnings) wastes the cheapest layer.

---

## Summary

- Strategy = **maximize confidence per cost**: the **pyramid** (many unit, some integration, few E2E) and the **trophy** (weight integration heaviest, static analysis as the free base) both say keep **E2E few** and balance fast-isolated vs realistic.
- **Unit-test logic-rich code** (mocked, isolated); **integration-test the seams** (real pipeline/DB via `WebApplicationFactory` + Testcontainers — [05](05-IntegrationTests.md), [06](06-TestContainers.md)); **E2E** only critical journeys. **Test behavior, not implementation** so tests survive refactors.
- **Mock** only at meaningful boundaries / external-nondeterministic deps; keep tests **deterministic** (`TimeProvider`, seeded randomness). Use **coverage** to find gaps (not as a goal) and **mutation testing** to measure assertion quality.
- A good suite is **fast, reliable (no flakiness), independent, and maintainable** — and leans on **static analysis** (compiler, nullable, analyzers) as the cheapest defense.

→ Next: [Questions.md](Questions.md)
