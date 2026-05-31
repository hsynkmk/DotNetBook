# Serialization

## Turning objects into bytes and back

Serialization converts in-memory objects to a portable representation (JSON, XML, binary) for storage or transmission, and deserialization reverses it. The BCL's primary serializer is **System.Text.Json (STJ)**; XML serializers exist for legacy/interop; and binary serialization is mostly deprecated for security reasons.

> CSharpBook Chapter 13 §05–06 covers STJ and XML in depth (options, `JsonNode`/`JsonDocument`, custom converters, source generation, `XDocument`/`XmlSerializer`). This file is the **decision guide** — which serializer for which job, the security caveats, and the AOT story.

---

## System.Text.Json — the default

STJ is the built-in, high-performance, `Span`/UTF-8-based JSON serializer (replaced Newtonsoft as the ASP.NET Core default):

```csharp
using System.Text.Json;

record Person(string Name, int Age);

string json = JsonSerializer.Serialize(new Person("Alice", 30));   // {"name":"Alice","age":30} (with web defaults)
Person? p = JsonSerializer.Deserialize<Person>(json);

// Stream / async for large payloads:
await JsonSerializer.SerializeAsync(stream, data, options, ct);
await foreach (var item in JsonSerializer.DeserializeAsyncEnumerable<Item>(stream, ct))  // constant memory
    Process(item);
```

Four API levels (CSharpBook Ch13 §05):
1. **`JsonSerializer`** — object ↔ JSON (most common).
2. **`JsonNode`/`JsonObject`/`JsonArray`** — mutable DOM (unknown/dynamic shapes).
3. **`JsonDocument`/`JsonElement`** — read-only, low-allocation DOM (pooled; dispose it).
4. **`Utf8JsonReader`/`Utf8JsonWriter`** — streaming low-level (fastest; ref structs).

**Cache `JsonSerializerOptions`** as a static instance — creating one per call rebuilds its metadata cache (a common perf bug).

---

## STJ source generation — for AOT and speed

Reflection-based serialization isn't AOT/trim-safe and pays startup cost. The **source generator** emits serialization code at compile time:

```csharp
[JsonSourceGenerationOptions(PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase)]
[JsonSerializable(typeof(Person))]
[JsonSerializable(typeof(List<Order>))]
public partial class AppJsonContext : JsonSerializerContext { }

// No reflection — AOT-safe, faster startup
string json = JsonSerializer.Serialize(person, AppJsonContext.Default.Person);
Person? p = JsonSerializer.Deserialize(json, AppJsonContext.Default.Person);
```

**Required for Native AOT** ([Ch01 §03](../01-Runtime/03-NativeAOT.md)) and recommended for trimmed apps and hot serialization paths — it's faster and removes the reflection startup cost. (Source generators: CSharpBook Ch12 §05.)

---

## Shaping JSON

```csharp
public class Product {
    [JsonPropertyName("product_id")] public int Id { get; set; }
    [JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)] public string? Note { get; set; }
    [JsonConverter(typeof(MyMoneyConverter))] public decimal Price { get; set; }
    [JsonExtensionData] public Dictionary<string, JsonElement>? Extra { get; set; }  // capture unknowns
}

// Polymorphism (.NET 7+)
[JsonPolymorphic(TypeDiscriminatorPropertyName = "$type")]
[JsonDerivedType(typeof(Dog), "dog")]
[JsonDerivedType(typeof(Cat), "cat")]
public abstract class Animal { }
```

Naming policies, ignore conditions, custom `JsonConverter<T>`, extension data, and polymorphic (de)serialization are all built in. Records bind via their primary constructor automatically. (Full coverage: CSharpBook Ch13 §05.)

---

## XML — `XmlSerializer` and `DataContractSerializer`

For XML interop (SOAP, legacy systems, document formats):

```csharp
// XmlSerializer — opt-out (serializes all public read/write members)
[XmlRoot("person")]
public class Person { [XmlAttribute] public int Id { get; set; } public string Name { get; set; } = ""; }
private static readonly XmlSerializer Serializer = new(typeof(Person));   // CACHE it!

// DataContractSerializer — opt-in (only [DataMember] members), WCF-era
[DataContract] public class Order { [DataMember] public int Id { get; set; } }
```

