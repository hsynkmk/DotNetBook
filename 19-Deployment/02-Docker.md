# Docker

## Containerizing a .NET app

Containers are the dominant deployment unit for server-side .NET: a container bundles your app and its runtime into a portable image that runs identically on any host with a container runtime. .NET has first-class container support — both a hand-written **multi-stage Dockerfile** and a **Dockerfile-less** SDK publish (`dotnet publish` directly to an image). The keys to a good .NET container are: **multi-stage builds** (build with the SDK, run on a small runtime image), **layer caching** (restore before copying source), and **choosing the right base image** (size vs compatibility).

```dockerfile
# Multi-stage Dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY ["MyApi.csproj", "./"]
RUN dotnet restore                       # cached layer — only re-runs if the csproj changes
COPY . .
RUN dotnet publish -c Release -o /app --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS final   # small runtime-only image
WORKDIR /app
COPY --from=build /app .
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

---

## Multi-stage builds

A multi-stage Dockerfile uses **two images**: a large **SDK image** to compile/publish, and a small **runtime image** to actually run — copying only the published output into the final stage. This keeps the shipped image small (no compilers, no SDK) while still building inside Docker:

- **`build` stage**: `mcr.microsoft.com/dotnet/sdk` — has the full SDK to `restore`/`publish`.
- **`final` stage**: `mcr.microsoft.com/dotnet/aspnet` (or `runtime` for non-web) — only the runtime needed to execute.

The final image contains your app + the runtime, not the build toolchain — dramatically smaller and more secure (less attack surface).

---

## Layer caching — order matters

Docker caches each layer and reuses it if its inputs haven't changed. The classic optimization: **copy the project file and `restore` *before* copying the full source**, so the (slow) NuGet restore layer is cached and only re-runs when dependencies change — not on every source edit:

```dockerfile
COPY ["MyApi.csproj", "./"]
RUN dotnet restore          # ← cached unless the csproj changes
COPY . .                    # source changes invalidate from here down only
RUN dotnet publish -c Release -o /app --no-restore
```

Without this ordering, any source change busts the cache back to `restore`, making every build re-download all packages — slow. Order Dockerfile steps from least- to most-frequently-changing.

---

## Base image choices

The runtime base image trades **size, compatibility, and security**:

| Image | Base | Size | Notes |
|---|---|---|---|
| `aspnet:10.0` | Debian | medium | default; broad compatibility |
| `aspnet:10.0-alpine` | Alpine (musl) | small | tiny, but musl libc — test native deps |
| `aspnet:10.0-noble-chiseled` | Ubuntu Chiseled | very small | minimal, **no shell**, non-root, low CVE surface ([03-ChiseledImages.md](03-ChiseledImages.md)) |
| `aspnet:10.0-azurelinux` | Azure Linux | small | Microsoft's distro |

- **Debian** (default) — most compatible, larger.
- **Alpine** — very small, but uses **musl** libc (not glibc); some native dependencies need the musl RID (`linux-musl-x64`) and testing.
- **Chiseled** — Microsoft's ultra-minimal images: no package manager, **no shell**, runs as **non-root**, far fewer CVEs — the modern recommendation for production ([03-ChiseledImages.md](03-ChiseledImages.md)).

Smaller images mean faster pulls, less storage, and a smaller attack surface — prefer Chiseled/Alpine for production unless a dependency needs the fuller Debian image.

---

## Dockerfile-less publish (SDK container build)

.NET can build a container image **without a Dockerfile** using the SDK's built-in container support:

```bash
dotnet publish -c Release --os linux --arch x64 /t:PublishContainer
# or configure in the .csproj:
#   <EnableSdkContainerSupport>true</EnableSdkContainerSupport>
#   <ContainerBaseImage>mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled</ContainerBaseImage>
```

This produces an OCI image directly from `dotnet publish`, using sensible defaults (multi-layer, non-root, Chiseled base) — no Dockerfile to maintain. It's the simplest path for standard apps; use a hand-written Dockerfile when you need custom build steps, native deps, or fine control.

---

## Container best practices

- **Run as non-root** — Chiseled images default to this; otherwise add a non-root `USER`. Reduces blast radius if compromised.
- **`EXPOSE`/ports** — ASP.NET Core in containers listens on port 8080 by default (modern images); configure via `ASPNETCORE_HTTP_PORTS`.
- **Health checks** — define container/orchestrator health probes hitting your `/health` and `/alive` endpoints ([Ch04 §16](../04-AspNetCore/16-HealthChecks.md)).
- **Configuration via environment** — pass config as env vars ([Ch13 §02](../13-Configuration/02-Providers.md)); don't bake secrets into the image ([Ch13 §07](../13-Configuration/07-Secrets.md)).
- **Graceful shutdown** — the host handles SIGTERM to drain requests; ensure your app respects cancellation ([Ch03 §01](../03-HostingAndDI/01-GenericHost.md)).
- **`.dockerignore`** — exclude `bin/`, `obj/`, `.git/` so they don't bloat the build context or bust caching.

---

## Common gotchas

### Building on the SDK image as the final image

Shipping the SDK image (with compilers) is huge and insecure. Use **multi-stage** builds — publish on the SDK image, run on the small runtime image.

### Cache-busting layer order

Copying all source before `restore` invalidates the restore layer on every edit, re-downloading packages each build. Copy the csproj and `restore` first, then the source.

### Alpine native-dependency surprises

Alpine uses musl, not glibc — some native libraries fail or need the `linux-musl-x64` RID. Test native deps on Alpine; use Debian/Chiseled if they don't cooperate.

### Baking secrets into the image

Secrets in the image (or layers) leak — anyone with the image can read them. Inject secrets at runtime (env/secret store — [Ch13 §07](../13-Configuration/07-Secrets.md)), never in the Dockerfile.

### Missing `.dockerignore`

Without it, `bin/obj/.git` get copied into the build context, bloating it and busting cache. Always include a `.dockerignore`.

---

## Summary

- Containerize .NET with a **multi-stage Dockerfile** (build on the **SDK** image, run on a small **runtime** image — `aspnet`/`runtime`) or the **Dockerfile-less SDK build** (`/t:PublishContainer`) for standard apps.
- Optimize **layer caching** by copying the **csproj + `restore` before** the full source, so package restore is cached and only re-runs when dependencies change.
- Choose the **base image** by size/compatibility/security: Debian (default, compatible), **Alpine** (tiny, musl — test native deps), **Chiseled** (ultra-minimal, **no shell**, **non-root**, low CVEs — the production recommendation).
- Follow container best practices: **non-root**, env-based config (no baked secrets), **health probes** to `/health`/`/alive`, graceful SIGTERM shutdown, and a **`.dockerignore`**.

→ Next: [03-ChiseledImages.md](03-ChiseledImages.md)
