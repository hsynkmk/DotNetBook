# Chapter 01 — Runtime & CLR — Hands-On

These exercises make the runtime internals tangible: inspect IL, watch tiered compilation and the JIT's output, observe the GC, dump method tables, build an unloadable plugin, and measure P/Invoke overhead. Run on .NET 10, Release config where noted.

---

## Exercise 1: See IL, then the JIT'd native code

Observe the compile-to-IL, JIT-to-native pipeline.

<details><summary>Solution</summary>

```bash
dotnet new console -o JitDemo && cd JitDemo
```

```csharp
// Program.cs
int Square(int x) => x * x;
Console.WriteLine(Square(7));
```

```bash
dotnet build -c Release
ilspycmd bin/Release/net10.0/JitDemo.dll        # IL: ldarg, mul, ret

# Native code the JIT produced:
DOTNET_JitDisasm="*Square*" dotnet bin/Release/net10.0/JitDemo.dll
```

`DOTNET_JitDisasm` prints the actual x64/ARM instructions RyuJIT emitted — you'll see the multiply and likely inlining. This is the IL → native step from [02-JIT.md](02-JIT.md).

</details>

---

## Exercise 2: Watch tiered compilation happen

See a method recompile from Tier 0 to Tier 1.

<details><summary>Solution</summary>

```csharp
// Program.cs — call a method enough to make it "hot"
static long Hot(long n) { long s = 0; for (long i = 0; i < n; i++) s += i % 7; return s; }
for (int i = 0; i < 200; i++) _ = Hot(1_000_000);
```

```bash
# Show tiering decisions
DOTNET_TieredCompilation=1 DOTNET_JitDisasmSummary=1 dotnet run -c Release
```

Compare timing with tiering off (always Tier 1 from the start):

```bash
DOTNET_TieredCompilation=0 dotnet run -c Release    # slower startup, no Tier 0
```

The first run starts faster (Tier 0) then promotes `Hot` to Tier 1; the second compiles everything optimized upfront. This is tiered compilation + OSR from [02-JIT.md](02-JIT.md). (Note: env-var names/output evolve across versions; the concept is the point.)

</details>

---

## Exercise 3: Observe the GC live

Generate garbage and watch generations collect.

<details><summary>Solution</summary>

```csharp
// Program.cs — allocate steadily
while (true) {
    var junk = new byte[10_000];          // Gen 0 churn
    if (Random.Shared.Next(100) == 0) GC.KeepAlive(new byte[100_000]); // occasional LOH
    Thread.Sleep(1);
}
```

In another terminal:

```bash
dotnet-counters monitor -p <pid> System.Runtime
```

Watch `gen-0-gc-count` climb fast, `gen-2-gc-count` slowly, `gc-heap-size`, `alloc-rate`, and `% time in gc`. Try toggling Server GC and DATAS in the `.csproj` and compare memory:

```xml
<ServerGarbageCollection>true</ServerGarbageCollection>
```

This makes the generational model and GC modes from [04-GCDeepDive.md](04-GCDeepDive.md) concrete.

</details>

---

## Exercise 4: Demonstrate the cost of boxing (type system)

Show that value-type generics avoid the boxing the type system would otherwise impose.

<details><summary>Solution</summary>

```csharp
// BenchmarkDotNet project, Release
[MemoryDiagnoser]
public class BoxingBench {
    [Benchmark(Baseline = true)]
    public long GenericSum() {                 // List<int> specialized — no boxing
        var list = new List<int>();
        for (int i = 0; i < 1000; i++) list.Add(i);
        long s = 0; foreach (int x in list) s += x;
        return s;
    }

    [Benchmark]
    public long ObjectSum() {                    // ArrayList boxes every int
        var list = new System.Collections.ArrayList();
        for (int i = 0; i < 1000; i++) list.Add(i);   // box
        long s = 0; foreach (object x in list) s += (int)x;  // unbox
        return s;
    }
}
```

`ObjectSum` allocates ~1000 boxes (visible in the `Allocated` column); `GenericSum` allocates only the backing array. This shows value-type generic **specialization** from [05-TypeSystem.md](05-TypeSystem.md) — `List<int>` stores raw ints inline.

</details>

---

## Exercise 5: Dump a method table

Inspect the runtime's type representation.

<details><summary>Solution</summary>

```csharp
// Program.cs
public class Animal { public virtual void Speak() {} }
public class Dog : Animal { public override void Speak() {} public int Age; }
var d = new Dog();
Console.WriteLine("attach now, then press enter"); Console.ReadLine();
GC.KeepAlive(d);
```

```bash
dotnet-dump collect -p <pid>
dotnet-dump analyze <dumpfile>
  > dumpheap -type Dog      # find the Dog instance address
  > dumpobj <addr>          # shows its MethodTable pointer + fields (Age)
  > dumpmt -md <mt-addr>    # the method table: methods, vtable slots, EEClass
```

You'll see `Dog`'s vtable with `Speak` pointing at `Dog.Speak` (the override reusing the base slot) — the dispatch mechanism from [05-TypeSystem.md](05-TypeSystem.md).

</details>

---

## Exercise 6: Build an unloadable plugin with AssemblyLoadContext

Load a plugin in a collectible context and unload it.

<details><summary>Solution</summary>

