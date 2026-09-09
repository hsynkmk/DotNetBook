# Rapid Fire — 250 One-Line Q&A

> Self-quiz: cover the right-hand side, answer **out loud**, star every miss. Come back to the
> starred ones. Answers here are deliberately one line — the depth is in the topic files.

---

## Platform & runtime

| Q | A |
|---|---|
| .NET vs C#? | .NET is the platform (runtime + BCL + tooling); C# is a language that targets it. |
| What does C# compile to? | IL + metadata in an assembly; the JIT turns it into native code at runtime. |
| Which .NET version should production use? | The latest **LTS** — even-numbered, ~3 years support. **.NET 10** today. |
| Release cadence? | One release every November; even = LTS, odd = STS (~18 months). |
| Three runtimes? | **CoreCLR** (server/desktop), **Mono** (mobile, Blazor WASM), **Native AOT**. |
| SDK vs runtime? | SDK builds (Roslyn, MSBuild, CLI); runtime runs. SDK includes a runtime. |
| What is `global.json` for? | Pinning the SDK version per repository. |
| What does "managed code" mean? | The runtime owns memory (GC), enforces type safety, and mediates native interop. |
| Is .NET Standard still relevant? | Mostly historical — it bridged Framework and Core. Target `net10.0`. |
| What is tiered compilation? | Tier 0 compiles fast for startup; hot methods recompile at Tier 1 fully optimized. |
| What is OSR? | On-Stack Replacement — promotes a method already running a long loop to Tier 1. |
| What is dynamic PGO? | Tier 0 collects a runtime profile; Tier 1 uses it to devirtualize and inline. On by default since .NET 8. |
| What is ReadyToRun? | Precompiled native shipped **alongside** IL — faster startup, no compatibility cost. |
| Native AOT's main restrictions? | No `Reflection.Emit`/`Expression.Compile`/`dynamic`; trimming implied; limited reflection. |
| Why is AOT sometimes *slower* at peak? | No JIT means no dynamic PGO, no OSR, no runtime re-optimization. |
| When do you choose Native AOT? | CLIs, serverless, short-lived containers — where startup and footprint dominate. |
| Why does trimming break reflection? | Static analysis can't see a lookup by string name, so the type gets removed. |
| What predicts trimming failures? | `IL2xxx` warnings. Treat them as errors. |
| Real fix for trim/AOT reflection? | **Source generators** — they emit statically-analyzable code. |

## Garbage collection

| Q | A |
|---|---|
| What's the generational hypothesis? | Most objects die young — so collect the young generation often and cheaply. |
| Gen 0/1/2? | Fresh allocations / survivors buffer / long-lived. Gen 2 collection = a full GC. |
| How does collection work? | Mark from roots, compact survivors together, update all references. |
| Why is allocation so cheap? | The heap stays compacted, so allocation is a pointer bump. |
| What's the LOH? | Large Object Heap — objects ≥ **85,000 bytes**, collected with Gen 2, **not compacted** by default. |
| How do you avoid LOH churn? | Pool large buffers with `ArrayPool<T>`. |
| Workstation vs Server GC? | One heap vs one per core with dedicated GC threads. Server is the ASP.NET Core default. |
| What is background GC? | Concurrent Gen 2 marking, to shrink pause times. |
| What is DATAS? | Dynamically sizes the heap to the real working set — modern Server GC default. |
| Write barriers and card tables? | Track old→young references so a Gen 0 collection needn't scan all of Gen 2. |
| Should you call `GC.Collect()`? | Essentially never — it promotes survivors and makes the next full GC worse. |
| What's the real cost metric? | **Survivors**, not allocations. An object that dies in Gen 0 is nearly free. |
| Why are finalizers expensive? | The object survives an extra collection to be queued for finalization. |
| Boxing, at runtime level? | Allocating a heap object with header + method table and copying the value in. |

## Type system & threading internals

| Q | A |
|---|---|
| What's in an object header? | Sync block / hash, plus a **method table pointer**. |
| How does virtual dispatch work? | Load the method table, index a fixed vtable slot, call. |
| Generics with value types vs reference types? | Value types get **specialized** native code; reference types **share** one body. |
| What does `sealed` buy you? | Cheaper cast checks and easier devirtualization. |
| What replaced AppDomains? | `AssemblyLoadContext` — isolation, side-by-side versions, and (if collectible) unloading. |
| Why doesn't ALC unload work? | Unload is a *request*; one lingering reference pins the whole context. |
| Why is reflection slow? | Metadata lookup, dynamic dispatch, and boxing. Cache `MemberInfo` or generate code. |
| How does the thread pool schedule? | Global queue + per-thread local queues with work-stealing, tuned by hill-climbing. |
| Why does async scale? | Async I/O registers with the OS (IOCP/epoll) and **returns the thread**. |
| What is thread-pool starvation? | Pool threads blocked (usually sync-over-async); the pool injects slowly, so throughput collapses. |
| Cost of a P/Invoke call? | The cooperative→preemptive GC-mode transition (tens of ns) — so batch native calls. |
| `[LibraryImport]` vs `[DllImport]`? | Compile-time generated stub (AOT/trim-safe) vs runtime-generated. |

