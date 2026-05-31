# Publish Modes

## How your app gets packaged

`dotnet publish` produces the deployable output of your app — but *how* it's packaged has several dimensions that trade off **size, startup time, runtime requirements, and compatibility**. The core decisions: **framework-dependent vs self-contained** (does the target have .NET?), **single-file or not** (one exe vs a folder), **trimmed or not** (remove unused code?), and **AOT or not** (compile to native?). Choosing well depends on your deployment target (a container with a known runtime vs an arbitrary machine), startup/size constraints, and whether your code is trim/AOT-compatible.

```bash
dotnet publish -c Release                                   # framework-dependent (default)
dotnet publish -c Release -r linux-x64 --self-contained     # bundles the runtime
dotnet publish -c Release -r linux-x64 -p:PublishSingleFile=true
dotnet publish -c Release -r linux-x64 -p:PublishTrimmed=true
dotnet publish -c Release -r linux-x64 -p:PublishAot=true    # native AOT
```

---

## Framework-dependent vs self-contained

The first decision: does the **.NET runtime** exist on the target?

| | Framework-dependent (FDD) | Self-contained (SCD) |
|---|---|---|
| Runtime | **required** on the target | **bundled** in the output |
| Size | small (just your app) | large (app + runtime, ~70MB+) |
| Updates | runtime patched by the host/OS | you ship runtime updates |
| Portability | needs matching .NET installed | runs anywhere for that RID |
| RID needed | no (portable) | yes (`-r linux-x64`, etc.) |

- **Framework-dependent** is smaller and lets the host patch the runtime independently — ideal when you **control the environment** (a base image with the .NET runtime, a server with .NET installed).
- **Self-contained** bundles the runtime so the app runs on a machine **without .NET** (or pins an exact runtime version) — at the cost of size, and you're responsible for shipping runtime security updates.

For containers, FDD on a .NET runtime base image is common (small layers, runtime managed by the base image). For "xcopy to any machine," SCD avoids the "install .NET first" problem.

---

## Runtime Identifier (RID)

Self-contained, single-file, trimming, and AOT all require a **RID** — the target platform (`linux-x64`, `win-x64`, `linux-arm64`, `osx-arm64`, …). It tells the publisher which runtime to bundle and which native bits to produce. Framework-dependent **portable** builds don't need a RID (the installed runtime handles the platform). Match the RID to where the app runs (e.g., `linux-arm64` for ARM containers/cloud).

---

## Single-file publish

`PublishSingleFile=true` packs the app (and, if self-contained, the runtime) into **one executable** — easier to distribute and copy:

```bash
dotnet publish -c Release -r linux-x64 --self-contained -p:PublishSingleFile=true
```

It's a packaging convenience (at runtime, parts may be extracted/loaded from the bundle). It pairs naturally with self-contained for "one file you can hand someone." Note some native libraries may still sit alongside, and certain assembly-loading patterns behave differently — test the single-file output.

---

## Trimming

`PublishTrimmed=true` runs the **IL trimmer**, which removes assemblies/types/members your app doesn't use, shrinking self-contained output significantly ([07-Trimming.md](07-Trimming.md)):

```bash
dotnet publish -c Release -r linux-x64 --self-contained -p:PublishTrimmed=true
```

The catch: the trimmer analyzes static references, so code reached only via **reflection** (serialization, DI by name, plugins) can be trimmed away, causing runtime `MissingMethod`/type-load failures. The compiler emits **trim warnings** for unsafe patterns; you annotate (`[DynamicallyAccessedMembers]`, `[DynamicDependency]`) or avoid reflection to make code "trim-safe." Trimming is most valuable (and safest) for **AOT** and minimal-container scenarios where size matters and the code is trim-compatible.

---

## Native AOT

`PublishAot=true` compiles your app **ahead-of-time to native code** ([06-NativeAOT.md](06-NativeAOT.md)) — no JIT, no IL at runtime:

```bash
dotnet publish -c Release -r linux-x64 -p:PublishAot=true   # implies self-contained + trimming
```

