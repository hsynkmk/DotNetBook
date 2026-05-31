# What Is .NET?

## The platform, not the language

People say ".NET" to mean several different things. Precisely, **.NET is a development platform** — a set of runtimes, libraries, tools, and frameworks for building and running applications. **C# is a language** that targets that platform (so do F# and Visual Basic). This book is about the platform; `CSharpBook/` is about the language.

```
┌─────────────────────────────────────────────────────────┐
│  Languages:   C#        F#        VB                      │  ← CSharpBook covers C#
├─────────────────────────────────────────────────────────┤
│  Compilers:   Roslyn (C#/VB)      F# compiler             │
│               ↓ produces IL (Intermediate Language)        │
├─────────────────────────────────────────────────────────┤
│  Runtime:     CoreCLR / Mono / NativeAOT                  │  ← executes IL / native code
│               JIT, GC, type system, threading             │
├─────────────────────────────────────────────────────────┤
│  BCL:         System.*, collections, IO, networking...    │  ← the standard library
├─────────────────────────────────────────────────────────┤
│  Frameworks:  ASP.NET Core, EF Core, Blazor, MAUI...      │  ← app-building libraries
├─────────────────────────────────────────────────────────┤
│  SDK / Tooling: dotnet CLI, MSBuild, templates, NuGet     │  ← build & manage
└─────────────────────────────────────────────────────────┘
```

When you write C#, the compiler produces **IL** (a CPU-independent bytecode). The runtime loads that IL, JIT-compiles it to native machine code (or, with Native AOT, compiles ahead of time), and executes it with services like garbage collection and a unified type system. Everything above the runtime — collections, file I/O, HTTP, JSON — comes from the **Base Class Library (BCL)**.

---

## The four layers you'll actually touch

### 1. The runtime (CLR)

The **Common Language Runtime** executes your code. It provides:
- **JIT compilation** — IL → native code at runtime (or AOT ahead of time).
- **Garbage collection** — automatic memory management.
- **Type system** — a unified type model shared across languages (the CTS).
- **Threading, exceptions, security, interop** — the execution services.

Modern .NET ships three runtime flavors (CoreCLR, Mono, Native AOT) — see [03-Runtimes.md](03-Runtimes.md). Deep internals are in [Chapter 01](../01-Runtime/README.md).

### 2. The Base Class Library (BCL)

The standard library: `System.Collections`, `System.IO`, `System.Net.Http`, `System.Text.Json`, `System.Threading`, `System.Linq`, and thousands more types. It's the same across runtimes and platforms. Covered in [Chapter 02](../02-BCL/README.md).

### 3. The frameworks

Higher-level libraries for building specific kinds of apps:
- **ASP.NET Core** — web APIs and apps ([Ch04](../04-AspNetCore/README.md))
- **Entity Framework Core** — database access ([Ch05](../05-EFCore/README.md))
- **Blazor** — web UI in C# ([Ch14](../14-Blazor/README.md))
- **MAUI** — cross-platform mobile/desktop ([Ch15](../15-MAUI/README.md))
- ...and the rest of this book.

### 4. The SDK and tooling

The **SDK** is what you install to build apps: the compilers (Roslyn), MSBuild, the `dotnet` CLI, project templates, and NuGet. End users who only *run* apps need just the **runtime**, not the SDK. See [04-WhatsInTheBox.md](04-WhatsInTheBox.md).

---

## "Managed" code and the CLR's promise

.NET code is **managed** — it runs under the supervision of the runtime, which provides:
- **Automatic memory management** (GC) — you rarely free memory manually.
- **Type safety** — the runtime verifies types; no arbitrary pointer arithmetic (outside `unsafe`).
- **Cross-language interop** — C#, F#, VB compile to the same IL and share types.
- **Portability** — the same IL runs on Windows, Linux, macOS (and via Mono, mobile/WASM).

This contrasts with **unmanaged** (native C/C++) code where you manage memory and there's no runtime. .NET can call into unmanaged code via P/Invoke (CSharpBook Chapter 14), but your C# is managed by default.

---

## One .NET, many workloads

Since .NET 5 (2020), there's a **single, unified .NET** that targets everything:

| Workload | What you build |
|---|---|
| Web | APIs, MVC, Razor Pages, Blazor (ASP.NET Core) |
| Desktop | WPF, WinUI, WinForms (Windows); MAUI (cross-platform) |
| Mobile | iOS, Android (MAUI) |
| Cloud / microservices | Containers, serverless, Aspire-orchestrated services |
| Console / tools | CLI apps, daemons, workers |
| Games | Unity (uses a .NET runtime), MonoGame |
| AI/ML | ML.NET, ONNX, Semantic Kernel |
| IoT / embedded | .NET IoT, Native AOT for small footprints |

The same language, runtime, and BCL underpin all of these. That unification — after years of fragmentation — is the headline story of modern .NET (see [02-History.md](02-History.md)).

---

## Open source and cross-platform

Modern .NET is:
- **Open source** — the runtime, libraries, and compilers live on GitHub (`dotnet/runtime`, `dotnet/roslyn`, `dotnet/aspnetcore`), under the .NET Foundation, MIT-licensed.
- **Cross-platform** — first-class on Windows, Linux, and macOS; ARM and x64; containers and cloud.
- **Backed by Microsoft** but developed in the open with broad community contribution.

This is a sharp break from the old .NET Framework, which was Windows-only and closed.

---

## What .NET is *not*

- **Not just Windows** — that was .NET Framework. Modern .NET is cross-platform.
- **Not a language** — that's C#/F#/VB.
- **Not an IDE** — Visual Studio, VS Code, and Rider are tools *for* .NET, not .NET itself.
- **Not Mono-only or CoreCLR-only** — modern .NET unifies multiple runtimes under one platform.

---

## Summary

- **.NET is a platform**: runtime (CLR) + standard library (BCL) + frameworks + SDK/tooling. **C# is a language** that targets it.
- Your C# compiles to **IL**, which the runtime JITs (or AOT-compiles) to native code and runs as **managed** code with GC, type safety, and interop.
- Since .NET 5 there's **one unified .NET** for web, desktop, mobile, cloud, and more.
- It's **open source and cross-platform**, developed in the open under the .NET Foundation.
- This book covers the platform layer by layer; the next files trace its history, runtimes, SDK contents, and ecosystem.

→ Next: [02-History.md](02-History.md)
