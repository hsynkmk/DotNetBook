# The Runtimes — CoreCLR, Mono, Native AOT

## Three runtimes under one platform

"Modern .NET" isn't a single runtime — it's a platform with **multiple runtime implementations**, each tuned for different workloads. They share the same BCL and tooling, so your code is mostly portable across them, but they execute it differently.

| Runtime | Used for | Execution model |
|---|---|---|
| **CoreCLR** | Server, desktop, cloud (the default) | JIT (with tiered compilation + PGO) |
| **Mono** | Mobile (iOS/Android), WebAssembly, games | JIT, interpreter, or AOT depending on platform |
| **Native AOT** | CLI tools, serverless, containers, constrained | Fully ahead-of-time compiled to native |

When you run `dotnet run` on a server or desktop, you're on **CoreCLR**. When you build a MAUI mobile app or a Blazor WebAssembly app, you're on **Mono**. When you publish with `PublishAot=true`, you get **Native AOT**.

---

## CoreCLR — the default workhorse

CoreCLR is the high-performance runtime for server, cloud, and desktop. It's what you use unless you've specifically chosen otherwise.

How it runs your code:
1. Loads your assembly's **IL**.
2. **JIT-compiles** methods to native code as they're first called.
3. Uses **tiered compilation**: methods start at Tier 0 (fast to compile, less optimized), and hot methods are recompiled at Tier 1 (highly optimized) once they prove worth it.
4. Applies **Dynamic PGO** (Profile-Guided Optimization, default in .NET 8+): it observes which branches/types are hot at runtime and re-optimizes accordingly — often matching or beating C++ on hot paths.
5. Manages memory with a **generational, concurrent garbage collector** (with DATAS adaptive sizing in .NET 9+).

**Strengths**: peak steady-state throughput (the JIT keeps optimizing), mature GC, full feature set. **Trade-off**: JIT warmup means slower cold start than AOT, and it carries the runtime. Deep internals: [Chapter 01](../01-Runtime/README.md).

---

## Mono — mobile, WASM, and games

Mono is the older, highly portable runtime (originally a cross-platform .NET implementation, now part of the unified platform). It targets environments CoreCLR doesn't:

- **Mobile** (iOS, Android) via MAUI — iOS forbids JIT, so Mono uses **AOT** there.
- **WebAssembly** (Blazor WebAssembly) — runs .NET in the browser; uses an **interpreter** plus optional AOT for hot code.
- **Games** — Unity uses a Mono-derived runtime.

Mono is smaller and more portable than CoreCLR but historically slower for raw compute. It's the right (and often only) choice for the platforms above. You don't usually *choose* Mono explicitly — the workload (MAUI, Blazor WASM) selects it.

---

## Native AOT — ahead-of-time native binaries

Native AOT compiles your **entire app to a single native executable** at publish time. No IL, no JIT, no runtime install — like a C++ program with a GC linked in.

```bash
dotnet publish -r linux-x64 -c Release   # with <PublishAot>true</PublishAot>
```

**Strengths**:
- **Instant startup** (~milliseconds — no JIT warmup). Ideal for serverless/Lambda cold starts and CLI tools.
- **Low memory**, **small self-contained binary**, **no runtime dependency**.
- **Predictable latency** (no JIT/tiering jitter).

**Limitations**:
- **No runtime code generation** — `Reflection.Emit`, `Expression.Compile`, `dynamic`, runtime assembly loading don't work.
- **Limited reflection** + **mandatory trimming** — code must be AOT-compatible (use source generators instead of runtime reflection).
- **Per-platform builds** (one per RID).
- May lose **steady-state throughput** to CoreCLR's JIT+PGO on long-running servers.

Full treatment in CSharpBook Chapter 14 and DotNetBook [Chapter 19](../19-Deployment/README.md).

---

## Choosing a runtime

You rarely choose explicitly — the **workload picks it**:

```
Building a web API / service / desktop app?     → CoreCLR (default)
Building a mobile app or Blazor WebAssembly?     → Mono (chosen by MAUI / WASM)
Building a CLI tool, serverless fn, tiny container, want fast cold start?
                                                 → Native AOT (opt in)
Long-running high-throughput server?             → CoreCLR (JIT + PGO wins steady state)
```

| Concern | CoreCLR | Mono | Native AOT |
|---|---|---|---|
| Startup speed | medium | varies | **fastest** |
| Steady-state throughput | **best** | lower | good |
| Memory footprint | medium | low–medium | **lowest** |
| Reflection / dynamic | **full** | full | **limited** |
| Runtime install needed | yes (or self-contained) | bundled | **none** |
| Target platforms | server/desktop/cloud | mobile/WASM/games | per-RID native |

---

## They share the same code

The crucial point: **your C#, the BCL, and most NuGet packages are the same** across all three. A `List<T>`, an `HttpClient`, a LINQ query behave identically. The differences are in *how* the code is executed and which advanced features (reflection emit, JIT) are available. This is what "one unified .NET" means in practice — write once, run on the runtime your workload needs.

The main portability caveat is **Native AOT's restrictions** (no runtime codegen, trimming). Code that uses reflection-heavy libraries may run fine on CoreCLR/Mono but break under AOT — which is why the modern ecosystem increasingly uses **source generators** (compile-time codegen) instead of runtime reflection.

---

## Summary

- Modern .NET has **three runtimes** sharing one BCL and toolchain: **CoreCLR** (default, JIT + tiered + PGO, server/desktop/cloud), **Mono** (mobile, Blazor WASM, games), **Native AOT** (single native binary, fast cold start, constrained).
- **CoreCLR** wins steady-state throughput via JIT; **Native AOT** wins startup/size but forbids runtime codegen and requires trimming; **Mono** is the portable choice for mobile/WASM.
- You usually don't pick a runtime — the **workload selects it**; you only opt into AOT explicitly.
- Your code is largely **portable across runtimes**; the chief exception is Native AOT's reflection/JIT restrictions, mitigated by source generators.

→ Next: [04-WhatsInTheBox.md](04-WhatsInTheBox.md)
