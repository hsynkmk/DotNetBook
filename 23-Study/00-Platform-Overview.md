# 00 — Platform Overview

> Short by design. This is the "warm-up question" file — the things an interviewer asks in the
> first five minutes to check you know what you're standing on.

## ⚡ 30-second answer

**.NET is a platform**: a runtime (**CLR**), a standard library (**BCL**), frameworks, and the
SDK/tooling. **C# is a language** that targets it. Your C# compiles to **IL**, which the runtime
**JITs** (or AOT-compiles) to native code and runs as **managed** code with garbage collection,
type safety and interop. Since **.NET 5** there is **one unified .NET** for web, desktop, mobile
and cloud — cross-platform and open source, on a **November release cadence** where **even
versions are LTS** (3 years) and odd are STS (18 months). **.NET 10** is the current LTS. The old
**.NET Framework 4.8** is Windows-only and maintenance-only: don't start new projects on it.

---

## Core mechanics

### The stack

```text
your C#  →  Roslyn  →  IL + metadata (an assembly)  →  CoreCLR: JIT → native
                                                    ↘  or Native AOT: native at publish
```

- **SDK builds; runtime runs.** The SDK includes the runtime plus Roslyn, MSBuild, the `dotnet`
  CLI, templates and NuGet.
- Runtimes come in families: **.NET Runtime** (console), **ASP.NET Core Runtime** (web), **.NET
  Desktop Runtime** (WPF/WinForms).
- Versions install **side-by-side**; pin one per repo with **`global.json`**.
- `dotnet --list-sdks`, `--list-runtimes`, `--info`.

### History, in one line each

| Era | What |
|---|---|
| **.NET Framework** (2002–2019) | Windows-only, closed. Final version **4.8**, maintenance-only. |
| **.NET Core** (2016–2019) | The cross-platform, open-source, side-by-side rewrite. |
| **.NET 5+** (2020–) | Unified into "just .NET". One release every November. |

**Even = LTS (~3 years), odd = STS (~18 months).** Target the latest LTS in production.

### Three runtimes, one BCL

| Runtime | Where | Trade |
|---|---|---|
| **CoreCLR** | server, desktop, cloud (default) | JIT + tiering + **dynamic PGO** → best steady-state throughput |
| **Mono** | mobile (MAUI), Blazor WebAssembly, games | portable, AOT-capable, smaller footprint |
| **Native AOT** | CLIs, serverless, high-density services | instant startup, small binary; **no JIT**, restricted reflection |

You usually don't *pick* a runtime — the **workload selects it**. You only opt into AOT
explicitly. Code is largely portable across all three; the chief exception is Native AOT's
reflection and codegen restrictions, mitigated by source generators
([01](01-Runtime-GC-JIT.md)).

### The ecosystem map

| Area | Pieces |
|---|---|
| Web & services | ASP.NET Core, gRPC, SignalR ([04](04-AspNetCore.md), [09](09-Http-gRPC-SignalR.md)) |
| Data | EF Core, Dapper, caching ([05](05-EFCore.md), [06](06-DataAccess-Caching.md)) |
| Background & messaging | hosted services, Channels, brokers ([07](07-Messaging.md), [08](08-BackgroundProcessing.md)) |
| Client UI | Blazor, MAUI, WPF/WinUI ([14–16](14-16-ClientSide.md)) |
| Cross-cutting | resilience, observability, configuration ([11](11-Resilience.md), [12](12-Observability.md), [03](03-Hosting-DI-Config.md)) |
| Composition | **.NET Aspire** ([18–19](18-19-Aspire-Deployment.md)) |

The **Generic Host** (DI + configuration + logging) is the shared backbone underneath essentially
all of it — which is why [03](03-Hosting-DI-Config.md) is the highest-leverage file in this set.

---

## 🪤 Traps & gotchas

- **Saying ".NET" when you mean C#** (or the reverse) — the platform runs F# and VB too, and C#
  targets other things. Small, but interviewers notice.
- **Confusing .NET Framework, .NET Core and .NET 5+** — "we're on .NET 4.8" and "we're on .NET 8"
  are completely different worlds. Framework is Windows-only and closed to new features.
- **Assuming .NET Standard still matters** — it was the portability contract *between* Framework
  and Core. With unified .NET it's largely historical; you target `net10.0` and multi-target only
  for library compatibility.
