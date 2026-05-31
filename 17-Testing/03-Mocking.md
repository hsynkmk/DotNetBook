# Mocking

## Isolating the unit under test

A **unit** test should test *one* thing in isolation — but real classes depend on others (a repository, an HTTP client, a clock). **Mocking** replaces those dependencies with controllable **test doubles** so you can (a) make the dependency behave however the test needs (return a value, throw), and (b) verify how the unit *interacts* with it (was `SaveAsync` called once?). The three popular .NET libraries are **Moq**, **NSubstitute**, and **FakeItEasy** — all do the same job with different syntax. Mocking works against **interfaces** (or virtual members), which is one reason to depend on abstractions ([Ch03 DI](../03-HostingAndDI/README.md)).

```csharp
// NSubstitute
var repo = Substitute.For<IOrderRepository>();
repo.GetAsync(1).Returns(new Order { Id = 1 });
var sut = new OrderService(repo);                 // inject the fake
var result = await sut.GetAsync(1);
await repo.Received(1).GetAsync(1);               // verify interaction
```

---

## The vocabulary of test doubles

"Mock" is often used loosely, but the precise terms matter:

| Double | Purpose |
|---|---|
| **Stub** | returns canned values (no verification) — "make it return X" |
| **Mock** | a stub you also **verify** interactions on — "assert it was called" |
| **Fake** | a working but simplified implementation (e.g., an in-memory repository) |
| **Spy** | records calls for later inspection |
| **Dummy** | a placeholder passed but never used |

The libraries below create **mocks/stubs**; a **fake** (like EF Core's in-memory provider or a hand-written in-memory repo) is sometimes a better choice for a stateful dependency.

---

## Moq

The most widely used; uses a lambda-based setup syntax on a `Mock<T>`:

```csharp
var repo = new Mock<IOrderRepository>();
repo.Setup(r => r.GetAsync(1)).ReturnsAsync(new Order { Id = 1 });
repo.Setup(r => r.GetAsync(It.IsAny<int>())).ReturnsAsync((Order?)null);  // argument matcher

var sut = new OrderService(repo.Object);          // .Object is the fake instance
await sut.GetAsync(1);

repo.Verify(r => r.SaveAsync(It.IsAny<Order>()), Times.Once);   // verify
```

- `Setup(...).Returns/ReturnsAsync/Throws` configures behavior.
- `It.IsAny<T>()`, `It.Is<T>(predicate)` are **argument matchers**.
- `Verify(..., Times.Once/Never/Exactly(n))` asserts interactions.
- `mock.Object` is the actual instance to inject.

---

## NSubstitute

A cleaner, less ceremonial syntax — you call methods directly on the substitute (no `.Object`):

```csharp
var repo = Substitute.For<IOrderRepository>();
repo.GetAsync(1).Returns(new Order { Id = 1 });
repo.GetAsync(Arg.Any<int>()).Returns((Order?)null);

var sut = new OrderService(repo);                 // the substitute IS the instance
await sut.GetAsync(1);

await repo.Received(1).SaveAsync(Arg.Any<Order>());   // verify
repo.DidNotReceive().Delete(Arg.Any<int>());
```

`Arg.Any<T>()`/`Arg.Is<T>(...)` are matchers; `Received()`/`DidNotReceive()` verify. Many teams prefer NSubstitute for readability.

---

## FakeItEasy

Similar goals, fluent `A.Fake<T>()` / `A.CallTo(...)` syntax:

```csharp
var repo = A.Fake<IOrderRepository>();
A.CallTo(() => repo.GetAsync(1)).Returns(new Order { Id = 1 });
// ...
A.CallTo(() => repo.SaveAsync(A<Order>._)).MustHaveHappenedOnceExactly();
```

All three are capable; the choice is mostly team preference. Pick one and use it consistently.

---

## What (and what not) to mock

- **Mock**: dependencies you own/abstract — repositories, services, an `IClock`/`TimeProvider` ([Ch02 §06](../02-BCL/06-DateTimeAndTime.md)), an `HttpClient` boundary, an email sender. These are slow, nondeterministic, or have side effects.
- **Don't mock**: types you don't own with complex behavior (mock a thin *adapter interface* you wrote around them instead), value objects/POCOs (just construct them), or the system under test itself.
- **Don't over-mock**: a test that mocks everything verifies *implementation* (which calls happen) rather than *behavior* (the result). Over-mocked tests are brittle — they break on harmless refactors. Prefer asserting outcomes; verify interactions only when the interaction *is* the behavior (e.g., "the email was sent").

---

## Mocking `HttpClient` and other awkward types

`HttpClient` isn't an interface — you don't mock it directly. Instead, mock the **`HttpMessageHandler`** it wraps, or (better) abstract the call behind your own interface/typed client ([Ch09 §02](../09-NetworkingAndHttp/02-IHttpClientFactory.md)) and mock that:

```csharp
var handler = new Mock<HttpMessageHandler>();
handler.Protected()
    .Setup<Task<HttpResponseMessage>>("SendAsync", ItExpr.IsAny<HttpRequestMessage>(), ItExpr.IsAny<CancellationToken>())
    .ReturnsAsync(new HttpResponseMessage(HttpStatusCode.OK) { Content = new StringContent("{}") });
var client = new HttpClient(handler.Object);
```

Similarly, prefer injecting `TimeProvider`/`IClock` over mocking `DateTime.Now` (which you can't), and abstract static/sealed dependencies behind interfaces so they're mockable.

---

## Common gotchas

### Over-mocking → brittle tests

Mocking every collaborator tests implementation details, not behavior; harmless refactors break the tests. Mock only at meaningful boundaries (slow/nondeterministic/side-effecting); assert outcomes over interactions.

### Mocking concrete classes / non-virtual members

These libraries mock **interfaces** and **virtual/abstract** members. To make a dependency mockable, depend on an interface (or make members virtual). Sealed types/static methods aren't mockable — wrap them.

### Verifying too much

Asserting the exact sequence/count of every call couples tests to internals. Verify an interaction only when it *is* the behavior under test ("email sent once"); otherwise assert the result.

### Using a mock where a fake is better

For a stateful dependency (a repository you read-then-write), a hand-written in-memory **fake** is often clearer and less brittle than a pile of `Setup`/`Returns`. Consider fakes for stateful collaborators.

### Mocking `DateTime.Now`/`HttpClient` directly

You can't mock statics or non-virtual `HttpClient`. Inject **`TimeProvider`** and mock the **`HttpMessageHandler`** (or a typed-client abstraction) instead.

---

## Summary

- **Mocking** replaces a unit's dependencies with controllable **test doubles** (stub = canned values; mock = stub + interaction verification; fake = simplified working impl) to isolate the unit and control/verify its collaborators.
- **Moq** (`Mock<T>`, `.Object`, `Setup`/`Verify`), **NSubstitute** (`Substitute.For<T>`, `.Returns`/`Received()` — no `.Object`), and **FakeItEasy** (`A.Fake<T>`/`A.CallTo`) are equivalent in capability — pick one and use it consistently.
- Mock **abstractions you own** (repositories, services, `TimeProvider`, an HTTP boundary); **don't over-mock** (it tests implementation, not behavior, and is brittle) and don't mock value objects or the SUT.
- For awkward types: mock **`HttpMessageHandler`** (not `HttpClient`), inject **`TimeProvider`** (not `DateTime.Now`), and wrap sealed/static dependencies behind interfaces. Consider a **fake** for stateful collaborators.

→ Next: [04-FluentAssertions.md](04-FluentAssertions.md)
