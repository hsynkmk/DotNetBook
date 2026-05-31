# Async Idioms at Scale

## Getting async right where it matters most

Async/await is straightforward in a single method but has **scale-level idioms** that separate robust systems from fragile ones — especially in **libraries** (consumed by code you don't control) and **high-throughput services**. The recurring concerns: **`ConfigureAwait` in libraries**, **`CancellationToken` propagation** everywhere, **avoiding deadlocks and thread-pool starvation**, and **not blocking on async**. These are the async best practices that matter under real load and across library boundaries. (The async *mechanics* — state machines, `Task` vs `ValueTask`, the synchronization context — are in [CSharpBook Ch08](../../CSharpBook/08-Concurrency/README.md); this is the *at-scale discipline*.)

---

## Never block on async (the cardinal rule)

The most damaging async mistake, repeated for emphasis ([10-AntiPatterns.md](10-AntiPatterns.md), [Ch21 §10](../21-Performance/10-CommonWins.md)): **never** `.Result` / `.Wait()` / `.GetAwaiter().GetResult()` on async work in request-handling code.

```csharp
var data = service.GetAsync().Result;   // ✗ blocks a thread; starvation + possible deadlock
var data = await service.GetAsync();    // ✓ async all the way
```

- Blocking **ties up a thread** waiting on async work → under load, the **thread pool starves** (requests queue, app stalls — visible in counters/dumps — [Ch21 §03](../21-Performance/03-DotnetCounters.md)).
- In contexts with a **synchronization context** (legacy ASP.NET, UI), blocking can **deadlock** (the continuation needs the context the blocked thread holds).

The fix is always **async all the way** — propagate `await` up the entire call chain; don't bridge sync↔async by blocking.

---

## `ConfigureAwait(false)` in libraries

When an `await` resumes, it normally tries to return to the captured **synchronization context** ([CSharpBook Ch08 §06](../../CSharpBook/08-Concurrency/README.md)). In **library** code that doesn't need the original context, capturing it adds overhead and risks deadlocks if a caller blocks. So **library code uses `ConfigureAwait(false)`** to *not* capture the context:

```csharp
// In a LIBRARY method — don't capture the caller's context:
public async Task<Data> LoadAsync() {
    var raw = await _http.GetStringAsync(_url).ConfigureAwait(false);
    return Parse(raw);   // resumes on any thread pool thread — no context capture
}
```

| | `ConfigureAwait(true)` (default) | `ConfigureAwait(false)` |
|---|---|---|
| Resumes on | captured context | any thread pool thread |
| Use in | code that **needs** the context (UI, some app code) | **libraries** and context-agnostic code |

Key points:
- **ASP.NET Core has no synchronization context**, so `ConfigureAwait(false)` is largely a **no-op** there — but **libraries** should still use it because they may be consumed by apps that *do* have a context (UI, legacy ASP.NET), where it prevents deadlocks and overhead.
- In **app** code that needs the context (a Blazor/WPF UI method that touches UI after the await — [Ch14](../14-Blazor/README.md), [Ch16](../16-Desktop/README.md)), you **don't** use `ConfigureAwait(false)` (you need to resume on the UI context).

The idiom: **`ConfigureAwait(false)` in library code; default in app/UI code that needs the context.**

---

## Propagate `CancellationToken` everywhere

A `CancellationToken` lets a caller signal "stop, I no longer need this" — and at scale, **honoring cancellation** prevents wasted work (a cancelled HTTP request, an abandoned query keeps running otherwise). The idiom: **accept a `CancellationToken` parameter and pass it down** through every async call:

```csharp
public async Task<Order> GetAsync(int id, CancellationToken ct) {   // accept it
    var order = await _db.Orders.FindAsync([id], ct);              // pass it down
    await _cache.SetAsync(id, order, ct);                          // ...everywhere
    return order;
}
```

- ASP.NET Core supplies `HttpContext.RequestAborted` as the token, fired when the client disconnects — so cancelled requests stop doing work ([Ch04](../04-AspNetCore/README.md)).
- The host's shutdown token ([Ch03 §01](../03-HostingAndDI/01-GenericHost.md)) cancels background work on graceful shutdown ([Ch19 §04](../19-Deployment/04-Kubernetes.md)).
- **Link tokens** (`CancellationTokenSource.CreateLinkedTokenSource`) to combine (e.g., request token + a timeout).

A method that ignores cancellation (or doesn't accept a token) keeps consuming resources after the caller gave up — at scale, that's wasted threads, connections, and DB work. **Thread the token through the whole chain.**

---

## Other scale idioms

- **`Task.WhenAll` for concurrency** — await independent operations **together**, not sequentially:

```csharp
var (a, b) = (FetchAAsync(ct), FetchBAsync(ct));
await Task.WhenAll(a, b);   // both run concurrently, not one-then-the-other
```

- **`ValueTask` for hot paths that often complete synchronously** — avoids a `Task` allocation when the result is usually already available (e.g., a cache hit) ([CSharpBook Ch08 §04](../../CSharpBook/08-Concurrency/README.md)); don't await a `ValueTask` more than once.
- **`IAsyncEnumerable<T>`** for streaming async sequences (with `[EnumeratorCancellation]` to flow the token) — process items as they arrive without buffering ([Ch05 §02](../05-EFCore/02-Querying.md)).
- **`async Task`, never `async void`** (except event handlers) — `async void` can't be awaited or caught, crashing the process on exception ([10-AntiPatterns.md](10-AntiPatterns.md)).
- **No fire-and-forget without care** — an un-awaited task's exceptions are lost and it may outlive its scope; if you must, observe it (and consider a background queue — [Ch08 §04](../08-BackgroundProcessing/04-QueuedBackgroundWork.md)).

---

## Common gotchas

### Blocking on async (`.Result`/`.Wait()`)

Causes **thread-pool starvation** under load and **deadlocks** in contexts with a sync context. Always **await**; go async all the way ([10-AntiPatterns.md](10-AntiPatterns.md)).

### Missing `ConfigureAwait(false)` in a library

A library that captures the caller's context can deadlock/slow apps that *have* a context (UI/legacy ASP.NET). Use `ConfigureAwait(false)` in **library** code (it's a no-op in ASP.NET Core but still correct for portability).

### `ConfigureAwait(false)` in UI code that needs the context

In a UI method that touches UI after the await, `ConfigureAwait(false)` resumes off the UI thread → cross-thread errors. Don't use it where you **need** the context.

### Not propagating `CancellationToken`

Methods that ignore cancellation keep working after the caller gave up — wasted threads/connections/queries at scale. **Accept and pass** the token through the whole chain.

### `async void` / un-awaited tasks

`async void` exceptions crash the process; un-awaited tasks lose exceptions and may outlive scope. Use `async Task` and await (or deliberately manage background work — [Ch08](../08-BackgroundProcessing/README.md)).

---

## Summary

- The cardinal async rule at scale: **never block on async** (`.Result`/`.Wait()`) — it causes **thread-pool starvation** under load and **deadlocks** with a sync context; go **async all the way**.
- **`ConfigureAwait(false)` in library code** (don't capture the caller's context — prevents deadlocks/overhead in UI/legacy-ASP.NET consumers; a no-op in ASP.NET Core but correct for portability); **default** in app/UI code that needs to resume on its context.
- **Propagate `CancellationToken`** through every async call (accept a parameter, pass it down) so cancelled requests/shutdown stop wasted work; use `HttpContext.RequestAborted`/host shutdown tokens and link tokens for timeouts.
- Use **`Task.WhenAll`** for concurrency, **`ValueTask`** for often-synchronous hot paths, **`IAsyncEnumerable`** for streaming, and **`async Task` not `async void`** — avoid careless fire-and-forget (mechanics in [CSharpBook Ch08](../../CSharpBook/08-Concurrency/README.md)).

→ Next: [Questions.md](Questions.md)
