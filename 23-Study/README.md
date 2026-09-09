# Chapter 23 — Study & Interview Prep

> The whole of `DotNetBook/` — 23 chapters, 224 topic files, ~42,600 lines — distilled into a set
> you can read in **2 days**. Self-contained: you only need this folder. Weighted toward what a
> **backend C#/.NET role** actually asks, with the client-side stack kept as breadth insurance.

---

## How the files are organized

Every concept file has the same six sections:

1. **⚡ 30-second answer** — what you say out loud first.
2. **Core mechanics** — the bullets and the minimum code that matter.
3. **Comparison tables** — the side-by-sides interviewers ask for.
4. **🪤 Traps & gotchas** — the wrong-answer pitfalls, with find-the-bug snippets.
5. **❓ Likely questions** — crisp Q→A you can fire back.
6. **🎓 Senior Extra** — the internals and judgment calls that separate a *correct* answer from a
   *senior* one.

**The aim**: know the ⚡30-second answer and the 🪤traps for every file. Those two sections carry
most of the value; the rest is there when you want the depth.

Each file ends with a `→ Deeper:` link back to the full chapter, but you don't need it — this
folder stands alone.

## Quick reference — use daily and morning-of

| File | Use |
|---|---|
| [**RapidFire.md**](RapidFire.md) | **250 one-line Q&A** — cover the answers, quiz yourself out loud |
| [**CheatSheet.md**](CheatSheet.md) | Every table, the top 45 traps, power phrases, and the answer protocol |

---

## Topic map

### The core five — where most interview time goes

| # | File | Covers |
|---|---|---|
| 03 | [**Hosting, DI & Config**](03-Hosting-DI-Config.md) | lifetimes, **captive dependency**, scopes, `IConfiguration` layering, Options, `ValidateOnStart` |
| 04 | [**ASP.NET Core**](04-AspNetCore.md) | **middleware order**, Minimal APIs vs MVC, binding, validation, ProblemDetails, rate limiting, health checks |
| 05 | [**EF Core**](05-EFCore.md) | UoW + repository, **N+1**, tracking, migrations, concurrency, filters, testing |
| 01 | [**Runtime: CLR, JIT & GC**](01-Runtime-GC-JIT.md) | tiered compilation, **dynamic PGO**, generational **GC**, LOH, thread pool, AOT |
| 22 | [**Best Practices & Architecture**](22-BestPractices-Architecture.md) | modular monolith, aggregates, **CQRS**, outbox, multi-tenancy, **the anti-pattern list** |

### Services, data & integration

