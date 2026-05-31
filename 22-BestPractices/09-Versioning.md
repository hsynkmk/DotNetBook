# Versioning

## Evolving APIs and libraries without breaking consumers

Software changes, but **consumers depend on its current shape** — other services calling your API, other projects referencing your library. **Versioning** is how you evolve while managing that dependency: signaling what changed, letting consumers opt into new versions, and not breaking them unexpectedly. Two distinct cases: **API versioning** (HTTP APIs — multiple versions served simultaneously so clients migrate on their own schedule) and **semantic versioning** (libraries/NuGet packages — communicating the *nature* of a change so consumers know if it's safe to upgrade). The unifying principle: **a breaking change is a contract violation** — version it explicitly rather than silently breaking callers.

---

## Semantic versioning (SemVer) for libraries

**SemVer** (`MAJOR.MINOR.PATCH`, e.g., `3.2.1`) encodes the *nature* of a change so consumers can reason about upgrade safety:

| Part | Increment when | Consumer impact |
|---|---|---|
| **MAJOR** | a **breaking** change | may need code changes — upgrade deliberately |
| **MINOR** | new functionality, **backward-compatible** | safe to upgrade; new features available |
| **PATCH** | backward-compatible **bug fix** | safe to upgrade |

The contract: bumping **MAJOR** signals "this may break you"; MINOR/PATCH promise backward compatibility. This lets consumers (and NuGet version ranges) upgrade safely within a major version and treat major bumps with care. **What counts as breaking** for a library: removing/renaming public members, changing signatures/behavior, altering serialization — anything a dependent's code or runtime relies on ([CSharpBook Ch17 API Design](../../CSharpBook/17-BestPractices/README.md)). Honoring SemVer is a *promise*; violating it (a breaking change in a minor bump) erodes trust and breaks builds.

```xml
<!-- In the .csproj for a NuGet package: -->
<Version>3.2.1</Version>
<AssemblyVersion>3.0.0.0</AssemblyVersion>   <!-- often pinned to MAJOR for binding -->
```

Tools like **public API analyzers** can flag accidental breaking changes before release.

---

## API versioning for HTTP APIs

For HTTP APIs consumed by clients you don't control (or can't upgrade in lockstep), you serve **multiple versions simultaneously** so clients migrate on their own schedule. The **`Asp.Versioning`** library (the modern successor to Microsoft's API versioning) provides this:

```csharp
builder.Services.AddApiVersioning(o => {
    o.DefaultApiVersion = new ApiVersion(1, 0);
    o.AssumeDefaultVersionWhenUnspecified = true;
    o.ReportApiVersions = true;            // advertise supported versions in responses
}).AddApiExplorer(o => o.GroupNameFormat = "'v'VVV");   // for OpenAPI ([Ch04 §10])
```

### Where the version goes

Common strategies for a client to specify the version:

| Strategy | Example | Notes |
|---|---|---|
| **URL path** | `/api/v1/orders`, `/api/v2/orders` | most explicit/visible; common |
| **Query string** | `/api/orders?api-version=2.0` | easy, less RESTful-purist |
| **Header** | `api-version: 2.0` | clean URLs; less discoverable |
| **Media type** | `Accept: application/json;v=2.0` | content negotiation ([Ch04 §13](../04-AspNetCore/13-ContentNegotiation.md)) |

```csharp
[ApiVersion("1.0")] [ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/orders")]
public class OrdersController : ControllerBase {
    [HttpGet, MapToApiVersion("2.0")] public IActionResult GetV2() => ...;
}
```

Whichever strategy, the goal is the same: **old and new versions coexist**, clients pick a version, and you can introduce a breaking change in `v2` without breaking `v1` consumers — then **deprecate** v1 with notice and eventually retire it. Integrate with **OpenAPI** ([Ch04 §10](../04-AspNetCore/10-OpenAPI.md)) so each version is documented.

---

## Managing breaking changes

The strategy for both APIs and libraries is to **avoid breaking changes when possible, and version them explicitly when not**:

- **Add, don't change/remove** — adding an optional property/endpoint/overload is backward-compatible (MINOR); removing/renaming/changing is breaking (MAJOR/new API version).
- **Deprecate before removing** — mark old members `[Obsolete]` (libraries) or advertise deprecation headers (`Sunset` — APIs), give consumers time/notice, *then* remove in the next major/version.
- **Tolerant readers** — clients that ignore unknown fields tolerate additive server changes (a serialization best practice — [Ch02 §05](../02-BCL/05-Serialization.md)).
- **Document the change** — changelogs/release notes; for APIs, advertise supported/deprecated versions (`ReportApiVersions`).

The discipline is treating the public surface (API or library) as a **contract** and changing it responsibly.

---

## Common gotchas

### Breaking change in a minor/patch bump

Removing/changing public behavior in a MINOR/PATCH release violates SemVer and breaks consumers' builds silently. Breaking changes require a **MAJOR** bump (or a new API version).

### Not versioning a public API at all

A single unversioned API means any breaking change breaks all clients at once. Version from the start (even v1) so you can evolve to v2 without breaking v1.

### Removing without deprecating

Deleting an endpoint/member with no notice strands consumers. **Deprecate** (`[Obsolete]` / sunset headers), give a migration window, then remove.

### Versioning everything prematurely

Over-versioning (a new version for every tiny change) explodes the surface to maintain. Version when there's a **breaking** change; use additive changes for the rest.

### Forgetting OpenAPI/docs per version

Multiple API versions without per-version docs confuse consumers. Wire API versioning into OpenAPI ([Ch04 §10](../04-AspNetCore/10-OpenAPI.md)) so each version is discoverable.

---

## Summary

- **Versioning** evolves software without breaking dependents; a **breaking change is a contract violation** — version it explicitly, don't break callers silently.
- **SemVer** (`MAJOR.MINOR.PATCH`) for libraries/NuGet communicates change nature: **MAJOR** = breaking (upgrade deliberately), **MINOR** = backward-compatible features, **PATCH** = compatible fixes — honoring it lets consumers upgrade safely; breaking what counts as the public surface ([CSharpBook Ch17](../../CSharpBook/17-BestPractices/README.md)) requires a MAJOR bump.
- **API versioning** (**`Asp.Versioning`**) serves **multiple versions simultaneously** (via URL path/query/header/media type) so clients migrate on their own schedule — introduce breaking changes in a new version without breaking old consumers; integrate with **OpenAPI** ([Ch04 §10](../04-AspNetCore/10-OpenAPI.md)).
- Manage change by **adding rather than removing/changing**, **deprecating before removing** (`[Obsolete]`/sunset headers with notice), using **tolerant readers**, and documenting — version when there's a **breaking** change, not for every tweak.

→ Next: [10-AntiPatterns.md](10-AntiPatterns.md)