## BCL

| Q | A |
|---|---|
| Why is `+=` in a loop bad? | `string` is immutable — O(n²) allocations and copies. Use `StringBuilder`. |
| When `StringComparison.Ordinal`? | Identifiers, keys, paths — anything not shown to a human. Fast and exact. |
| What's the Turkish-I bug? | In `tr-TR`, `"I".ToLower()` is `"ı"` — so culture-aware identifier comparison fails. |
| Why can't you `await` across a `Span<T>`? | It's a `ref struct` — stack-only. Use `Memory<T>`. |
| `ArrayPool<T>` sharp edges? | Return in `finally`; the array may be **larger** than requested and is **not zeroed**. |
| Why cache `JsonSerializerOptions`? | Each instance builds its own reflection metadata cache. |
| Why was `BinaryFormatter` removed? | It deserialized arbitrary types — a reliable RCE vector. |
| `DateTime` or `DateTimeOffset`? | `DateTimeOffset` — it carries the offset and is unambiguous. |
| How do you make time testable? | Inject **`TimeProvider`**; use `FakeTimeProvider` in tests. |
| `Random` or `RandomNumberGenerator`? | `Random` for games/simulation; **`RandomNumberGenerator` for anything security-related**. |
| Why `Stopwatch` over `DateTime` subtraction? | `Stopwatch` is monotonic — wall clock can jump backwards. |
| `AsyncLocal` vs `ThreadStatic`? | `AsyncLocal` flows across `await`; `ThreadStatic` does not. |
| `Math.Round(2.5)` returns? | **2** — banker's rounding (half-to-even). |
| What's the `Stream.Read` contract? | It may return fewer bytes than asked — loop, or use `ReadExactlyAsync`. |
| `ConcurrentDictionary.GetOrAdd` gotcha? | The factory isn't atomic and may run more than once. Wrap in `Lazy<T>`. |
| When is `FrozenDictionary` right? | Built once at startup, read on every request — fastest lookups. |

## Hosting, DI & configuration

