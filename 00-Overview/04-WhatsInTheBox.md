# What's in the Box — SDK, Runtimes, Tooling

## SDK vs Runtime — install the right thing

The single most common beginner confusion:

- **Runtime** — what you need to **run** a .NET app. End users / production servers install this (or you bundle it via self-contained publish).
- **SDK** — what you need to **build** a .NET app. It *includes* the runtime, plus the compilers, build engine, CLI, and templates. Developers install this.

```
.NET SDK  ⊇  .NET Runtime(s)
  ├── Runtime(s): CoreCLR + BCL (run apps)
  ├── ASP.NET Core Runtime (run web apps)
  ├── Roslyn compilers (C#, VB)
  ├── MSBuild (build engine)
  ├── dotnet CLI (the driver)
  ├── NuGet (package manager)
  └── Project templates
```

```bash
dotnet --list-sdks        # SDKs installed (for building)
dotnet --list-runtimes    # runtimes installed (for running)
dotnet --info             # everything: SDKs, runtimes, RIDs, environment
```

If `dotnet build` fails with "SDK not found," you installed only the runtime. If an app fails with "runtime not found," the target machine has no matching runtime (or publish self-contained).

---

## The runtime families

The runtime ships in a few flavors depending on what your app uses:

| Runtime package | Contains | For |
|---|---|---|
| **.NET Runtime** (`Microsoft.NETCore.App`) | CoreCLR + BCL | console apps, libraries, services |
| **ASP.NET Core Runtime** (`Microsoft.AspNetCore.App`) | the above + web stack | web apps/APIs |
| **.NET Desktop Runtime** (`Microsoft.WindowsDesktop.App`) | the above + WPF/WinForms | Windows desktop apps |

`dotnet --list-runtimes` shows which you have. The ASP.NET Core runtime is a superset that includes the base runtime; the desktop runtime adds Windows UI assemblies.

---

## What the SDK gives you

### The compilers (Roslyn)

The C#/VB compiler. It also exposes APIs for **analyzers** and **source generators** (compile-time code inspection and generation — see CSharpBook Chapter 12 and 15). You don't invoke `csc.exe` directly; the SDK and MSBuild orchestrate it.

### MSBuild — the build engine

Reads your `.csproj` (an MSBuild file) and orchestrates restore → compile → publish. The `dotnet build`/`publish` commands are thin wrappers over MSBuild. Covered in CSharpBook Chapter 15.

### The `dotnet` CLI

The command-line driver for everything:

```bash
dotnet new <template>     # scaffold a project (console, web, classlib, xunit...)
dotnet restore            # fetch NuGet dependencies
dotnet build              # compile
dotnet run                # build + run
dotnet test               # run tests
dotnet publish            # produce deployable output
dotnet pack               # build a NuGet package
dotnet tool install -g X  # install a global tool
dotnet add package X      # add a dependency
dotnet --info             # diagnostics
```

(Full CLI reference: CSharpBook Chapter 15.)

### Templates

`dotnet new list` shows what you can scaffold: `console`, `classlib`, `web`, `webapi`, `mvc`, `blazor`, `maui`, `worker`, `xunit`, `aspire`, `gitignore`, `editorconfig`, and more. Templates are themselves NuGet packages; you can install custom ones.

### NuGet

The package manager. The SDK restores packages declared in your project into a shared cache (`~/.nuget/packages`). The public feed is nuget.org. Covered in CSharpBook Chapter 15.

---

## Picking an SDK version — LTS vs STS

```
Even versions (6, 8, 10) = LTS (Long-Term Support, ~3 years)
Odd versions  (7, 9, 11) = STS (Standard-Term Support, ~18 months)
One release every November.
```

**Recommendation**: target the latest **LTS** for production. Today that's **.NET 10** (supported to November 2028). Adopt STS only for a needed feature, accepting a faster upgrade treadmill.

The SDK is **roll-forward compatible**: a newer SDK can build apps targeting older runtimes. You generally install the latest SDK and set each project's `<TargetFramework>` (e.g., `net10.0`).

### Pinning the SDK with `global.json`

For a repo, pin the SDK version so the whole team and CI build identically:

```json
{
  "sdk": {
    "version": "10.0.100",
    "rollForward": "latestFeature"
  }
}
```

`rollForward` controls how flexibly a newer installed SDK satisfies the pin. Without `global.json`, the latest installed SDK is used.

---

## Where it installs

```
Windows:  C:\Program Files\dotnet\
Linux:    /usr/share/dotnet/  (or via package manager / install script)
macOS:    /usr/local/share/dotnet/
```

Multiple SDK and runtime versions coexist **side-by-side** in version-named subfolders — no "DLL hell," unlike old .NET Framework's single machine-wide install. Each app resolves the runtime version it needs.

```
dotnet/
├── sdk/
│   ├── 8.0.404/
│   └── 10.0.100/
├── shared/
│   ├── Microsoft.NETCore.App/8.0.11/  10.0.0/
│   └── Microsoft.AspNetCore.App/10.0.0/
```

---

## Installing

- **Official installers** from dotnet.microsoft.com (Windows/macOS `.pkg`, Linux packages).
- **Install script** (`dotnet-install.sh` / `.ps1`) — great for CI, installs specific versions without admin.
- **Package managers** — `apt`, `winget`, `brew`, etc.
- **Docker images** — `mcr.microsoft.com/dotnet/sdk` (build) and `.../aspnet` or `.../runtime` (run). Multi-stage Dockerfiles use the SDK image to build and the smaller runtime image to run. See [Chapter 19](../19-Deployment/README.md).

---

## A note on `dotnet workload`

Some targets (MAUI, WASM tooling, Android/iOS) need extra **workloads** beyond the base SDK:

```bash
dotnet workload install maui
dotnet workload list
dotnet workload update
```

Workloads keep the base SDK small; you add only what your project needs (mobile, WASM AOT, etc.).

---

## Summary

- **SDK builds; runtime runs.** The SDK includes the runtime plus Roslyn, MSBuild, the `dotnet` CLI, templates, and NuGet. Use `--list-sdks`/`--list-runtimes`/`--info`.
- Runtimes come in families: **.NET Runtime** (console), **ASP.NET Core Runtime** (web), **.NET Desktop Runtime** (WPF/WinForms).
- Versions install **side-by-side**; pin per-repo with **`global.json`**.
- Choose the latest **LTS** (.NET 10) for production; even=LTS, odd=STS, November releases.
- Extra targets (MAUI/WASM/mobile) need **`dotnet workload`** installs; containers use SDK + runtime Docker images.

→ Next: [05-EcosystemMap.md](05-EcosystemMap.md)
