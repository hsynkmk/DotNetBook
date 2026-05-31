# History — From .NET Framework to .NET 10

## Why the history matters

You'll encounter all eras of .NET in real codebases, job descriptions, and Stack Overflow answers. Knowing the lineage tells you which advice is current, what's deprecated, and why some things are named confusingly (looking at you, ".NET Standard"). The short version: **there was Windows-only .NET Framework, then a cross-platform rewrite called .NET Core, and since .NET 5 they merged into one "just .NET."**

---

## The timeline

```
2002  .NET Framework 1.0      Windows-only, closed source
...    (Framework 2.0–4.8)     matured; 4.8 (2019) is the LAST Framework version
2016  .NET Core 1.0           cross-platform, open source, rewritten from scratch
2018  .NET Core 2.x / 3.x     fast, broad; 3.1 was the first Core LTS
2020  .NET 5                  "Core" dropped from the name; unification begins
2021  .NET 6 (LTS)            unified, mature; minimal APIs, hot reload
2022  .NET 7 (STS)            perf, generic math
2023  .NET 8 (LTS)            Native AOT matured, Blazor unification, Aspire preview
2024  .NET 9 (STS)            Aspire GA, perf, more AOT
2025  .NET 10 (LTS)  ← today  C# 14, runtime gains, the current baseline
```

---

## Era 1 — .NET Framework (2002–2019)

The original .NET: **Windows-only**, closed source, installed machine-wide (one shared copy in `C:\Windows\Microsoft.NET`). It powered WinForms, WPF, ASP.NET (the old "System.Web"), and WCF.

- **Strengths**: huge ecosystem, deep Windows integration, stable.
- **Limitations**: Windows-only; tied to the OS (you couldn't run two app versions needing different Framework versions side-by-side easily); slow release cadence; closed.
- **Status today**: **.NET Framework 4.8 is the final version.** It's still supported (it ships with Windows) and maintained for security, but it gets **no new features**. Don't start new projects on it. Migrate when feasible.

You'll still find Framework in legacy enterprise apps, especially WCF services and old WebForms sites.

---

## Era 2 — .NET Core (2016–2019)

Microsoft rewrote .NET from scratch as **.NET Core**: cross-platform (Windows/Linux/macOS), open source, modular (NuGet-delivered), and **side-by-side** (each app carries or references a specific runtime version — no machine-wide single version).

- **.NET Core 1.0 (2016)** — minimal but cross-platform.
- **.NET Core 2.x (2017–18)** — broad API parity, much faster.
- **.NET Core 3.x (2019)** — added desktop (WPF/WinForms on Windows), and **3.1 was the first Core LTS**.

This is when ASP.NET Core and EF Core (the "Core" rewrites) appeared — leaner, faster, cross-platform replacements for the Framework versions.

---

## Era 3 — Modern .NET (2020–today)

With **.NET 5 (2020)**, Microsoft dropped "Core" from the name (and skipped "4" to avoid confusion with Framework 4.x). The goal: **one unified .NET** for all workloads, one release per year.

The cadence settled into a predictable pattern:

| Version | Year | Support | Notable |
|---|---|---|---|
| .NET 5 | 2020 | (ended) | Unification begins, single BCL |
| .NET 6 | 2021 | LTS (ended 2024) | Minimal APIs, hot reload, `record struct` |
| .NET 7 | 2022 | STS (ended) | Generic math, perf, container tooling |
| .NET 8 | 2023 | **LTS** (to Nov 2026) | Native AOT matured, Blazor unified, keyed DI |
| .NET 9 | 2024 | STS (to mid-2026) | Aspire GA, more AOT, perf |
| **.NET 10** | **2025** | **LTS** (to Nov 2028) | **C# 14, runtime gains — this book's baseline** |

### The LTS / STS cadence

- **Even versions (6, 8, 10)** → **LTS (Long-Term Support)**, ~3 years of support.
- **Odd versions (7, 9, 11)** → **STS (Standard-Term Support)**, ~18 months.
- One major release every **November**.

**Rule of thumb**: production projects target the latest **LTS** (today, .NET 10). Adopt STS only if you need a specific feature and can keep upgrading. See [04-WhatsInTheBox.md](04-WhatsInTheBox.md) for picking versions.

---

## The ".NET Standard" detour (and why it's fading)

During the Framework↔Core split, code couldn't easily target both. **.NET Standard** was a *specification* — a common API surface that Framework, Core, Mono, and Xamarin all implemented. A library targeting `netstandard2.0` ran everywhere.

```
A library targeting netstandard2.0 can be consumed by:
.NET Framework 4.6.1+   .NET Core 2.0+   Mono   Xamarin   Unity
```

Since unification, you can mostly just target `net10.0` (or multi-target). **.NET Standard is now legacy** — still useful for libraries that must support .NET Framework consumers, but for new code targeting modern .NET only, target the version directly. `netstandard2.0` remains the lowest common denominator for maximum reach. (See CSharpBook Chapter 15 on target frameworks.)

---

## What's deprecated / what to use today

| Old (avoid for new work) | Modern replacement |
|---|---|
| .NET Framework 4.8 | .NET 10 |
| ASP.NET (System.Web, WebForms) | ASP.NET Core |
| WCF (server) | gRPC, ASP.NET Core, CoreWCF (community) |
| Web API 2 / MVC 5 (Framework) | ASP.NET Core MVC / Minimal APIs |
| EF6 | EF Core 10 |
| `System.Web.Http`, `HttpClient` patterns | `IHttpClientFactory` |
| `.NET Standard` (new code) | target `net10.0` directly |
| Newtonsoft.Json (default) | System.Text.Json (still use Newtonsoft when needed) |
| AppDomains | `AssemblyLoadContext` |
| Remoting | gRPC / HTTP / messaging |

---

## Why the rewrite happened

The original Framework couldn't evolve fast enough: Windows-only blocked cloud/Linux/containers, the machine-wide install model caused "DLL hell," and the closed, slow-release model lagged competitors. The Core rewrite delivered:
- **Cross-platform** (cloud and containers need Linux).
- **Side-by-side / self-contained** deployment (no global version conflicts).
- **Open source** (community trust and contribution).
- **Performance** (each release is measurably faster; see [Chapter 21](../21-Performance/README.md)).
- **Yearly cadence** (predictable, fast feature delivery).

---

## Summary

- **.NET Framework (2002–2019, Windows-only, closed)** → final version **4.8**, maintenance-only, don't start new projects on it.
- **.NET Core (2016–2019)** was the cross-platform, open-source, side-by-side rewrite.
- **.NET 5+ (2020–today)** unified everything into "just .NET," one release each November.
- **Even = LTS (~3 yr), odd = STS (~18 mo).** Target the latest LTS — **.NET 10** today.
- **.NET Standard** was a compatibility spec for the split era; now legacy except for libraries needing Framework reach.
- Prefer the modern stack (ASP.NET Core, EF Core, System.Text.Json, `IHttpClientFactory`, `AssemblyLoadContext`) over Framework-era equivalents.

→ Next: [03-Runtimes.md](03-Runtimes.md)
