# The JIT Compiler

## What it is

The JIT (Just-In-Time compiler, codenamed **RyuJIT**) translates your method's IL into native machine code **the first time that method is called**. Subsequent calls run the already-compiled native code. This "compile on demand" model is why .NET starts without a separate native-build step yet runs at near-native speed once warmed up.

```
First call to Foo():   IL  ──RyuJIT──▶  native code  ──▶  execute
Later calls to Foo():                    (cached native) ──▶  execute
```

The JIT is the heart of CoreCLR's performance story. Modern .NET's JIT does far more than naive translation — tiered compilation, profile-guided optimization, and on-stack replacement let it match or beat ahead-of-time-compiled languages on hot code.

---

## Why JIT instead of AOT-always

JIT trades startup time for two big wins:
1. **Portability** — IL is CPU-agnostic; the JIT targets the *actual* machine (using its specific instruction sets: AVX2, AVX-512, ARM NEON).
2. **Runtime knowledge** — the JIT sees real execution: which branches are hot, which types actually flow through a virtual call. It optimizes based on *observed* behavior, which a static compiler can't know.

The cost is **warmup**: early calls run less-optimized code while the JIT compiles. Tiered compilation (below) minimizes this. Native AOT ([03-NativeAOT.md](03-NativeAOT.md)) chooses the opposite trade-off: no warmup, but no runtime adaptation.

---

## Tiered compilation

Compiling every method with full optimization upfront would make startup slow. Compiling nothing optimized would make hot code slow. **Tiered compilation** gets both:

```
Method first called
   ↓
Tier 0  — compiled FAST, minimal optimization (quick startup)
   ↓  (method called many times → it's "hot")
Tier 1  — recompiled with FULL optimization (peak throughput)
```

- **Tier 0** ("quick JIT"): compiles in a hurry, little optimization. Gets the app running fast. Includes lightweight call counting so the runtime knows when a method gets hot.
- **Tier 1** ("optimizing JIT"): once a method's call count crosses a threshold, it's recompiled with all optimizations (inlining, loop opts, vectorization). The new native code replaces the old.

This is on by default. Result: fast startup *and* fast steady state — the method-level equivalent of "warm up the hot paths, don't waste effort on cold ones."

```xml
<!-- runtimeconfig knobs (rarely needed) -->
<PropertyGroup>
  <TieredCompilation>true</TieredCompilation>          <!-- default -->
  <TieredPGO>true</TieredPGO>                            <!-- default in .NET 8+ -->
</PropertyGroup>
```

---

## On-Stack Replacement (OSR)

Tiered compilation has a gap: what about a method with a **long-running loop** that's called only once (e.g., `Main`'s main loop)? It would be stuck in slow Tier 0 forever because it never gets *re-called* to trigger Tier 1.

**OSR** solves this: it can swap a method's Tier 0 code for optimized Tier 1 code **while it's still executing in the loop** — replacing the running frame on the stack. So even a one-shot method with a hot loop gets optimized mid-flight. (Introduced in .NET 7, refined since.)

```csharp
static void Main() {
    for (long i = 0; i < 10_000_000_000; i++) {   // OSR kicks in here mid-loop
        Compute(i);                                  // → swaps to optimized code without restarting
    }
}
```

---

## Dynamic PGO (Profile-Guided Optimization)

The JIT's superpower in modern .NET. **Dynamic PGO** instruments Tier 0 code to record *runtime behavior* — which branches are taken, which concrete types flow through virtual/interface calls — then uses that profile when compiling Tier 1.

Key optimizations it enables:
- **Devirtualization + guarded devirtualization** — if a virtual/interface call almost always targets one concrete type, the JIT inlines that type's method behind a cheap type check, falling back to the virtual call otherwise. Huge for LINQ, interfaces, and polymorphism.
- **Hot/cold block splitting** — frequently-executed blocks are laid out together; rarely-hit blocks (e.g., exception throws) are moved out of the hot path for better instruction-cache behavior.
- **Better inlining** decisions based on actual call frequency.

Dynamic PGO is **on by default since .NET 8**, and it's why modern .NET hot paths often rival C++. There's also **static PGO** (profiles captured at build time), which complements it.

---

## What the JIT optimizes