- **Pros**: **fast startup** (no JIT warmup — [Ch01 §02](../01-Runtime/02-JIT.md)), **low memory**, small self-contained native binary, no runtime needed — great for CLI tools, serverless/functions, and high-density microservices.
- **Cons**: heavy **restrictions** — no runtime code generation (`Reflection.Emit`, dynamic), limited reflection (requires trim/AOT-safe code), no plugin loading, longer build, and **not supported for many UI frameworks** (WPF/WinUI — [Ch16 §07](../16-Desktop/07-Packaging.md)). AOT implies trimming, so it inherits trim constraints.

AOT is the most aggressive mode — best for compatible console/server workloads where startup/memory/size matter; not a drop-in for reflection-heavy apps.

---

## ReadyToRun (the middle ground)

`PublishReadyToRun=true` precompiles IL to native **alongside** the IL ([08-ReadyToRun.md](08-ReadyToRun.md)) — improving **startup** (less JIT work) **without** AOT's restrictions:

```bash
dotnet publish -c Release -r linux-x64 -p:PublishReadyToRun=true
```

R2R keeps full runtime/reflection/JIT capabilities (the JIT can still re-optimize hot paths), so it's a **safe startup optimization** for apps that can't go AOT (including UI apps). Larger output than IL-only, but no compatibility cost. It's the pragmatic choice when you want faster startup without rewriting for AOT.

---

## Choosing a mode

```
Target has .NET runtime (controlled env / base image)?  → framework-dependent
Target may lack .NET / want exact version?              → self-contained (+ single-file)
Need smallest size & fastest startup, code is AOT-safe? → Native AOT (console/server)
Want faster startup but need reflection/JIT/UI?         → ReadyToRun
Container: small image + managed runtime?               → FDD on a runtime base image ([02-Docker.md])
```

These compose: e.g., self-contained + trimmed + single-file, or AOT (which implies self-contained + trimmed). Pick based on environment, constraints, and compatibility.

---

## Common gotchas

### Self-contained when the runtime is available

Shipping the whole runtime to a target that already has .NET bloats size needlessly. Use framework-dependent in controlled environments (base images, managed hosts).

### Trimming/AOT breaking reflection

The trimmer/AOT removes code not statically referenced — reflection-based serialization/DI/plugins can fail at runtime. Heed **trim warnings**, annotate (`[DynamicallyAccessedMembers]`/`[DynamicDependency]`), or use source-generated alternatives (e.g., System.Text.Json source gen). Always test trimmed/AOT output.

### Forgetting the RID

Self-contained/single-file/trim/AOT need a RID (`-r linux-x64`). Omitting it (when required) fails or produces the wrong target. Match the RID to the deployment platform.

### Expecting AOT for UI/reflection-heavy apps

Native AOT generally **doesn't support** WPF/WinUI and reflection-heavy frameworks. Use **ReadyToRun** for startup gains there instead ([Ch16 §07](../16-Desktop/07-Packaging.md)).

### Not testing the published artifact

Behavior can differ between a `dotnet run` debug session and a trimmed/single-file/AOT Release artifact. Test the **actual published output** in a representative environment.

---

## Summary

- `dotnet publish` packaging trades off **size, startup, runtime requirement, and compatibility** across four axes: **framework-dependent vs self-contained**, **single-file**, **trimming**, and **AOT**.
- **Framework-dependent** (small, needs .NET installed — controlled envs/base images) vs **self-contained** (bundles the runtime, runs anywhere for a **RID**, larger, you ship runtime updates); **single-file** packs it into one executable.
- **Trimming** shrinks self-contained output but can remove **reflection-reached** code (heed trim warnings; annotate); **Native AOT** compiles to native for **fast startup/low memory/no runtime** but imposes heavy **restrictions** (no dynamic codegen, limited reflection, no UI frameworks) and implies trimming.
- **ReadyToRun** is the safe middle ground — precompiled native **alongside** IL for **faster startup without AOT's restrictions** (keeps reflection/JIT/UI support).
- Choose by environment/constraints/compatibility; modes compose; always **test the published artifact**.

→ Next: [02-Docker.md](02-Docker.md)