- **Starting a new project on .NET Framework** — maintenance-only, no new features, Windows-only.
- **Targeting an STS release for a long-lived production service** — 18 months of support means an
  upgrade project sooner than you planned.
- **Forgetting `global.json`** — a developer with a newer SDK silently builds differently from CI.
- **Thinking Native AOT is just "a faster build option"** — it changes what your code is allowed to
  do (no `Reflection.Emit`, no `dynamic`, trimming implied).
- **Assuming the ASP.NET Core Runtime is installed** because the .NET Runtime is — they're separate
  packages, and it's a classic first-deploy failure.

---

## ❓ Likely questions

**Q: What's the difference between .NET and C#?**
A: .NET is the platform — the runtime, the base class library, the frameworks and the tooling. C#
is a language that compiles to IL and targets that platform, alongside F# and VB.

**Q: What happens when you build and run a C# program?**
A: Roslyn compiles your source to **IL plus metadata** in an assembly. At runtime CoreCLR loads
it, the **JIT** compiles each method to native code on first call (or Native AOT did that at
publish time), and it runs as **managed** code — with garbage collection, type safety, and
interop to native code when needed.

**Q: .NET Framework vs .NET Core vs .NET 5+?**
A: Framework is the original Windows-only, closed-source platform, ending at 4.8 and now
maintenance-only. .NET Core was the cross-platform open-source rewrite that could run side by
side. .NET 5 unified them into a single "just .NET", which is what everything since means.

**Q: What is the release cadence and which version should you target?**
A: One release every November. Even-numbered versions are **LTS** with about three years of
support; odd ones are **STS** with about eighteen months. For production, target the latest LTS —
currently **.NET 10**.

**Q: What are the .NET runtimes?**
A: **CoreCLR** is the default for servers, desktop and cloud, with a JIT, tiered compilation and
dynamic PGO for the best steady-state throughput. **Mono** powers mobile via MAUI, Blazor
WebAssembly, and Unity. **Native AOT** compiles ahead of time for instant startup and a small
binary, at the cost of no JIT and restricted reflection. They share one BCL and toolchain.

**Q: SDK or runtime — what does a build server need, and what does a production host need?**
A: The build server needs the **SDK** (which includes a runtime, plus Roslyn, MSBuild and the
CLI). A production host running a framework-dependent app needs only the matching **runtime** —
and for a web app, specifically the **ASP.NET Core Runtime**. A self-contained or AOT-published
app needs neither.

**Q: What is `global.json` for?**
A: Pinning the SDK version for a repository, so every developer and CI agent builds with the same
toolchain rather than whatever they happen to have installed.

**Q: Is .NET Standard still relevant?**
A: Mostly historical. It existed so a library could target both .NET Framework and .NET Core. With
unified .NET you target `net10.0` and only multi-target when you must still support Framework
consumers.

---

## 🎓 Senior Extra

- **"Managed" is a precise claim**, and worth being able to unpack: the runtime owns memory
  (garbage collection), enforces type and memory safety, and mediates access to native code
  through marshaling — which is why a `NullReferenceException` is an exception rather than a
  segfault.
- **Side-by-side installation was the point of .NET Core.** The Framework's single machine-wide
  version was what made upgrades a company-wide event; side-by-side plus self-contained publishing
  is why an upgrade is now a per-service decision.
- **The November cadence is a planning tool.** Adopting each LTS on release gives you a predictable
  three-year window; skipping to every other LTS means a bigger jump each time, and there's a real
  argument either way.
- **Performance is a headline feature of each release now** — every version ships measurable
  throughput and allocation improvements, so "upgrade the runtime" is frequently the cheapest
  performance work available ([21](21-Performance-Tooling.md)).
- **The unification isn't total.** WPF and WinForms remain Windows-only, and Blazor WebAssembly
  runs on Mono in the browser sandbox — "one .NET" means one BCL, SDK and language, not one set of
  capabilities everywhere.
- **Source generators are the platform's current direction**: serialization, logging, regex,
  P/Invoke, configuration binding and Minimal API routing all moved from runtime reflection to
  compile-time generation. That's what makes AOT and trimming viable, and it's a good thing to name
  when asked "what's changed recently in .NET?"

→ Deeper: [`../00-Overview/`](../00-Overview/README.md)