A non-exhaustive list of what RyuJIT does at Tier 1:
- **Inlining** — small/hot methods are inlined to remove call overhead (and enable further optimization). `[MethodImpl(MethodImplOptions.AggressiveInlining)]` hints it; `NoInlining` prevents it.
- **Devirtualization** — turn virtual/interface calls into direct calls when the type is known (sealed types, or via PGO).
- **Bounds-check elimination** — remove array bounds checks the JIT can prove are safe (e.g., `for (i=0; i<arr.Length; i++)`).
- **Loop optimizations** — unrolling, hoisting invariants, cloning.
- **Vectorization** — using SIMD (`Vector128/256/512`) for suitable loops, and recognizing hardware intrinsics.
- **Constant folding / propagation**, dead-code elimination, common-subexpression elimination.
- **Escape analysis** (improving in .NET 9/10) — stack-allocate objects that don't escape the method, eliminating heap allocations.

You write idiomatic C#; the JIT does this for you. The performance idioms in CSharpBook Chapter 17 are largely about *helping* the JIT (e.g., writing loops it can bounds-check-eliminate and vectorize).

---

## ReadyToRun (R2R) — pre-JIT at publish

You can pre-compile assemblies to native code at **publish** time with ReadyToRun:

```bash
dotnet publish -c Release -r linux-x64 -p:PublishReadyToRun=true
```

R2R embeds native code alongside the IL. At startup, methods run from the precompiled native code (no Tier 0 JIT cost) — **faster startup**. But:
- R2R code is conservatively optimized (it can't assume the exact CPU). The JIT can still **re-optimize hot methods to Tier 1** at runtime, and Dynamic PGO still applies.
- Larger binary (native + IL).

R2R is the middle ground: faster cold start than pure JIT, while keeping the JIT's runtime adaptation. Good for long-running services where you want both. See [Chapter 19](../19-Deployment/README.md).

---

## What the JIT can't do (and AOT can't either)

- **Re-JIT arbitrary code at will** — tiering and OSR are controlled; you can't force recompilation from user code (except via diagnostics).
- **Optimize across a P/Invoke boundary** — native code is opaque.
- **Devirtualize truly polymorphic calls** — if many types flow through a call site, PGO can't pick one; it stays virtual.
- **Beat algorithmic problems** — the JIT optimizes your code as written; it won't fix an O(n²) algorithm.

---

## Observing the JIT

```bash
# Show the asm the JIT produced for methods (set before running)
export DOTNET_JitDisasm="MyNamespace.MyClass:Compute"
dotnet run -c Release

# Tiering / OSR diagnostics
export DOTNET_TieredCompilation=0     # disable tiering (always Tier 1) — for benchmarking
export DOTNET_TC_QuickJitForLoops=0
```

`DOTNET_JitDisasm` prints the actual native code for named methods — invaluable for verifying inlining/vectorization. BenchmarkDotNet's `[DisassemblyDiagnoser]` does this cleanly (CSharpBook Ch16).

---

## Common gotchas

### Benchmarking cold (Tier 0) code

Measuring a method's first few calls measures Tier 0 + JIT time, not steady-state. Use BenchmarkDotNet (it warms up to Tier 1) — never a naive `Stopwatch` on first call.

### Assuming `[AggressiveInlining]` everywhere helps

Over-inlining bloats code and hurts the instruction cache. The JIT inlines well by default; add the attribute only for measured tiny hot methods.

### Expecting AOT-level startup from JIT

JIT apps pay warmup. If cold start dominates (serverless, CLI), use R2R or Native AOT.

### Disabling tiering in production

`DOTNET_TieredCompilation=0` forces Tier 1 everywhere — slower startup, no benefit for most apps. Only use it for benchmarking isolation.

---

## Summary

- The **JIT (RyuJIT)** compiles IL → native code per method on first call, targeting the actual CPU.
- **Tiered compilation**: Tier 0 (fast compile, quick startup) → Tier 1 (full optimization for hot methods). **OSR** upgrades long-running loops mid-execution.
- **Dynamic PGO** (default in .NET 8+) profiles runtime behavior to drive **devirtualization**, hot/cold splitting, and inlining — often matching C++ on hot paths.
- The JIT does inlining, bounds-check elimination, vectorization, escape analysis, and more — write idiomatic code and help it.
- **ReadyToRun** pre-JITs at publish for faster startup while keeping runtime re-optimization.
- Benchmark at steady state (BenchmarkDotNet), not on cold Tier 0 code.

→ Next: [03-NativeAOT.md](03-NativeAOT.md)
