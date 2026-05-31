# Native AOT

## Ahead-of-time compilation to native code

**Native AOT** compiles your .NET app **directly to a native executable** at build time — no IL, no JIT, no .NET runtime needed at execution. Instead of shipping IL that the JIT compiles on startup ([Ch01 §02](../01-Runtime/02-JIT.md)), the entire app (plus the bits of the runtime it needs) is compiled ahead of time into a self-contained native binary. The result is **fast startup, low memory, small footprint, and no runtime dependency** — at the cost of significant **restrictions** on dynamic features. AOT is transformative for the right workloads (CLIs, serverless, high-density microservices) and unsuitable for others (reflection-heavy apps, most UI).

```bash
dotnet publish -c Release -r linux-x64 -p:PublishAot=true
```

```xml
<PropertyGroup>
  <PublishAot>true</PublishAot>
</PropertyGroup>
```

---

## What you gain

| Benefit | Why |
|---|---|
| **Fast startup** | no JIT warmup — code is already native ([Ch01 §02](../01-Runtime/02-JIT.md)) |
| **Low memory** | no JIT, smaller runtime footprint — high density |
| **Small, self-contained binary** | one native file, no runtime install needed |
| **Predictable performance** | no tiered-JIT warmup curve; steady from the first request |
| **No runtime dependency** | runs on minimal images (`runtime-deps` chiseled — [03-ChiseledImages.md](03-ChiseledImages.md)) |

These make AOT compelling for **serverless/functions** (cold-start dominated by startup), **CLI tools** (instant launch), and **microservices** at high density (memory per instance matters). Startup that took hundreds of ms with JIT warmup can drop to single-digit ms.

---

## What you give up (the restrictions)

AOT requires the **whole program** to be analyzable at build time — which rules out features that generate or load code dynamically:

- **No runtime code generation** — `System.Reflection.Emit`, `Expression.Compile()` to IL, dynamic methods don't work (there's no JIT to compile them).
- **Limited reflection** — reflection over types not statically referenced may be trimmed away ([07-Trimming.md](07-Trimming.md)); reflection-based serialization/DI/model-binding needs AOT-safe alternatives (source generators).
- **No dynamic assembly loading** — no `Assembly.LoadFrom`/plugins at runtime.
- **No `dynamic`** keyword, limited runtime type creation.
- **AOT implies trimming** — so it inherits all trim constraints and warnings.
- **Not for most UI** — WPF/WinUI rely on reflection/dynamic features AOT disallows ([Ch16 §07](../16-Desktop/07-Packaging.md)).

The compiler emits **AOT/trim warnings** for incompatible patterns; an AOT-clean app must resolve them (or it may fail at runtime). This is why AOT suits *new, AOT-aware* code more than retrofitting a reflection-heavy legacy app.

---

## Making code AOT-compatible

The ecosystem has shifted to **source generators** that replace runtime reflection with compile-time code — the key to AOT compatibility:

- **System.Text.Json source generation** — `[JsonSerializable]` generates serializers at compile time instead of reflecting over types ([Ch02 §05](../02-BCL/05-Serialization.md)):

```csharp
[JsonSerializable(typeof(Order))]
internal partial class AppJsonContext : JsonSerializerContext { }
// Use AppJsonContext.Default.Order — no runtime reflection, AOT-safe
```

- **Minimal APIs** and the **Request Delegate Generator** make ASP.NET Core endpoints AOT-friendly; **Aspire/Configuration binding source generators** replace reflection-based binding.
- **`[DynamicallyAccessedMembers]` / `[DynamicDependency]`** annotate the reflection you *must* keep so the trimmer preserves it.

ASP.NET Core has invested heavily in AOT support (Minimal APIs, gRPC, Worker services are AOT-friendly); MVC and some reflection-heavy stacks are less so. Check feature AOT-compatibility before committing.

---

## When to use Native AOT

- **Strong fit**: CLI tools (instant startup), serverless/Azure Functions (cold-start sensitive), high-density microservices (memory-per-instance), Minimal API/gRPC services written AOT-aware, containers wanting the smallest image.
- **Poor fit**: reflection/dynamic-heavy apps (heavy reflection, `Reflection.Emit`, plugins), most desktop UI (WPF/WinUI), apps depending on libraries that aren't AOT-compatible, or large legacy codebases where resolving every trim/AOT warning isn't worth it.

For apps that *want* faster startup but *can't* meet AOT's constraints, **ReadyToRun** ([08-ReadyToRun.md](08-ReadyToRun.md)) is the no-compromise middle ground.

---

## Common gotchas

### Reflection/serialization breaking at runtime

Reflection-based code (JSON, DI, binding) may be trimmed/unsupported under AOT, failing at runtime. Use **source generators** (System.Text.Json source gen, etc.) and heed AOT/trim warnings — don't ignore them.

### Assuming all libraries are AOT-compatible

A dependency using `Reflection.Emit`/dynamic loading breaks AOT. Verify your libraries declare AOT compatibility before committing to AOT.

### Trying to AOT a UI app

WPF/WinUI aren't AOT-supported. Use ReadyToRun for startup gains there ([Ch16 §07](../16-Desktop/07-Packaging.md)).

### Ignoring build warnings

AOT/trim warnings predict runtime failures. Treat them as errors for AOT builds — an AOT-clean app resolves them all.

### Longer build times / platform-specific output

AOT compilation is slower and produces a **platform-specific** native binary (per RID) — build per target platform, and expect longer CI build steps.

---

## Summary

- **Native AOT** compiles the app to a **native, self-contained binary** at build time — **fast startup, low memory, small footprint, no runtime dependency** (no JIT, no IL) — ideal for **CLIs, serverless, and high-density microservices**.
- It imposes heavy **restrictions**: **no runtime code generation** (`Reflection.Emit`/`Expression.Compile`), **limited reflection**, no dynamic assembly loading, no `dynamic`, **implies trimming**, and is **unsupported for most UI** (WPF/WinUI).
- Achieve AOT compatibility with **source generators** (System.Text.Json source gen, Minimal API RDG, config binding gen) replacing runtime reflection, plus `[DynamicallyAccessedMembers]`/`[DynamicDependency]` for reflection you must keep — and **resolve all AOT/trim warnings**.
- Use AOT for **new, AOT-aware** console/server workloads; for apps needing reflection/JIT/UI but wanting faster startup, use **ReadyToRun** ([08-ReadyToRun.md](08-ReadyToRun.md)) instead. Pair AOT with `runtime-deps` **chiseled** images for the smallest container ([03-ChiseledImages.md](03-ChiseledImages.md)).

→ Next: [07-Trimming.md](07-Trimming.md)