| Q | A |
|---|---|
| Three DI lifetimes? | **Singleton** (per app), **Scoped** (per request/scope), **Transient** (per resolution). |
| What is a captive dependency? | A longer-lived service capturing a shorter-lived one — e.g. a singleton holding a scoped `DbContext`. |
| Why is that bad? | The scoped instance is never released and gets used from multiple threads. |
| How do you fix it? | Inject `IServiceScopeFactory` and create a scope per unit of work. |
| What catches it at startup? | Scope validation — on by default in Development. Leave it on. |
| Only injection style the container supports? | Constructor injection. No property or method injection. |
| `GetService` vs `GetRequiredService`? | Returns null vs **throws**. Prefer the throwing one. |
| What's the DI leak to know? | A transient `IDisposable` resolved from the **root** provider lives until shutdown. |
| Why `TryAdd*` in a library? | So you don't stomp the consumer's registration or create duplicate `IEnumerable<T>` entries. |
| How do you register an open generic? | `services.AddScoped(typeof(IRepo<>), typeof(EfRepo<>))` — note the empty `<>`. |
| What are keyed services for? | Picking one of N implementations of an interface by key (.NET 8+). |
| How do you decorate a service? | Scrutor or a factory registration — the built-in container has no `Decorate`. Last applied is outermost. |
| Is `IConfiguration` typed? | No — a flat `string→string` dictionary. Every value is a string. |
| Config precedence order? | appsettings.json → appsettings.{Env}.json → user secrets → env vars → command line. **Later wins.** |
| Env-var hierarchy separator? | **`__`** (double underscore), not `:`. |
| What if `ASPNETCORE_ENVIRONMENT` is unset? | It defaults to **Production**. |
| What happens on a missing config key? | Returns **null silently** — no exception. |
| `IOptions` vs `IOptionsSnapshot` vs `IOptionsMonitor`? | Singleton read-once / scoped per-request / singleton **live** with `OnChange`. |
| Which one in a singleton? | **`IOptionsMonitor<T>`** — never `IOptionsSnapshot` (it's scoped). |
| What does `ValidateOnStart()` do? | Turns lazy validation into a **startup failure** — so bad config fails the deploy, not a request. |
| Which config sources reload? | File providers (`reloadOnChange`). **Env vars and command-line args are read once.** |
| Where do secrets live? | User Secrets in dev; a secret store (Key Vault) in prod, ideally via managed identity. |
| A secret got committed — now what? | **Rotate it.** Git history keeps the old value forever. |
| Why message templates, not interpolation? | Templates preserve structured queryable fields and skip formatting when filtered out. |
| `IHostedService` vs `BackgroundService`? | Start/stop hooks (`StartAsync` **blocks startup**) vs a long-running `ExecuteAsync` loop. |

## ASP.NET Core

| Q | A |
|---|---|
| Canonical middleware order? | Exception handler → HSTS/HTTPS → static files → routing → CORS → **authn → authz** → rate limit/cache → endpoints. |
| Why must routing come before auth? | So auth can read the endpoint's `[Authorize]` metadata. |
| How does middleware work? | Takes `HttpContext` + `next`; runs code before and after `await next()`, or short-circuits. |
| `Use` vs `Run` vs `Map`? | Calls next / terminal / branches the pipeline. |
| Class-based middleware lifetime rule? | It's built **once** — inject scoped services into `InvokeAsync`, not the constructor. |
| Minimal APIs vs MVC? | Lambdas, low ceremony, **AOT-friendly** vs controllers with a rich filter pipeline. |
| Why do Minimal APIs support AOT? | The Request Delegate Generator emits binding glue at compile time — no runtime reflection. |
| `Results` vs `TypedResults`? | `IResult` vs the concrete type — testable handlers and inferred OpenAPI. |
| What does `[ApiController]` add? | Automatic 400 ValidationProblemDetails, binding-source inference, ProblemDetails errors. |
| Minimal API binding order? | Route → query (simple types) → body (complex) → DI → special types. |
| Are route constraints validation? | **No** — they decide matching. A failed constraint is a **404**, not a 400. |
| Why bind to DTOs, not entities? | Over-posting — a client can set any bindable property (`IsAdmin`). |
| Do Minimal APIs validate DataAnnotations? | Historically **no** — add a filter, `.AddValidation()` (.NET 10), or FluentValidation. |
| Middleware vs filter vs decorator? | Raw HTTP on every request / bound args + endpoint metadata / one service's calls. |
| Exception filter or exception middleware? | **Middleware** — it catches everything, including binding and other middleware. |
| What is ProblemDetails? | RFC 7807/9457 — the standard `application/problem+json` error body. |
| Prod vs dev error detail? | Prod: generic body + a `traceId`. Never stack traces — that's information disclosure. |
| Four rate-limiting algorithms? | Fixed window, sliding window, token bucket, concurrency. |
| Rate limiting's built-in caveat? | It's **per-instance** — N replicas multiply the limit by N. |
| Cardinal output-caching bug? | Wrong or missing **vary-by** — everyone gets the first caller's response. |
| Liveness vs readiness? | Alive? → **restart** (app only). Can serve? → **stop routing** (checks dependencies). |
| Why must liveness not check the DB? | A DB blip fails it on every pod and restarts your whole fleet. |
| How do health checks give zero-downtime deploys? | On SIGTERM, fail readiness → traffic stops → drain in-flight → exit. |
| Do Minimal APIs do content negotiation? | No — JSON only. MVC does, via formatters. |
| Post-Redirect-Get — why? | So a browser refresh after a POST doesn't re-submit the form. |

## EF Core

| Q | A |
|---|---|
| What patterns does `DbContext` implement? | **Unit of Work** + its `DbSet<T>`s are **repositories**. |
| `DbContext` lifetime? | **Scoped** — one per request. It is **not thread-safe**. |
| What is the N+1 problem? | One query for parents plus one per parent for children. Fix with `Include` or projection. |
| When `AsNoTracking()`? | Read-only queries — skips the snapshot and identity resolution. |
| What is identity resolution? | One instance per key within a context. `AsNoTracking` skips it. |
| When `AsSplitQuery()`? | Multiple **collection** includes — avoids cartesian explosion. |
| `Attach` vs `Update` vs load-then-mutate? | Tracks Unchanged / tracks **all props Modified** (writes every column) / minimal + safe. |
| What does `SaveChanges` do about transactions? | It **is** one — all statements commit or roll back together. |
| When do you need an explicit transaction? | When a unit of work spans multiple saves or mixes raw SQL. |
| Explicit transaction + `EnableRetryOnFailure`? | You must use `CreateExecutionStrategy()` or it throws. |
| `ExecuteUpdate`/`ExecuteDelete`? | Set-based SQL, nothing loaded — but they **bypass the tracker and interceptors**. |
| How does optimistic concurrency work? | A rowversion token in the UPDATE's `WHERE`; 0 rows matched → `DbUpdateConcurrencyException`. |
| Conflict resolution options? | Store wins, client wins, or merge — decide deliberately. |
| Cascade delete default? | Required relationships default to **Cascade** — deleting a customer deletes their orders. |
| How should migrations run in prod? | A reviewed **idempotent script** as a controlled step — never `Database.Migrate()` across replicas. |
| Why can a rename lose data? | EF may generate `DropColumn` + `AddColumn`. Review and fix to `RenameColumn`. |
| Zero-downtime schema change? | Backward-compatible/additive, or **expand/contract** across deploys. |
| Safe raw SQL? | `FromSql`/`ExecuteSql` with an interpolated string (parameterized). `…Raw` takes a plain string → injectable. |
| What are global query filters for? | Soft delete and **multi-tenancy** — so the filter can't be forgotten. |
| What does transparent soft delete need? | **Both** a query filter (reads) and a `SaveChanges` interceptor (writes). |
| Owned type vs related entity? | A value object with no identity, mapped into the owner's table, vs its own key and `DbSet`. |
| When `IDbContextFactory`? | Parallel queries, Blazor Server, singletons/background services. |
| Why not the InMemory provider in tests? | Not relational — no constraints, no SQL translation, no real transactions. |
| Best test database? | SQLite in-memory as default; **Testcontainers** for real fidelity. |
| Why not mock `DbContext`? | A mocked `IQueryable` runs LINQ-to-Objects — it never exercises translation. |
| Most common "EF is slow" cause? | A **missing index**, not EF. |

## Data access & caching

| Q | A |
|---|---|
| When Dapper over EF? | Read-heavy, complex, or perf-critical queries where you want to own the SQL. |
| What does Dapper not do? | Change tracking, migrations, LINQ translation. |
| Key ADO.NET fact? | Connections are **pooled** — open late, dispose early; the pool is small. |
| Why avoid `AddWithValue`? | Inferred types cause implicit conversions that can defeat an index. |
| `IMemoryCache` vs `IDistributedCache`? | In-process, fastest, **per-instance** vs shared (Redis), serialization + network hop. |
| What is `HybridCache`? | .NET 9+ L1+L2 with **stampede protection**, typed serialization, and **tag invalidation**. |
| What is a cache stampede? | A hot key expires and all concurrent requests hit the database at once. |
| Does `IMemoryCache.GetOrCreateAsync` prevent it? | **No.** `HybridCache` does. |
| Absolute vs sliding expiration? | Fixed lifetime vs reset-on-access. Combine sliding with an absolute cap. |
| Why is an unbounded `IMemoryCache` dangerous? | No limit unless you set `SizeLimit` + per-entry `Size` — it grows until OOM. |
| Why not cache mutable objects? | Every caller gets the **same reference**; one mutation corrupts it for everyone. |
| Cache key must include? | Tenant, user (or explicitly not), culture, version. |
| Output caching vs data caching? | Whole responses server-side vs values inside your handler. They layer. |
| EF second-level cache caveat? | Table-granular invalidation that misses `ExecuteUpdate`, raw SQL, and other writers. |

## Messaging & background work

| Q | A |
|---|---|
| What is `Channel<T>`? | An in-process async producer/consumer queue. Bounded gives back-pressure. |
| What happens if you forget `Writer.Complete()`? | `ReadAllAsync` never finishes and the app won't shut down. |
| When move from a channel to a broker? | Durability, cross-service delivery, or independent scaling. |
| RabbitMQ routing model? | Producer → **exchange** → bindings → queue → consumer. |
| RabbitMQ exchange types? | direct (exact key), fanout (broadcast), topic (pattern), headers. |
| How do you get at-least-once in RabbitMQ? | `autoAck: false` — ack after processing. Plus durable queues **and** persistent messages. |
| Kafka vs RabbitMQ? | An append-only **replayable log** with consumer groups vs a queue where a consumed message is gone. |
| What gives ordering in Kafka? | The **partition** — via the partition key. There is no global ordering. |
| What limits Kafka consumer parallelism? | The partition count — one consumer per partition per group. |
| Send vs Publish? | A **command** to one consumer vs an **event** to all subscribers. |
| Why must consumers be idempotent? | All brokers deliver **at-least-once**. |
| What is the dual-write problem? | You can't atomically commit the DB and publish to a broker. |
| What solves it? | The **outbox** — write the message in the same transaction, relay it afterwards. |
| What's an inbox? | Consumer-side dedupe table — outbox + inbox ≈ effectively-once. |
| What is a saga? | A multi-service workflow with **compensating actions** instead of a distributed transaction. |
| How do you handle a poison message? | Retry with backoff, then **dead-letter**. Never requeue forever. Then monitor the DLQ. |
| Service Bus sessions? | Per-key FIFO ordering to one consumer, without serializing the whole queue. |
| Cardinal rule for hosted services? | Create a **DI scope per work item** — they're singletons. |
| What if `ExecuteAsync` throws? | The **host stops** by default. Wrap each iteration in try/catch. |
| Why `PeriodicTimer`? | Async, cancellable, and **non-overlapping** — unlike `while + Task.Delay`. |
| Biggest scheduled-work gotcha? | A `BackgroundService` runs in **every replica** — N nightly reports. |
| Fixes for run-once-per-cluster? | Distributed lock, leader election, Hangfire/Quartz clustering, or a K8s CronJob. |
| Why Hangfire over a channel? | Durable jobs that survive restarts, automatic retries, cluster-safe recurring, dashboard. |
| What do you pass to a Hangfire job? | **Ids and simple values** — never entities. Re-fetch inside the job. |
| Why schedule in UTC? | DST means a local "2 AM daily" runs twice one night and not at all another. |

## HTTP, gRPC & SignalR

| Q | A |
|---|---|
| Why not `new HttpClient()` per request? | Socket exhaustion — sockets linger in `TIME_WAIT` and you run out of ports. |
| Why not a static singleton? | It never picks up **DNS changes**. |
| How does `IHttpClientFactory` fix both? | It pools handlers and **rotates** them (2 min default). |
| Named or typed clients? | **Typed** — config bound to a service class, testable, no magic strings. |
| Does `HttpClient` throw on 500? | **No.** Call `EnsureSuccessStatusCode()`. |
| What is a `DelegatingHandler`? | Middleware for **outbound** HTTP. First registered is outermost. |
| `DelegatingHandler` gotcha? | Handlers are pooled and long-lived — don't capture scoped services in fields. |
| Can you reuse an `HttpRequestMessage`? | No — single-use. Clone it to retry. |
| What does `AddStandardResilienceHandler` add? | Rate limiter, total timeout, retry with backoff + jitter, circuit breaker, per-attempt timeout. |
| When gRPC over REST? | Internal service-to-service — typed contracts, compact Protobuf, HTTP/2, streaming. |
| gRPC's four call types? | Unary, server-streaming, client-streaming, bidirectional. |
| What does gRPC require? | End-to-end **HTTP/2**. Browsers need gRPC-Web. |
| What are gRPC deadlines? | A time budget propagated to the server, which sees it as cancellation. |
| Protobuf contract rule? | Only add fields with new numbers; never reuse a retired number (`reserved`). |
| SignalR's #1 scaling gotcha? | Multi-instance needs a **Redis backplane** (or Azure SignalR) or messages don't cross instances. |
| What's lost on a SignalR reconnect? | **Group membership** — it's per connection id. Re-join. |
| Classic raw TCP bug? | Treating one `Read` as one message. TCP is a byte stream — you must **frame**. |
| What is `System.IO.Pipelines` for? | High-throughput protocol parsing — buffer pooling, partial reads, back-pressure. |

## Identity & security

| Q | A |
|---|---|
| Authentication vs authorization? | Who are you (produces a `ClaimsPrincipal`) vs what may you do. |
| 401 vs 403? | Not authenticated vs authenticated but forbidden. |
| Is a JWT encrypted? | **No — signed.** The payload is readable by anyone. Never put secrets in it. |
| What must you validate on a JWT? | **Signature, `exp`, `iss`, `aud`** — all four. |
| RS256 vs HS256? | Asymmetric (IdP signs, many validate via JWKS, no shared secret) vs symmetric. |
| How do you revoke a JWT? | You largely can't — use short access tokens + revocable refresh tokens. |
| OAuth vs OIDC? | Delegated **authorization** vs OAuth **plus authentication** (the ID token) and discovery. |
| Which OAuth flow? | **Authorization Code + PKCE** for all user-facing apps; client credentials for service-to-service. |
| What does PKCE prevent? | Authorization-code interception — binding the code to the client that requested it. |
| Access token vs ID token? | Call an API vs authenticate the user to *your* app. Don't send an ID token to an API. |
| Cookie auth vs bearer tokens? | Browser-automatic, opaque, **revocable**, **CSRF-prone** vs explicit header, stateless, CSRF-free. |
| Which needs CSRF protection? | **Cookie auth.** Bearer tokens don't — the header isn't sent automatically. |
| Three cookie flags? | `HttpOnly` (XSS), `Secure` (HTTPS), `SameSite` (CSRF). |
| How do antiforgery tokens work? | A token in the page that a cross-origin attacker can't read (same-origin policy). |
| Classic multi-instance auth bug? | The **Data Protection key ring** isn't persisted to shared storage → random logouts. |
| What else breaks without a shared key ring? | Antiforgery tokens, password-reset and email-confirmation tokens, TempData. |
| Roles or policies? | **Policies over claims** — roles are coarse and cause role explosion. |
| When resource-based authorization? | When the decision depends on the specific instance ("may they edit *this* document?"). |
| Are endpoints protected by default? | **No — public.** Set a fallback policy requiring authentication. |
| How should passwords be stored? | A slow **salted KDF** — PBKDF2, bcrypt, Argon2. Never a fast hash. |
| Why `FixedTimeEquals`? | Ordinary comparison short-circuits, leaking how many bytes were correct via timing. |
| What does `IClaimsTransformation` run per? | **Every request** — keep it cheap and idempotent. |
| Should you build your own login? | No — delegate to an IdP over OIDC, or use ASP.NET Core Identity. |

## Resilience

| Q | A |
|---|---|
| Resilience vs reliability? | Working despite failure vs not failing in the first place. |
| What is cascading failure? | A slow dependency ties up callers' threads until the whole system stops responding. |
| How do you prevent it? | Turn slow failures into fast ones — timeouts, circuit breakers, bulkheads, fallbacks. |
| Circuit breaker states? | **Closed → Open → Half-open.** |
| What does a breaker actually buy you? | It frees *your* resources and takes load off the recovering dependency. |
| Correct strategy order? | Total timeout → retry → circuit breaker → per-attempt timeout → call. |
| Why build the pipeline once? | Breaker state lives in the pipeline — a per-call pipeline **never trips**. |
| When is retry safe? | Idempotent operations, or writes with an **idempotency key**. |
| Why jitter? | Without it, every client retries at the same instant — a synchronized thundering herd. |
| Should you retry 4xx? | No (except **429**, honoring `Retry-After`). |
| Cardinal timeout trap? | Not propagating the `CancellationToken` — the timeout fires, the work keeps running. |
| Why per-attempt *and* total timeouts? | Bound one try, and bound the whole operation including retries and backoff. |
| What is hedging? | Extra parallel attempts after a short delay, first response wins — for **tail latency**. |
| When must you not hedge? | Non-idempotent operations, and any system near capacity. |
| What is a bulkhead? | Isolated resource pools so one dependency can't consume them all. |
| Is Polly enough? | No — you also need idempotency, the outbox, sagas, health checks and graceful shutdown. |

## Observability

| Q | A |
|---|---|
| Three pillars and their roles? | Metrics **detect**, traces **localize**, logs **explain**. |
| Monitoring vs observability? | Watching known problems vs being able to investigate unknown ones. |
| Why templates, not interpolation? | Structured queryable fields, and no formatting cost when filtered out. |
| What is metric cardinality? | Each unique dimension-value combination is a separate time series. |
| Where do high-cardinality ids belong? | On **trace spans** or in logs — never as metric dimensions. |
| Why percentiles over averages? | An average hides the tail; p99 is what your unhappiest users experience. |
| How does a trace cross services? | **W3C Trace Context** (`traceparent`) — automatic over HTTP, **manual over messaging**. |
| What does instrumentation cost when unused? | Nearly nothing — `StartActivity` returns `null` and meters no-op. |
| Relationship between .NET APIs and OTel? | `ILogger`/`Meter`/`ActivitySource` **are** the OTel APIs. |
| What is RED? | Rate, Errors, Duration — the service golden signals. |
| What is USE? | Utilization, Saturation, Errors — for resources. |
| What does OTel auto-instrumentation give you? | RED metrics per route, runtime metrics, and end-to-end traces, for a few lines. |
| Why sample traces? | Cost and volume — but use **tail sampling** so you keep errors and slow traces. |
| Where should containers log? | **JSON to stdout.** Never to files. |
| First tool for triage? | **`dotnet-counters`.** |
| How do you find a leak? | Two `dotnet-gcdump` snapshots, diff them, then **`gcroot`** for the retention path. |
| Signature of thread-pool starvation? | Thread count **and** queue length both climbing while CPU is low. |

## Testing

| Q | A |
|---|---|
| What's distinctive about xUnit? | A **new class instance per test** — setup in the constructor, teardown in `Dispose`. |
| `[Fact]` vs `[Theory]`? | A single test vs data-driven via `[InlineData]`/`[MemberData]`. |
| How do you share expensive setup? | `IClassFixture<T>` (per class), `ICollectionFixture<T>` + `[Collection]` (across classes). |
| Why never `async void` tests? | The runner can't await them — failures vanish and the test reports green. |
| Stub vs mock vs fake? | Canned values / also verifies interactions / a simplified working implementation. |
| What shouldn't you mock? | Types you don't own, value objects, the SUT, and `DbContext`. |
| How do you test `HttpClient` code? | Mock the **`HttpMessageHandler`**, not the client. |
| What does `WebApplicationFactory` catch? | Middleware order, DI wiring, routing, binding, serialization — the full pipeline. |
| `ConfigureServices` or `ConfigureTestServices`? | **`ConfigureTestServices`** — it runs after the app's registrations, so it overrides. |
| How do you test `[Authorize]` endpoints? | Register a test authentication handler injecting a known principal. |
| Best database test strategy? | SQLite in-memory by default; **Testcontainers** for real fidelity. |
| SQLite in-memory gotcha? | The database dies when the connection closes — hold it open for the fixture. |
| How do you isolate integration tests? | Transaction rollback, Respawn, or a fresh database per class. Pick one. |
| Why BenchmarkDotNet over `Stopwatch`? | JIT warmup, dead-code elimination, GC noise, and proper statistics. |
| Two benchmarking musts? | **Release, not attached**, and **return your results**. |
| What is property-based testing for? | Invariant-rich code — parsers, serializers, algorithms. Its killer feature is **shrinking**. |
| Is coverage a good target? | It's a good way to find gaps and a bad target. Mutation testing measures assertion quality. |
| Pyramid or trophy? | Both say keep **E2E few**; the trophy weights integration heaviest. |

## Architecture & best practices

| Q | A |
|---|---|
| Default architecture for a new system? | A **modular monolith** — real boundaries, one deployable, splits later. |
| What is the Dependency Rule? | Source dependencies point **inward**; the domain depends on nothing. |
| Is an anemic model always wrong? | No — it's fine for CRUD. Match richness to actual business complexity. |
| What is an aggregate? | A consistency boundary accessed via a root that guards its invariants. |
| Four aggregate rules? | Protect true invariants together; keep it small; reference others **by id**; one per transaction. |
| Should you wrap EF in a repository? | Not generically — `DbContext` is already UoW + repository. A per-aggregate one, maybe. |
| Why is `Repository<T>` over EF bad? | It either leaks `IQueryable` (abstracts nothing) or hides `Include`/projections/`AsNoTracking`. |
| What is CQRS? | Separate read and write models because they have opposite needs. A spectrum, not a switch. |
| When is CQRS overkill? | CRUD, where the read and write models are the same shape. |
| Domain event vs integration event? | In-process, same transaction vs cross-service via a broker, eventually consistent. |
| Who dispatches domain events? | **Infrastructure, after persistence** — not the aggregate itself. |
| What is vertical slice architecture? | Organize by feature, not layer — code that changes together lives together. |
| Its trade-off? | Some duplication, accepted over a premature shared abstraction. |
| Multi-tenancy isolation models? | Per-row (`TenantId`), per-schema, per-database. |
| Biggest multi-tenancy risk? | **Cross-tenant leakage** from one forgotten filter. Use global query filters. |
| Where must the tenant id come from? | A **verified** claim — never an unvalidated header or body field. |
| Why must feature flags be runtime-changeable? | Otherwise you've kept the complexity and lost the deploy/release decoupling. |
| What counts as a breaking change? | Anything that makes a working consumer stop working — including adding an interface member. |
| Top .NET anti-patterns? | Service locator, anemic domain, sync-over-async, `async void`, fat controllers, generic repo, captive deps, N+1. |
| Cardinal async rule at scale? | **Never block on async.** Async all the way, propagate `CancellationToken`. |
| `ConfigureAwait(false)` — where? | Library code (portability). It's a **no-op** in ASP.NET Core, and wrong in UI code. |

## Deployment & Aspire

| Q | A |
|---|---|
| What is .NET Aspire? | A cloud-native composition layer — describe the app graph in C#, get discovery/telemetry/health/dashboard. |
| What is Aspire *not*? | A runtime, or a production orchestrator. It's the glue and control panel. |
| What does `WithReference` do? | Injects the resource's connection string or endpoint URL into the consumer's config. |
| Hosting vs client integration? | AppHost provisions the resource; the service consumes it. The **resource name** ties them. |
| What's in ServiceDefaults? | OpenTelemetry, health checks, service discovery, HTTP resilience — and it's source you own. |
| ReadyToRun vs Native AOT? | Faster startup with **no compatibility cost** vs fastest startup with heavy restrictions. |
| Why copy the csproj before the source in a Dockerfile? | So the `restore` layer caches and doesn't re-run on every code change. |
| What are chiseled images? | Distroless-style .NET base images — no shell, no package manager, **non-root**, few CVEs. |
| Why can't a chiseled container bind port 80? | It runs as non-root. Use 8080. |
| How do K8s probes map to health checks? | Liveness → `/alive` (restart), readiness → `/health` (stop routing), plus a startup probe. |
| Are Kubernetes Secrets encrypted? | **No — base64-encoded.** Enable encryption at rest or use an external store. |
| Why never tag an image `latest`? | You can't tell what's deployed and can't roll back deterministically. |
| What does Helm add over `kubectl`? | Templating with per-environment values, and versioned releases with a real `rollback`. |
| Zero-downtime on App Service? | **Deployment slots** — deploy to staging, warm up, **swap**. |
| What must be externalized to scale out? | Session, cache, uploaded files, and the **Data Protection key ring**. |

## Azure

| Q | A |
|---|---|
| How should a .NET app authenticate to Azure? | `DefaultAzureCredential` + **managed identity** — same code in dev and prod. |
| Why is managed identity better than a connection string? | No secret to store, leak or rotate — and it solves the bootstrapping problem. |
| Azure SDK client lifetime? | **Singleton** — thread-safe and connection-holding. |
| Auth succeeded but everything 403s — why? | A missing **RBAC role assignment**. |
| How do Key Vault secrets reach your code? | It registers as a **configuration provider** — they become ordinary `IConfiguration` keys. |
| Which options accessor for a rotated secret? | **`IOptionsMonitor<T>`** — `IOptions<T>` never reloads. |
| Key Vault or App Configuration? | Secrets in Key Vault; non-secret settings and feature flags in App Configuration. |
| Most important Cosmos decision? | The **partition key** — high cardinality, even distribution, query-aligned. Can't change it easily. |
| Cosmos default consistency? | **Session** — read-your-writes within a session. |
| Cheapest Cosmos read? | A **point read** — id + partition key. |
| Service Bus vs Event Grid vs Event Hubs? | Reliable messaging vs lightweight notifications vs a high-volume replayable stream. |
| How do you let a client upload a huge file? | A **user-delegation SAS** — direct to storage, bypassing your API. |
| Functions cold starts — mitigation? | Premium/Flex plan, or Native AOT. |
| Modern path into Application Insights? | **OpenTelemetry** instrumentation. |

## Client-side

| Q | A |
|---|---|
| Blazor Server vs WebAssembly? | C# on the server over a SignalR circuit (tiny download, round-trip latency, per-user server state) vs in the browser (large download, offline, no server state). |
| Blazor Server's scaling constraint? | A circuit and server state **per connected user** — plus a backplane and sticky sessions. |
| Is Blazor WebAssembly code private? | **No** — it's downloaded to the browser. No secrets, no trusted logic. |
| Where does JS interop belong? | **`OnAfterRenderAsync`** — the DOM doesn't exist before, and it's skipped during prerender. |
| `OnInitializedAsync` vs `OnParametersSetAsync`? | Once, ever vs on **every parameter change** — so parameter-dependent loading goes in the latter. |
| Biggest Blazor memory leak? | Subscribing to a state container event and not unsubscribing in `Dispose`. |
| What does `AddScoped` mean in Blazor Server? | **Per circuit — per user.** Never make per-user state a singleton. |
| Biggest Blazor list-performance win? | **`<Virtualize>`**, plus `@key` for stable identity. |
| How does MAUI render? | Cross-platform controls map to **real native widgets** via handlers. |
| What do WPF, WinUI, Avalonia and MAUI share? | XAML, data binding, and **MVVM**. |
| What makes MVVM work? | `INotifyPropertyChanged` — source-generated by CommunityToolkit.Mvvm's `[ObservableProperty]`. |
| EF Core in Blazor Server? | **`IDbContextFactory`** — components are long-lived, `DbContext` must not be. |
| How do you test Blazor components? | **bUnit** in-memory; pair with a small Playwright suite for real-browser journeys. |

---

→ Back to the [topic map](README.md) · Last-minute tables: [CheatSheet.md](CheatSheet.md)
