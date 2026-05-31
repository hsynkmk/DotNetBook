# Assemblies & AssemblyLoadContext

## What an assembly is

An **assembly** is the unit of deployment, versioning, and (partly) isolation in .NET — a `.dll` (or `.exe`) containing IL, metadata, and a manifest. It's what gets loaded, referenced, and versioned. Types live in assemblies; the runtime loads assemblies to resolve types.

```
MyLib.dll  (a PE file)
├── PE/COFF headers       (it's a Portable Executable, like native DLLs)
├── CLR header            (marks it as managed; entry point if executable)
├── Metadata tables       (types, methods, fields, references — see 07-Metadata.md)
├── IL                     (the compiled method bodies)
└── Manifest               (assembly name, version, culture, referenced assemblies, public key)
```

A `.dll` is physically a native PE file with extra managed sections — which is why Windows can "load" it but only the CLR can run its IL.

---

## Assembly identity

An assembly's identity is more than a filename:
```
MyLib, Version=2.1.0.0, Culture=neutral, PublicKeyToken=abc123...
```
- **Name**, **version**, **culture** (for localized satellite assemblies), and an optional **public key token** (for strong-named assemblies).
- **Strong naming** signs the assembly with a key, giving it a unique, tamper-evident identity. Less central in modern .NET than in the Framework GAC era, but still used.

The manifest lists referenced assemblies and their expected versions — the loader uses this to resolve dependencies.

---

## How assemblies load

The loader resolves and loads assemblies **lazily** — on first reference to a type they contain:

1. Code references a type in `MyLib`.
2. The runtime checks if `MyLib` is already loaded in the relevant load context.
3. If not, it **resolves** the file: for apps, via `MyApp.deps.json` (which lists assemblies and probing paths) and the framework/self-contained locations.
4. It maps the PE file, reads the manifest/metadata, and makes the types available.

`deps.json` and `runtimeconfig.json` (from [01-CoreCLRArchitecture.md](01-CoreCLRArchitecture.md)) drive this resolution. Lazy loading means an assembly you never touch is never loaded — keeping startup lean.

---

## AssemblyLoadContext — the modern isolation primitive

In .NET Framework, isolation was done with **AppDomains** (separate "sub-processes" within a process). Modern .NET **does not have AppDomains** — instead it has **`AssemblyLoadContext` (ALC)**, a lighter mechanism for loading assemblies into isolated, optionally-unloadable contexts.

```csharp
using System.Runtime.Loader;

// The runtime always has a Default context (your app + framework load here)
AssemblyLoadContext defaultCtx = AssemblyLoadContext.Default;

// Create an isolated, collectible context (e.g., for a plugin)
var pluginCtx = new AssemblyLoadContext(name: "Plugin1", isCollectible: true);
Assembly pluginAsm = pluginCtx.LoadFromAssemblyPath("/plugins/Plugin1.dll");
// ... use the plugin via interfaces ...
pluginCtx.Unload();   // request unload; assemblies GC'd once nothing references them
```

Key properties of an ALC:
- **Isolation** — different ALCs can load **different versions** of the same assembly without conflict (each context resolves independently).
- **Collectible / unloadable** — a `isCollectible: true` ALC can be **unloaded**, freeing its assemblies once no references remain. This enables plugin reload without restarting the process.
- **Custom resolution** — override `Load(AssemblyName)` to control how dependencies resolve (e.g., load a plugin's private deps from its own folder).

---

## The plugin pattern

The canonical use of ALC is **isolated extensibility** — loading plugins that may have their own dependency versions, with the ability to unload them:

```csharp
public sealed class PluginLoadContext(string pluginPath)
    : AssemblyLoadContext(isCollectible: true) {

    private readonly AssemblyDependencyResolver _resolver = new(pluginPath);

    protected override Assembly? Load(AssemblyName name) {
        // Resolve the plugin's private dependencies from its own folder
        string? path = _resolver.ResolveAssemblyToPath(name);
        return path is null ? null   // fall back to Default context (shared contracts)
                            : LoadFromAssemblyPath(path);
    }
}

// Usage
var ctx = new PluginLoadContext("/plugins/Acme/Acme.Plugin.dll");
var asm = ctx.LoadFromAssemblyPath("/plugins/Acme/Acme.Plugin.dll");
var type = asm.GetType("Acme.Plugin")!;
var plugin = (IPlugin)Activator.CreateInstance(type)!;   // IPlugin is a SHARED contract
plugin.Run();
```

`AssemblyDependencyResolver` reads the plugin's own `.deps.json` to find its private dependencies — so two plugins can use different versions of the same library.

### The shared-contract rule

The host and plugin must agree on a **shared interface** (`IPlugin`). That contract assembly should load in the **Default** context (so both host and plugin see the *same* `IPlugin` type). If the plugin loads its own copy of the contract, the cast `(IPlugin)` fails — the two `IPlugin`s are different types even with the same name. This is the #1 ALC gotcha: **return `null` from `Load` for shared contracts so they fall back to Default**.

---

## Unloading — how it actually works

`Unload()` is a *request*, not immediate. The ALC and its assemblies are collected only when **nothing references them** — no live objects of plugin types, no cached `Type`/`MethodInfo`, no running threads in plugin code, no static references. Common reasons an unload "doesn't work":
- A host field still holds a plugin object.
- A cached delegate/`Type` from the plugin.
- An event subscription keeping a plugin object alive.

After dropping all references and calling `Unload()`, a GC (sometimes two) reclaims the context. Verify with a `WeakReference` to the ALC.

---

## Reflection-only and metadata inspection

To inspect an assembly **without running its code** (e.g., a tool analyzing plugins), use `MetadataLoadContext` (from `System.Reflection.MetadataLoadContext`), which loads assemblies purely for metadata reading — no execution, no JIT, no Default-context pollution. Good for build tools and analyzers. (Reflection internals: [07-Metadata.md](07-Metadata.md).)

---

## Common gotchas

### Shared contract loaded in the wrong context

Causes `InvalidCastException` casting the plugin to your interface. Keep contracts in Default; return `null` from the ALC's `Load` for them.

### Unload that never completes

A lingering reference (field, cache, event, thread) roots the context. Audit for held references; use `WeakReference` to confirm collection.

### Assuming AppDomains exist

`AppDomain.CreateDomain` throws on modern .NET. Use `AssemblyLoadContext` (isolation) or separate processes (strong isolation/security).

### Version conflicts in Default context

Two libraries needing different versions of a dependency, both in Default, conflict (the loader picks one). ALCs isolate versions; or align versions via Central Package Management (CSharpBook Ch15).

---

## Summary

- An **assembly** (`.dll`/`.exe`) is the unit of deployment/versioning — a PE file of IL + metadata + a manifest, with an identity (name, version, culture, optional public key).
- Assemblies load **lazily** on first type reference, resolved via `deps.json`/`runtimeconfig.json`.
- Modern .NET has **no AppDomains**; isolation is via **`AssemblyLoadContext`** — supports loading multiple versions, custom resolution, and (when `isCollectible`) **unloading**.
- The **plugin pattern**: a collectible ALC with `AssemblyDependencyResolver`, plus a **shared contract** kept in the Default context (return `null` from `Load` for it) so casts work.
- **Unload is a request** — it completes only when all references are gone; lingering refs are the usual culprit.
- Use **`MetadataLoadContext`** to inspect assemblies without executing them.

→ Next: [07-Metadata.md](07-Metadata.md)