```csharp
// Shared contract (in the host / Default context)
public interface IPlugin { string Run(); }

// Plugin (separate project → Plugin.dll), references the contract
public class HelloPlugin : IPlugin { public string Run() => "hello from plugin"; }

// Host
using System.Reflection;
using System.Runtime.Loader;

static (WeakReference, string) Load(string path) {
    var ctx = new AssemblyLoadContext("plugin", isCollectible: true);
    var asm = ctx.LoadFromAssemblyPath(Path.GetFullPath(path));
    var type = asm.GetTypes().First(t => typeof(IPlugin).IsAssignableFrom(t) && !t.IsInterface);
    var plugin = (IPlugin)Activator.CreateInstance(type)!;
    string result = plugin.Run();
    ctx.Unload();                          // request unload
    return (new WeakReference(ctx), result); // don't keep plugin/ctx refs alive!
}

var (wr, msg) = Load("Plugin.dll");
Console.WriteLine(msg);
for (int i = 0; wr.IsAlive && i < 10; i++) { GC.Collect(); GC.WaitForPendingFinalizers(); }
Console.WriteLine($"Unloaded: {!wr.IsAlive}");   // True if no references linger
```

Key points from [06-Assemblies.md](06-Assemblies.md): the contract `IPlugin` lives in the Default context (shared type), the plugin loads in a **collectible** ALC, and unload completes only after all references drop (note the method returns *no* live plugin references). If `Unloaded: False`, something is rooting the context.

</details>

---

## Exercise 7: Read metadata without executing code

Inspect an assembly's types via `MetadataLoadContext`.

<details><summary>Solution</summary>

```csharp
using System.Reflection;

var resolver = new PathAssemblyResolver(
    Directory.GetFiles(RuntimeEnvironment.GetRuntimeDirectory(), "*.dll")
        .Append(Path.GetFullPath("SomeLib.dll")));

using var mlc = new MetadataLoadContext(resolver);
Assembly asm = mlc.LoadFromAssemblyPath(Path.GetFullPath("SomeLib.dll"));

foreach (var t in asm.GetTypes())
    Console.WriteLine($"{t.FullName} : {t.GetMethods().Length} methods");
// No code from SomeLib runs — pure metadata reading ([07-Metadata.md]).
```

This is how analyzers, the trimmer, and doc tools inspect assemblies — reading the metadata tables without loading types into the executing runtime.

</details>

---

## Exercise 8: Measure P/Invoke transition overhead

Quantify the GC-mode transition cost and `SuppressGCTransition`.

<details><summary>Solution</summary>

```csharp
using System.Runtime.CompilerServices;
using System.Runtime.InteropServices;

public static partial class Native {
    [LibraryImport("c", EntryPoint = "abs")]                 // libc abs (Linux/macOS)
    public static partial int Abs(int x);

    [LibraryImport("c", EntryPoint = "abs")]
    [SuppressGCTransition]                                     // skip the transition
    public static partial int AbsFast(int x);
}

// BenchmarkDotNet, Release
[Benchmark] public int WithTransition()  { int s=0; for(int i=0;i<1000;i++) s+=Native.Abs(-i);     return s; }
[Benchmark] public int WithoutTransition(){ int s=0; for(int i=0;i<1000;i++) s+=Native.AbsFast(-i); return s; }
```

`WithoutTransition` is measurably faster — it skips the cooperative↔preemptive switch from [09-PInvokeInternals.md](09-PInvokeInternals.md). Caveat: `SuppressGCTransition` is safe here only because `abs` is trivial and non-blocking; never use it on a call that could block or run long (it stalls GC process-wide). `int` is blittable, so there's no marshaling cost — only the transition.

</details>

---

## Exercise 9: Trigger and diagnose thread-pool starvation

Reproduce the classic blocking bug and observe it.

<details><summary>Solution</summary>

```csharp
// Program.cs — block pool threads with sync-over-async
async Task<int> SlowAsync() { await Task.Delay(500); return 1; }

var sw = System.Diagnostics.Stopwatch.StartNew();
var tasks = Enumerable.Range(0, 200)
    .Select(_ => Task.Run(() => SlowAsync().Result))   // ✗ blocks a pool thread each
    .ToArray();
await Task.WhenAll(tasks);
Console.WriteLine($"Blocking version: {sw.ElapsedMilliseconds} ms");
```

Monitor with `dotnet-counters monitor -p <pid> System.Runtime` — watch `ThreadPool Queue Length` spike and `ThreadPool Thread Count` climb slowly. Now fix it:

```csharp
var tasks = Enumerable.Range(0, 200).Select(_ => SlowAsync()).ToArray();   // ✓ no blocking
await Task.WhenAll(tasks);
```

The fixed version finishes in ~500 ms (all run concurrently); the blocking version takes far longer as the pool slowly injects threads. This is the starvation mechanism from [08-Threading.md](08-Threading.md) — the runtime reason for "never block on async."

</details>

---

You've now seen IL→native, tiering, the live GC, method tables, value-type specialization, collectible plugin loading, metadata-only inspection, P/Invoke transition cost, and thread-pool starvation — the runtime concepts made concrete. Next, the standard library that runs on top of all this.

→ Back to [Chapter 01 README](README.md) · Next chapter: [Chapter 02 — Base Class Library](../02-BCL/README.md)
