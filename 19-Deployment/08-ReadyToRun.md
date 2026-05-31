# ReadyToRun

## Faster startup without AOT's restrictions

**ReadyToRun (R2R)** precompiles your app's IL to **native code ahead of time** and ships it **alongside the IL** — so startup does far less JIT work, while the runtime keeps full JIT/reflection/dynamic capabilities. It's the **middle ground** between pure IL (small, but JIT-warms-up at startup — [Ch01 §02](../01-Runtime/02-JIT.md)) and Native AOT (fastest startup, but heavy restrictions — [06-NativeAOT.md](06-NativeAOT.md)). R2R buys **faster startup with no compatibility cost**, making it the safe choice when you want quicker boot but can't (or won't) meet AOT's constraints — including reflection-heavy apps and UI frameworks ([Ch16 §07](../16-Desktop/07-Packaging.md)).

```bash
dotnet publish -c Release -r linux-x64 -p:PublishReadyToRun=true
```

```xml
<PropertyGroup>
  <PublishReadyToRun>true</PublishReadyToRun>
</PropertyGroup>
```

---

## How R2R works

Normally, the JIT compiles IL methods to native code **on first call** at runtime — fast steady-state, but a startup cost (warmup) as hot methods get compiled and re-optimized ([Ch01 §02](../01-Runtime/02-JIT.md)). R2R does much of that compilation **at publish time**, embedding native code into the assemblies. At runtime:

- Methods that have R2R native code **run immediately** — no JIT needed on startup, so the app reaches working speed faster.
- The **IL is still present**, and the JIT can still **re-optimize hot methods** at runtime (tiered compilation upgrades R2R "tier-0-ish" code to fully optimized "tier-1" — [Ch01 §02](../01-Runtime/02-JIT.md)) and JIT anything not precompiled.

So R2R is **not** "no JIT" — it's "less JIT at startup." You keep the JIT (and thus reflection, dynamic code, runtime optimization), you just pay less startup tax.

---

## R2R vs AOT vs IL — the spectrum

| | IL only (default) | **ReadyToRun** | Native AOT |
|---|---|---|---|
| Startup | slow (full JIT warmup) | **fast** (precompiled) | fastest (no JIT) |
| Steady-state perf | best (full JIT/PGO) | best (JIT re-optimizes) | good (no runtime re-opt) |
| Size | smallest | **larger** (IL + native) | small (native only) |
| Runtime needed | yes (FDD) | yes (FDD) | no (self-contained) |
| Reflection/dynamic | ✅ full | ✅ **full** | ❌ restricted |
| UI frameworks | ✅ | ✅ | ❌ |
| Compatibility cost | none | **none** | high |

The key column is the last few rows: **R2R keeps full compatibility** (reflection, dynamic, UI, all libraries) — unlike AOT. That's its whole selling point: a startup win you can apply to *almost any* app without code changes.

---

## When to use R2R

- **Apps that need faster startup but can't go AOT** — reflection-heavy services, MVC apps, anything using libraries that aren't AOT-compatible.
- **Desktop UI** (WPF/WinUI) — AOT isn't supported, but R2R speeds launch ([Ch16 §07](../16-Desktop/07-Packaging.md)).
- **Containers/serverless** where cold start matters but AOT's constraints are too limiting.
- **Large existing apps** where resolving every trim/AOT warning isn't worth it — R2R needs no such work.

R2R is the **low-risk** startup optimization: turn it on, get faster boot, keep everything working. The cost is a **larger artifact** (it carries both native code and IL) and slightly longer publish — usually an easy trade.

---

## Combining with other options

R2R composes with publish modes ([01-PublishModes.md](01-PublishModes.md)):

```bash
# Self-contained + R2R for a no-runtime-dependency, fast-startup app:
dotnet publish -c Release -r linux-x64 --self-contained -p:PublishReadyToRun=true
```

It's commonly used with **framework-dependent** (container base image with the runtime) or **self-contained** publishes. Note R2R requires a **RID** (it produces platform-specific native code, like AOT/self-contained). You typically *don't* combine R2R with AOT (AOT already compiles natively); R2R is the alternative to AOT, not an addition.

---

## Common gotchas

### Expecting AOT-level startup/size

R2R speeds startup but isn't as fast as AOT (the JIT still runs for non-precompiled/hot methods), and the artifact is **larger** (IL + native), not smaller. For the absolute smallest/fastest, AOT — if you can meet its constraints.

### Forgetting the RID

R2R produces platform-specific native code, so it needs a **RID** (`-r linux-x64`). Build per target platform.

### Thinking R2R removes the JIT

The JIT is still present and **re-optimizes hot paths** at runtime (tiered compilation) and compiles anything not precompiled. R2R reduces startup JIT, it doesn't eliminate JIT.

### Using R2R when AOT would be better (or vice versa)

If your app meets AOT's constraints and wants the smallest/fastest result, AOT is better. If it needs reflection/dynamic/UI or broad library compatibility, R2R is the right choice. Pick per app.

### Ignoring the size increase

R2R artifacts are larger. In size-sensitive scenarios (tiny containers), weigh the startup gain against the size; AOT or plain IL may suit better.

---

## Summary

- **ReadyToRun (R2R)** precompiles IL to native code **at publish time, shipped alongside the IL** — so startup does **less JIT work** (faster boot) while the runtime **keeps full JIT/reflection/dynamic** capability and can **re-optimize hot methods** at runtime.
- It's the **middle ground** between IL-only (small, slow startup) and **Native AOT** (fastest/smallest, but restricted) — its defining advantage is **no compatibility cost**: works with reflection-heavy apps, UI frameworks, and any libraries, with **no code changes**.
- Use R2R when you want **faster startup but can't/won't go AOT** (MVC/reflection-heavy services, WPF/WinUI, large legacy apps, cold-start-sensitive containers); the trade-off is a **larger artifact** and a required **RID**.
- It **doesn't remove the JIT** (still re-optimizes hot paths) and isn't as fast/small as AOT — choose AOT when its constraints are acceptable, R2R otherwise.

→ Next: [09-CICD.md](09-CICD.md)