**Cache `XmlSerializer` instances** — the simple constructor caches a generated assembly internally, but constructors with extra args leak dynamic assemblies. Mind XML namespaces (the #1 LINQ-to-XML bug) and **XXE security** for untrusted XML (disable DTD processing). For new APIs prefer JSON; use XML only for interop. (Details: CSharpBook Ch13 §06.)

---

## Binary serialization — avoid `BinaryFormatter`

`BinaryFormatter` (the old .NET binary serializer) is **removed/disabled in modern .NET** because it's a **critical security vulnerability** — deserializing untrusted data with it enables remote code execution. Never use it.

```csharp
// ✗ — BinaryFormatter: removed in .NET 9; a known RCE vector. Do not use.
```

If you need compact binary serialization, use:
- **`System.Text.Json`** with UTF-8 (`SerializeToUtf8Bytes`) — fine for most cases.
- **MessagePack** (NuGet) — fast, compact binary, schema-optional.
- **Protocol Buffers / protobuf-net** — schema-based, cross-language (also gRPC's format — [Ch09](../09-NetworkingAndHttp/README.md)).
- **MemoryPack** (NuGet) — zero-encoding, extremely fast .NET-to-.NET binary.

These are safe (no arbitrary type instantiation from the payload) and faster/smaller than the old `BinaryFormatter`.

---

## The security rule for all deserialization

**Treat deserialized data as untrusted input.** Deserialization vulnerabilities arise when a payload can control which types get instantiated or trigger side effects:
- Never use `BinaryFormatter`/`NetDataContractSerializer`/`SoapFormatter` on untrusted data (RCE).
- With STJ polymorphism, **allow-list** derived types (`[JsonDerivedType]`) — don't let the payload name arbitrary types.
- Validate after deserializing (ranges, required fields) — deserialization bypasses constructors/validation in some configurations.
- Set limits (max depth, max length) for untrusted JSON to prevent resource-exhaustion attacks.

```csharp
var opts = new JsonSerializerOptions { MaxDepth = 32 };   // bound nesting for untrusted input
```

---

## Choosing a format

| Format | Pros | Cons | Use |
|---|---|---|---|
| **JSON (STJ)** | human-readable, web-standard, fast, AOT (source-gen) | larger than binary | APIs, config, most data |
| **XML** | schema (XSD), namespaces, comments | verbose, slower | SOAP, legacy, document formats |
| **Protobuf / MessagePack / MemoryPack** | compact, very fast | not human-readable, schema/codegen | high-throughput, gRPC, internal binary |

Default to **JSON** for interchange and config; binary formats for performance-critical internal communication; XML only for existing XML systems. **Never `BinaryFormatter`.**

---

## Common gotchas

### Not caching `JsonSerializerOptions` / `XmlSerializer`

Both cache type metadata internally; creating new instances per call destroys that cache (STJ) or leaks assemblies (XmlSerializer with overrides). Cache them.

### STJ case-sensitivity on deserialize

Case-sensitive by default; `{"name":...}` won't bind to `Name` unless `PropertyNameCaseInsensitive` or camelCase policy is set (ASP.NET's `Web` defaults are insensitive).

### Fields not serialized by STJ

Public **properties** only by default; use `[JsonInclude]` or `IncludeFields = true` for fields.

### Using `BinaryFormatter`

Removed and a security hole. Use STJ or a modern binary serializer.

### Unbounded untrusted deserialization

No depth/size limits and open polymorphism enable DoS/RCE. Allow-list types, set `MaxDepth`, validate output.

### Reflection-based serialization under AOT

Throws/trims. Use STJ source generation for AOT/trimmed apps.

---

## Summary

- **System.Text.Json** is the default: four API levels (`JsonSerializer`, `JsonNode`, `JsonDocument`, `Utf8JsonReader/Writer`); **cache `JsonSerializerOptions`**; use **source generation** for AOT/trim/perf.
- Shape JSON with attributes (naming, ignore, converters, extension data, polymorphism); records bind via primary constructor.
- **XML** (`XmlSerializer` opt-out, `DataContractSerializer` opt-in) for interop only — cache serializers, mind namespaces and XXE.
- **Never use `BinaryFormatter`** (removed; RCE risk). For binary, use MessagePack/Protobuf/MemoryPack.
- **Treat all deserialized data as untrusted**: allow-list polymorphic types, bound depth/size, validate output.
- Default to JSON; binary for performance; XML for legacy. Mechanics: CSharpBook Chapter 13 §05–06.

→ Next: [06-DateTimeAndTime.md](06-DateTimeAndTime.md)
