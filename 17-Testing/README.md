# Chapter 17 — Testing

> Unit, integration, end-to-end. xUnit + Moq for the small; WebApplicationFactory + TestContainers for the realistic. Plus BenchmarkDotNet for performance regression.

**Prerequisites**: comfortable writing C# code; understand DI from Chapter 03.

**Time to read**: ~6-8 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-xUnitDeep.md](01-xUnitDeep.md) | xUnit conventions, fixtures, parallelism, attributes, async tests. |
| [02-NUnitMSTest.md](02-NUnitMSTest.md) | Quick comparison for the other two frameworks. |
| [03-Mocking.md](03-Mocking.md) | Moq, NSubstitute, FakeItEasy — choosing and using. |
| [04-FluentAssertions.md](04-FluentAssertions.md) | The expressive assertion library. |
| [05-IntegrationTests.md](05-IntegrationTests.md) | `WebApplicationFactory<TEntryPoint>` — in-memory ASP.NET Core tests. |
| [06-TestContainers.md](06-TestContainers.md) | Docker-based real databases / brokers in tests. |
| [07-Bunit.md](07-Bunit.md) | Testing Blazor components. |
| [08-BenchmarkDotNet.md](08-BenchmarkDotNet.md) | Performance microbenchmarking — common pitfalls (dead-code elimination, cold caches). |
| [09-PropertyBased.md](09-PropertyBased.md) | FsCheck, Bogus for synthetic data, property-based testing. |
| [10-TestStrategy.md](10-TestStrategy.md) | Pyramid vs trophy, what to mock, what to integrate. |
| [Questions.md](Questions.md) | Drilling. |
| [Coding.md](Coding.md) | Build unit + integration tests for an API; benchmark a hot path. |

→ Begin: [01-xUnitDeep.md](01-xUnitDeep.md)
