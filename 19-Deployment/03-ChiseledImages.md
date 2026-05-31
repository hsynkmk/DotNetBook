# Chiseled Images

## Minimal, hardened base images

**Chiseled images** are Microsoft's ultra-minimal Linux base images for .NET, built on **Ubuntu Chiseled** (a "distroless"-style approach). They strip the OS down to the bare minimum needed to run a .NET app: **no package manager, no shell, no unnecessary libraries**, running as a **non-root** user by default. The payoff is a dramatically **smaller image**, a **smaller attack surface**, and **far fewer CVEs** to patch — making them the recommended base for production .NET containers ([02-Docker.md](02-Docker.md)).

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled AS final
WORKDIR /app
COPY --from=build /app .
USER $APP_UID          # non-root (set by the chiseled image)
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

---

## What's removed (and why it's good)

A standard Debian/Ubuntu base image includes a full userland — a shell (`bash`), a package manager (`apt`), many libraries and utilities — most of which your app never uses but which **expand the attack surface and the CVE count**. Chiseled removes them:

| Removed | Security/size benefit |
|---|---|
| **Shell** (`bash`/`sh`) | no shell for an attacker to exploit (no `kubectl exec` into a shell) |
| **Package manager** (`apt`) | can't install tools post-compromise; smaller |
| **Most OS utilities** | fewer binaries = fewer vulnerabilities |
| **Root by default** | runs as **non-root** — limits blast radius |

The result is a "distroless"-style image: just your app, the .NET runtime, and the minimal libraries needed — nothing to exploit, little to patch. Fewer packages directly means **fewer reported CVEs**, so your security scans are quieter and patching is less frequent.

---

## Variants

Chiseled images come in flavors matching your publish mode ([01-PublishModes.md](01-PublishModes.md)):

```dockerfile
# Standard runtime (framework-dependent app):
FROM mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled

# Extra-trimmed for AOT/self-contained (no .NET runtime libs — your app brings them):
FROM mcr.microsoft.com/dotnet/runtime-deps:10.0-noble-chiseled
```

- **`aspnet`/`runtime` chiseled** — for framework-dependent apps (includes the .NET runtime).
- **`runtime-deps` chiseled** — only the native OS dependencies, **no .NET runtime** — for **self-contained / Native AOT** apps that bundle their own runtime ([06-NativeAOT.md](06-NativeAOT.md)). This pairs perfectly with AOT for the smallest possible production image.
- **`-extra`** variants add a few common native libs (e.g., for globalization/ICU) when the base chiseled image is *too* minimal for your app's needs.

---

## Non-root by default

Chiseled images run as a **non-root** user (exposed via the `$APP_UID` environment variable) out of the box. Running as non-root is a security best practice — if the container is compromised, the attacker doesn't have root inside it, limiting what they can do and how they can escalate. With standard images you'd manually add a `USER`; chiseled does it for you. (Non-root also means you can't bind privileged ports <1024 — modern .NET listens on 8080, which is fine.)

---

## Trade-offs and when not to use

The minimalism has costs:

- **No shell** means you **can't `exec` into the container** to debug interactively (no `bash`). Debug via logs/telemetry ([Ch12](../12-Observability/README.md)), the dashboard, or ephemeral debug containers (`kubectl debug`) instead — a mindset shift but generally healthy (production containers shouldn't be poked at by hand).
- **Missing dependencies**: an app needing extra native libs (certain globalization/ICU scenarios, some native packages) may need the `-extra` variant or a fuller base. Test your app on the chiseled image.
- **Tooling expectations**: scripts/health checks that shell out (`CMD curl ...`) won't work without a shell — use HTTP health probes the orchestrator runs externally, not in-container shell commands.

When a dependency genuinely needs a fuller userland (and `-extra` doesn't cover it), fall back to a standard Debian/Ubuntu or Alpine image. But for the **majority** of server apps, chiseled is the better production default.

---

## Common gotchas

### Expecting a shell

There's no `bash`/`sh` — `docker exec -it ... bash` fails, and shell-based health checks/entrypoint scripts won't run. Use external HTTP probes and logs/telemetry for debugging; use `kubectl debug` ephemeral containers if you must.

### Missing native libraries

A too-minimal base can lack libs your app needs (e.g., ICU for globalization). Use the **`-extra`** variant or set `InvariantGlobalization` if appropriate — test the app on the chiseled image.

### Wrong chiseled variant for the publish mode

Use **`runtime-deps` chiseled** for self-contained/AOT (no runtime included) and **`aspnet`/`runtime` chiseled** for framework-dependent. Mismatching means a missing or duplicated runtime.

### Trying to install packages at runtime

No package manager — you can't `apt install` inside. Bake any needed native deps into the image at build time (or choose a base that includes them).

---

## Summary

- **Chiseled images** are Microsoft's ultra-minimal, "distroless"-style .NET base images (Ubuntu Chiseled): **no shell, no package manager, minimal libraries, non-root by default** — yielding a **much smaller image**, **smaller attack surface**, and **far fewer CVEs**; the recommended production base ([02-Docker.md](02-Docker.md)).
- Variants match the publish mode: **`aspnet`/`runtime` chiseled** (framework-dependent, includes runtime), **`runtime-deps` chiseled** (self-contained/**AOT** — no runtime, smallest possible — [06-NativeAOT.md](06-NativeAOT.md)), and **`-extra`** (adds common native libs like ICU).
- They run as **non-root** (`$APP_UID`) for reduced blast radius (can't bind privileged ports — use 8080).
- Trade-offs: **no shell** (debug via logs/telemetry/`kubectl debug`, use external HTTP health probes) and possible **missing native libs** (use `-extra`/a fuller base, test your app) — but for most server apps, chiseled is the better default.

→ Next: [04-Kubernetes.md](04-Kubernetes.md)
