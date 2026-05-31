# Chapter 00 — Overview — Q & A

---

### Q1. What's the difference between .NET and C#?

.NET is a **platform** — runtime (CLR), standard library (BCL), frameworks, and tooling. C# is a **language** that compiles to IL targeting that platform (F# and VB also target it). This book covers the platform; CSharpBook covers the language.

---

### Q2. What is IL and why does it exist?

Intermediate Language — the CPU-independent bytecode the C# compiler produces. The runtime JIT-compiles IL to native code at runtime (or AOT-compiles it ahead of time). IL enables cross-platform portability and cross-language interop (C#/F#/VB all compile to the same IL and share types).

---

### Q3. What does "managed code" mean?

Code that runs under the runtime's supervision, which provides automatic memory management (GC), type safety, cross-language interop, and portability. Contrast with unmanaged (native C/C++) code where you manage memory manually. C# is managed by default; it can call unmanaged code via P/Invoke.

---

### Q4. What are the four layers of the platform?

1. **Runtime (CLR)** — JIT/AOT, GC, type system. 2. **BCL** — the standard library (`System.*`). 3. **Frameworks** — ASP.NET Core, EF Core, Blazor, etc. 4. **SDK/tooling** — Roslyn, MSBuild, the `dotnet` CLI, NuGet, templates.

---

### Q5. .NET Framework vs .NET Core vs modern .NET?

**.NET Framework** (2002–2019) was Windows-only and closed; final version 4.8, maintenance-only. **.NET Core** (2016–2019) was the cross-platform, open-source rewrite. **Modern .NET** (5+, 2020 onward) unified them into one platform ("just .NET"), one release each November.

---

### Q6. Should you start a new project on .NET Framework 4.8?

No. It's the final Framework version, supported for security but receiving no new features. Start new projects on the latest LTS (.NET 10) and migrate legacy Framework apps when feasible.

---

### Q7. What is the LTS/STS cadence?

A major release every November. **Even versions (6, 8, 10) are LTS** (~3 years of support); **odd versions (7, 9, 11) are STS** (~18 months). Target the latest LTS for production — .NET 10 today (supported to November 2028).

---

### Q8. What was .NET Standard and is it still relevant?

A *specification* of a common API surface that Framework, Core, Mono, and Xamarin all implemented, so one library (`netstandard2.0`) could run everywhere during the split era. Now mostly legacy — target `net10.0` directly for modern-only code, or `netstandard2.0` when a library must still support .NET Framework consumers.

---

### Q9. Name the three runtimes and what each is for.

**CoreCLR** — the default, JIT-based runtime for server/desktop/cloud. **Mono** — portable runtime for mobile (MAUI), Blazor WebAssembly, and games. **Native AOT** — compiles the whole app to a native binary for fast cold start and small footprint (CLI, serverless, containers).

---

### Q10. How does CoreCLR execute code?

It loads IL and JIT-compiles methods on first call, using **tiered compilation** (Tier 0 fast/unoptimized → Tier 1 optimized for hot methods) and **Dynamic PGO** (re-optimizes based on runtime profiles). Memory is managed by a generational, concurrent GC.

---

### Q11. Why does iOS use Mono with AOT?

iOS forbids JIT compilation (no runtime code generation allowed). Mono compiles ahead-of-time on that platform so MAUI apps can run without a JIT.

---

### Q12. What are Native AOT's main trade-offs?

**Gains**: instant startup, low memory, small self-contained binary, no runtime install, predictable latency. **Costs**: no runtime codegen (`Reflection.Emit`, `Expression.Compile`, `dynamic`), limited reflection, mandatory trimming, per-platform builds, and possibly lower steady-state throughput than CoreCLR's JIT+PGO.

---

### Q13. Which runtime wins for a long-running high-throughput server?

CoreCLR — its JIT plus Dynamic PGO keeps re-optimizing hot paths at runtime, often beating AOT's build-time-only optimization once startup is amortized. Native AOT wins for short-lived/serverless workloads where cold start dominates.

---

### Q14. SDK vs runtime — which do you install?

Developers install the **SDK** (it includes the runtime plus compilers, MSBuild, CLI, templates) to *build*. End users / servers install just the **runtime** to *run* (or you bundle it via self-contained publish).

---

### Q15. What are the three runtime families?

**.NET Runtime** (`Microsoft.NETCore.App`) for console/services, **ASP.NET Core Runtime** (`Microsoft.AspNetCore.App`) which adds the web stack, and **.NET Desktop Runtime** (`Microsoft.WindowsDesktop.App`) which adds WPF/WinForms. Check with `dotnet --list-runtimes`.

---

### Q16. What does `global.json` do?

Pins the SDK version for a repo so the whole team and CI build with the same SDK (`{"sdk":{"version":"10.0.100","rollForward":"latestFeature"}}`). Without it, the latest installed SDK is used.

---

### Q17. How do multiple .NET versions coexist?

Side-by-side: SDKs and runtimes install in version-named subfolders under the dotnet directory. Each app resolves the runtime version it targets — no machine-wide single version, avoiding the old Framework "DLL hell."

---

### Q18. What is the Generic Host and why does it matter?

The shared application backbone that wires up dependency injection, configuration, logging, and lifetime — used by web *and* non-web apps. It's why DI/config/logging work the same everywhere in modern .NET (Chapter 03).

---

### Q19. What is .NET Aspire?

A cloud-native orchestration model: you describe a multi-service app (API + worker + database + cache) in C#, and get service discovery, telemetry, a live dashboard, and consistent deployment. It composes the other building blocks (DI, observability, resilience, data) into one runnable system (Chapter 18).

---

### Q20. When would you use Dapper instead of EF Core?

When you want fine-grained SQL control with minimal mapping overhead and maximum query performance, and don't need EF Core's change tracking, migrations, and LINQ-to-SQL translation. Dapper is a thin micro-ORM over ADO.NET; EF Core is a full-featured ORM (Chapters 05–06).

---

### Q21. Why was .NET rewritten as Core?

The original Framework was Windows-only (blocking cloud/Linux/containers), installed machine-wide (version conflicts), closed source, and slow to release. The rewrite delivered cross-platform support, side-by-side/self-contained deployment, open source, big performance gains, and a yearly cadence.

---

### Q22. What is a `dotnet workload`?

An optional add-on to the base SDK for specific targets (MAUI, WebAssembly tooling, Android/iOS). Keeps the base SDK small; install what you need with `dotnet workload install maui`, etc.

---

→ Next: [Coding.md](Coding.md)