| # | File | Covers |
|---|---|---|
| 02 | [BCL Essentials](02-BCL-Essentials.md) | strings, `Span<T>`, System.Text.Json, `TimeProvider`, crypto primitives |
| 06 | [Data Access & Caching](06-DataAccess-Caching.md) | Dapper, ADO.NET pooling, `IMemoryCache`/`IDistributedCache`/**`HybridCache`**, stampede |
| 07 | [Messaging](07-Messaging.md) | `Channel<T>`, RabbitMQ, Kafka, Service Bus, **idempotency**, **the outbox** |
| 08 | [Background Processing](08-BackgroundProcessing.md) | `BackgroundService`, **scope per work item**, scheduling across replicas, Hangfire/Quartz |
| 09 | [HTTP, gRPC & SignalR](09-Http-gRPC-SignalR.md) | **`IHttpClientFactory`**, delegating handlers, resilience handler, gRPC, SignalR backplane |
| 10 | [Identity & Security](10-Identity-Security.md) | authn vs authz, **JWT validation**, OAuth/OIDC + PKCE, cookies vs bearer, **Data Protection** |

### Cross-cutting & quality

| # | File | Covers |
|---|---|---|
| 11 | [Resilience](11-Resilience.md) | cascading failure, retry + **jitter**, circuit breaker, **cooperative timeouts**, hedging |
| 12 | [Observability](12-Observability.md) | the three pillars, structured logging, **metric cardinality**, traces, OpenTelemetry, `dotnet-*` |
| 17 | [Testing](17-Testing.md) | xUnit lifecycle, mocking boundaries, **`WebApplicationFactory`**, Testcontainers, BenchmarkDotNet |
| 21 | [Performance & Tooling](21-Performance-Tooling.md) | measure-first, `dotnet-counters`/`-trace`/`-gcdump`, leak hunting, the five bottlenecks |

### Ship it

| # | File | Covers |
|---|---|---|
| 18–19 | [Aspire & Deployment](18-19-Aspire-Deployment.md) | AppHost, ServiceDefaults, publish modes, Docker layer caching, chiseled images, K8s probes |
| 20 | [Azure Integration](20-Azure.md) | **managed identity**, Key Vault, App Configuration, Cosmos partition keys, the three brokers |

### Breadth — low frequency, but a visible gap if missed

| # | File | Covers |
|---|---|---|
| 00 | [Platform Overview](00-Platform-Overview.md) | .NET vs C#, LTS cadence, the three runtimes, SDK vs runtime |
| 14–16 | [Client-Side](14-16-ClientSide.md) | Blazor **render modes**, component lifecycle, MAUI handlers, WPF/WinUI, MVVM |

---

## The 2-day plan

| Day | Morning | Afternoon | Evening |
|---|---|---|---|
| **1** | **03** Hosting/DI/Config → **04** ASP.NET Core | **05** EF Core → **06** Data & Caching | **09** HTTP/gRPC → **01** Runtime/GC |
| **2** | **10** Identity → **11** Resilience → **12** Observability | **07** Messaging → **08** Background → **17** Testing | **22** Architecture → breadth (**00**, **02**, **18–19**, **20**, **21**, **14–16**) |

Then finish with **[RapidFire.md](RapidFire.md)** and **[CheatSheet.md](CheatSheet.md)**.

**Each session**: read the file → re-read its **⚡30-second answer** and **🪤traps** → say the
answers in **❓Likely questions** out loud before reading them. Memorize the *mental model* and the
*traps*, never code verbatim.

> **Short on time?** Day 1 plus `CheatSheet.md` and `RapidFire.md` covers the large majority of
> what actually gets asked. **03, 04, 05** alone are worth more than everything else combined.

---

## ⏰ Morning-of — a 90-minute skim

1. **CheatSheet.md** (15 min) — every table plus the power phrases.
2. **RapidFire.md** (35 min) — answer every question **out loud**, star the misses.
3. **The traps that end interviews** (25 min) — re-read 🪤 in **03** (captive dependency),
   **04** (middleware order), **05** (N+1, tracking), **09** (socket exhaustion),
   **11** (cooperative timeouts).
4. **Your starred misses** (15 min).

---

## How to talk in the interview

- **Restate** the question → **clarify** an assumption → answer the **direct question first**,
  then add depth.
- **"X vs Y"**: one-sentence difference → when you'd use each → **a trap**.
- **Design questions**: clarify scale → the simplest thing that works → what changes at 10× →
  the failure mode you're guarding against.
- **Coding**: state the approach and Big-O **before** typing; verify with an example after.
- **Don't bluff internals.** Reason aloud and state your assumptions — that reads as senior;
  a confident wrong answer does not.

---

## Relationship to the rest of the book

Every file links back with `→ Deeper:` to the chapter it was distilled from, if you want the full
treatment of something. The sibling **`../../CSharpBook/18-Study/`** covers the **language** side
(type system, LINQ, async internals, collections, equality) — this set covers the **platform**, and
goes considerably deeper on it. There's some deliberate overlap on the basics; repetition helps
recall, and you shouldn't be jumping between folders the night before.

→ Start here: [**03 — Hosting, DI & Configuration**](03-Hosting-DI-Config.md)
