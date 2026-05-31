# Chapter 00 — Overview — Hands-On

This chapter's "coding" is **exploration**: inspect your installed .NET, scaffold projects, and observe how the pieces fit. Run these to ground the concepts before diving into the deep chapters.

---

## Exercise 1: Inventory your installation

Find out exactly what .NET you have.

<details><summary>Solution</summary>

```bash
dotnet --version          # active SDK version (e.g., 10.0.100)
dotnet --list-sdks        # all SDKs (for building)
dotnet --list-runtimes    # all runtimes (for running)
dotnet --info             # everything: SDKs, runtimes, RID, OS, env vars
```

Read the output: SDKs let you build; runtimes let you run. `Microsoft.NETCore.App` is the base runtime, `Microsoft.AspNetCore.App` adds the web stack, `Microsoft.WindowsDesktop.App` (Windows) adds WPF/WinForms. The `RID` (Runtime Identifier, e.g., `win-x64`, `linux-x64`) is your platform.

</details>

---

## Exercise 2: See available templates

List what you can scaffold, then create one of each major type.

<details><summary>Solution</summary>

```bash
dotnet new list                 # all installed templates

dotnet new console  -o HelloConsole
dotnet new webapi   -o HelloApi
dotnet new classlib -o HelloLib
dotnet new worker   -o HelloWorker      # a Worker Service (Chapter 08!)
```

Each template produces a `.csproj` and starter code. Open the `.csproj` files — note how concise the SDK-style format is. Notice `webapi`'s `.csproj` uses `<Project Sdk="Microsoft.NET.Sdk.Web">` while `console` uses `Microsoft.NET.Sdk`.

</details>

---

## Exercise 3: Run a file-based app (no project)

.NET 10 / C# 14 lets you run a single `.cs` file directly.

<details><summary>Solution</summary>

```csharp
// hello.cs
Console.WriteLine($".NET version: {Environment.Version}");
Console.WriteLine($"OS: {System.Runtime.InteropServices.RuntimeInformation.OSDescription}");
Console.WriteLine($"Process arch: {System.Runtime.InteropServices.RuntimeInformation.ProcessArchitecture}");
```

```bash
dotnet run hello.cs
```

No `.csproj` needed — the SDK synthesizes one. `Environment.Version` shows the runtime version actually executing your code. (File-based apps are covered in CSharpBook Chapter 11.)

</details>

---

## Exercise 4: Inspect the IL your code compiles to

See the "intermediate language" the runtime actually loads.

<details><summary>Solution</summary>

```bash
dotnet new console -o IlDemo
cd IlDemo
dotnet build
```

Then inspect the produced assembly with a disassembler:

```bash
# Option A: ildasm (Windows SDK) or ilspycmd (cross-platform)
dotnet tool install -g ilspycmd
ilspycmd bin/Debug/net10.0/IlDemo.dll | head -40
```

You'll see IL opcodes (`ldstr`, `call`, `ret`) — not C#, not machine code. This is what the JIT compiles to native at runtime. It demonstrates the compile-to-IL, JIT-at-runtime model from [01-WhatIsDotNet.md](01-WhatIsDotNet.md).

</details>

---

## Exercise 5: Compare a framework-dependent vs self-contained publish

See the SDK-vs-runtime distinction concretely.

<details><summary>Solution</summary>

```bash
cd HelloConsole

# Framework-dependent (needs a runtime installed on the target)
dotnet publish -c Release -o out-fdd
ls -la out-fdd                         # small — just your DLL + deps

# Self-contained (bundles the runtime — no install needed on target)
dotnet publish -c Release -r linux-x64 --self-contained -o out-scd
du -sh out-scd                         # large (~70 MB) — includes the runtime
```

Framework-dependent output is tiny but needs the matching runtime present. Self-contained is large but runs anywhere of that RID with nothing pre-installed. This is the practical face of "SDK builds, runtime runs." (Publish modes: [Chapter 19](../19-Deployment/README.md) and CSharpBook Chapter 14.)

</details>

---

## Exercise 6: Pin the SDK with global.json

Make a repo build with a specific SDK.

<details><summary>Solution</summary>

```bash
dotnet new globaljson --sdk-version 10.0.100 --roll-forward latestFeature
cat global.json
```

```json
{
  "sdk": {
    "version": "10.0.100",
    "rollForward": "latestFeature"
  }
}
```

Now any `dotnet` command in this directory tree uses (at least) SDK 10.0.100. If that exact version isn't installed, `rollForward` decides whether a newer one satisfies it. Commit this to ensure reproducible builds across the team and CI.

</details>

---

## Exercise 7: Identify which runtime a workload uses

Reason about runtime selection.

<details><summary>Solution</summary>

For each, name the runtime (answers below):

1. `dotnet new webapi` → run on a Linux server.
2. A MAUI app deployed to an iPhone.
3. A Blazor WebAssembly app in the browser.
4. A CLI tool published with `<PublishAot>true</PublishAot>`.
5. A WPF desktop app on Windows.

**Answers:**
1. **CoreCLR** (default for server).
2. **Mono** with AOT (iOS forbids JIT).
3. **Mono** (interpreter + optional AOT) running in the browser's WASM sandbox.
4. **Native AOT** (no JIT, single native binary).
5. **CoreCLR** (via the .NET Desktop Runtime).

You rarely choose the runtime explicitly — the workload selects it (only AOT is opt-in). See [03-Runtimes.md](03-Runtimes.md).

</details>

---

## Exercise 8: Map a requirement to a chapter

Practice navigating the book.

<details><summary>Solution</summary>

For each need, which chapter?

| Need | Chapter |
|---|---|
| Build a JSON HTTP API | [04 — ASP.NET Core](../04-AspNetCore/README.md) |
| Talk to a SQL database with an ORM | [05 — EF Core](../05-EFCore/README.md) |
| Run a nightly cleanup job | [08 — Background Processing](../08-BackgroundProcessing/README.md) |
| Service-to-service RPC | [09 — Networking & HTTP](../09-NetworkingAndHttp/README.md) (gRPC) |
| Retry transient failures | [11 — Resilience](../11-Resilience/README.md) |
| Distributed tracing | [12 — Observability](../12-Observability/README.md) |
| Orchestrate API + worker + Redis locally | [18 — .NET Aspire](../18-Aspire/README.md) |
| Ship as a small native binary | [19 — Deployment](../19-Deployment/README.md) |

This is the mental index you'll build as you read.

</details>

---

You now have a working .NET installation, understand SDK vs runtime, have seen IL and the publish models, and can map needs to chapters. The deep chapters start at [Chapter 01 — Runtime & CLR](../01-Runtime/README.md).

→ Back to [Chapter 00 README](README.md) · Next chapter: [Chapter 01 — Runtime & CLR](../01-Runtime/README.md)
