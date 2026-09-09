# 18–19 — .NET Aspire & Deployment

## ⚡ 30-second answer

**.NET Aspire** is the **cloud-native composition layer** — you describe your multi-service app's
graph in **C#** in an **AppHost**, and get service discovery, telemetry, health checks, resilience
and a live **dashboard** by convention. It is **not** a runtime and **not** a replacement for
Kubernetes; it's the glue and control panel for development, plus a deployment manifest.
**Deployment** trades four axes: framework-dependent vs self-contained, single-file, trimming, and
AOT. **ReadyToRun** is the safe middle ground — faster startup with **no compatibility cost** —
while **Native AOT** gives the fastest startup and smallest footprint but forbids runtime code
generation and limits reflection. Containerize with a **multi-stage Dockerfile** (copy the csproj
and restore *before* the source, so layer caching works), run on **chiseled** images (non-root, no
shell, minimal CVEs), and wire Kubernetes probes to your **liveness/readiness** endpoints — which
is what makes rolling updates zero-downtime.

---

## Core mechanics

### Aspire: the AppHost

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var db    = builder.AddPostgres("pg").AddDatabase("ordersdb");
var cache = builder.AddRedis("cache");

var api = builder.AddProject<Projects.Api>("orderapi")
                 .WithReference(db).WithReference(cache)
                 .WaitFor(db);

builder.AddProject<Projects.Worker>("worker").WithReference(api);

builder.Build().Run();
```

A C# project describing your whole distributed app as a **graph of resources** — the type-safe
replacement for docker-compose plus launch scripts. **`WithReference`** wires the graph: it
injects the resource's **connection string** or **endpoint URL** into the consumer, which is how
you get service discovery **with no hardcoded URLs**. `WaitFor` orders startup; `WithReplicas`,
`WithHttpEndpoint` and `WithDataVolume` handle the rest.

### Aspire: the other three pieces

- **ServiceDefaults** — a shared project scaffolded as **source you own**.
  `builder.AddServiceDefaults()` wires OpenTelemetry, health checks, service discovery and HTTP
  resilience into every service; `app.MapDefaultEndpoints()` exposes **`/health`** (readiness) and
  **`/alive`** (liveness).
- **Integrations**, in two halves: **hosting** integrations run in the AppHost and provision the
  resource (`AddPostgres` — a container locally, a managed service in production); **client**
  integrations run in the service and register the client already connected, traced, health-checked
  and resilient. **The resource name is the contract between them.**
- **The dashboard** — resource state, aggregated structured logs, a **distributed trace waterfall
  across services**, and live metrics. It's the *development* observability surface using the same
  telemetry shape you export to a real APM in production ([12](12-Observability.md)).

**Deployment**: the same app model emits a platform-neutral **manifest**. `azd up` provisions
managed Azure resources and deploys to Container Apps; the manifest also targets Kubernetes.
Production differs in **backing, not code** — local containers become managed resources, logical
names resolve to platform addresses, secret parameters map to a secret store.

**Testing**: `DistributedApplicationTestingBuilder` starts the **whole graph with real
dependencies** — Testcontainers-grade fidelity, one layer above `WebApplicationFactory`
([17](17-Testing.md)).

### Publish modes

| Mode | Startup | Size | Needs .NET installed | Reflection |
|---|---|---|---|---|
| **Framework-dependent** | baseline | smallest | ✅ yes | full |
| **Self-contained** | baseline | large | no | full |
| **+ ReadyToRun** | **fast** | larger | either | **full** |
| **+ Trimmed** | baseline | smaller | no | ⚠️ **static only** |
| **Native AOT** | **instant** | small single binary | no | ❌ **heavily restricted** |

**ReadyToRun** precompiles IL to native **alongside** the IL, so startup does less JIT work while
the runtime keeps full JIT, reflection and re-optimization. Its defining advantage is **no
compatibility cost** — it works with reflection-heavy MVC apps and WPF/WinUI.

**Native AOT** is for new, AOT-aware console and server workloads: CLIs, serverless, high-density
microservices. It **implies trimming**, forbids `Reflection.Emit`/`Expression.Compile`/`dynamic`,
and is unsupported for most UI frameworks ([01](01-Runtime-GC-JIT.md)).

**Trimming** removes statically-unreachable code. The hazard: **reflection is invisible to static
analysis**, so reflection-reached code is trimmed and fails **at runtime, in the published build
only**. Trim warnings (`IL2xxx`) predict exactly this — treat them as errors. The good fix is
**source generators** (System.Text.Json, config binding, Minimal API RDG), which emit
statically-analyzable code and need no annotations.

### Docker

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY ["Api/Api.csproj", "Api/"]          # ← csproj first…
RUN dotnet restore "Api/Api.csproj"      # ← …so this layer caches
COPY . .
RUN dotnet publish "Api/Api.csproj" -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled AS final
WORKDIR /app
COPY --from=build /app .
EXPOSE 8080                              # non-root can't bind 80
ENTRYPOINT ["dotnet", "Api.dll"]
```

**Multi-stage**: build on the SDK image, run on a small runtime image. **Copy the csproj and
restore before the full source** — otherwise every source change invalidates the restore layer and
your builds crawl.

**Chiseled images** are Microsoft's distroless-style base: **no shell, no package manager,
non-root by default**, far fewer CVEs. Use `runtime-deps` chiseled for self-contained/AOT (the
smallest possible). The trade-offs: no shell for debugging (use logs, telemetry, `kubectl debug`)
and possibly missing native libraries.

Also: non-root, env-based config with no baked secrets, health probes, graceful SIGTERM handling,
and a `.dockerignore`.

### Kubernetes

| Your app | Maps to |
|---|---|
| replicas + pod template, self-healing | **Deployment** |
| stable load-balanced endpoint | **Service** |
| external HTTP routing | **Ingress** |
| config / secrets | **ConfigMap** / **Secret** (injected as env vars) |

**Probes are where your health checks pay off**: **liveness** → `/alive` (cheap, app-only —
failing **restarts** the pod), **readiness** → `/health` (dependency-aware — failing **stops
routing**), **startup** (grace for slow boot). Nested config keys use `__` in env vars, and .NET
reads them through the environment provider with **no Kubernetes-specific code**
([03](03-Hosting-DI-Config.md)).

**Rolling updates** are gated by readiness, which is what makes them zero-downtime. Scaling
requires **stateless** apps — externalize session, cache and the Data Protection key ring
([10](10-Identity-Security.md)).

**Helm** templates the manifests into a chart with per-environment values, and tracks installs as
versioned **releases** — so `helm rollback` is a real, auditable undo.

### CI/CD and App Service

CI on every push: restore → build → test → quality gates (warnings-as-errors, integration tests,
analyzers, vulnerability scan). CD: build the image, **tag it with the version or commit SHA —
never `latest`**, push, and deploy via Helm/`azd`/`kubectl`. Secrets live in the CI secret store or
come from OIDC federation; deploy identities are least-privilege.

**Azure App Service** is the no-Kubernetes path: App Settings injected as env vars (`__` for
nesting), **Key Vault references + managed identity** for secrets, and **deployment slots** for
zero-downtime blue/green — deploy to staging, warm up, **swap**, and swap back to roll back
([20](20-Azure.md)).

---

## 🪤 Traps & gotchas

- **Copying all the source before `dotnet restore`** — every code change busts the restore layer,
  so CI re-downloads every package on every build.
- **Building on the SDK image and shipping it** — a 700 MB image with a compiler in it. Multi-stage.
- **Baking secrets into the image** — anyone who can pull the image has them, and layer history
  keeps them even if a later layer deletes the file.
- **Running as root** — the default in many hand-written Dockerfiles, and unnecessary. Chiseled
  images are non-root by default, which is also why they **can't bind port 80** — use 8080.
- **`latest` as an image tag** — you can't tell what's deployed and you can't roll back
  deterministically. Tag with the SHA or version.
- **No `.dockerignore`** — `bin/`, `obj/`, and `.git` get copied into the build context, slowing
  everything and occasionally leaking secrets.
- **Liveness probe that checks the database** — a brief DB outage fails liveness on every pod and
  Kubernetes restarts the entire fleet. Restart storm. Liveness checks the app only.
- **No readiness probe** — traffic is routed to a pod that hasn't finished starting, so every
  rolling deploy drops requests.
- **Ignoring SIGTERM** — Kubernetes sends it, waits `terminationGracePeriodSeconds` (30 by
  default), then SIGKILLs. In-flight requests and background work die mid-operation
  ([08](08-BackgroundProcessing.md)).
- **Stateful app behind a load balancer** — in-memory session, in-memory cache, and especially an
  unpersisted **Data Protection key ring**, which logs users out at random as they hit different
  pods.
- **No resource limits** — one pod's memory leak evicts its neighbours.
- **Trimming without testing the published artifact** — trim failures never appear under
  `dotnet run`. They appear in production as `MissingMethodException`.
- **Suppressing `IL2xxx` warnings** to make a trimmed build compile — you've silenced the exact
  prediction of the runtime failure you're about to ship.
- **Choosing AOT for a reflection-heavy or MVC app** — it won't work. ReadyToRun gives you most of
  the startup benefit with none of the restrictions.
- **Choosing AOT for a long-running throughput server** — you give up dynamic PGO and OSR, so peak
  throughput is usually *worse* than the JIT's ([01](01-Runtime-GC-JIT.md)).
- **Alpine without testing native dependencies** — musl instead of glibc breaks some native
  libraries in ways that only show up at runtime.
- **Kubernetes Secrets assumed to be encrypted** — they're **base64-encoded**, not encrypted. Turn
  on encryption at rest or use an external store.
- **Auto-migrating the database on startup across replicas** — they race. A controlled deploy step
  ([05](05-EFCore.md)).
- **Confusing Aspire's two integration halves** — `AddPostgres` in the AppHost provisions;
  `AddNpgsqlDbContext` in the service consumes. Mismatch the resource names and nothing connects,
  with a confusing error.
- **Expecting Aspire to run in production as an orchestrator** — it's a development orchestrator
  and a deployment *model*. Production is Container Apps, Kubernetes, or App Service.
- **Committing secrets into Helm `values.yaml`** — it's in the repo and in the chart.

---

## ❓ Likely questions

**Q: What is .NET Aspire, in one sentence — and what is it not?**
A: A cloud-native composition layer: you describe your app's services and backing resources as a
C# graph in an AppHost and get service discovery, telemetry, health checks, resilience and a
dashboard by convention. It is **not** a runtime, not a production orchestrator, and not a
replacement for Kubernetes — it's the glue and the control panel.

**Q: What does `WithReference` actually do?**
A: It records the dependency in the app model and injects the referenced resource's connection
string or endpoint URL into the consumer's configuration. That's the mechanism behind service
discovery — your code calls `http://orderapi` and the real address is resolved from injected
config, so the same code works locally and in production.

**Q: What's in ServiceDefaults?**
A: OpenTelemetry (traces, metrics, logs via OTLP), health check endpoints, service discovery, and
standard HTTP resilience — wired into every service by one call. It's scaffolded as **source you
own**, so it's also the right place for organization-wide defaults.

**Q: Hosting integration vs client integration?**
A: The hosting integration lives in the AppHost and provisions or orchestrates the resource — a
container locally, a managed service in production. The client integration lives in the service
and registers the client already connected, traced, health-checked and resilient. The resource
name ties them together.

**Q: Framework-dependent or self-contained?**
A: Framework-dependent is small and needs .NET installed — right for controlled environments and
container base images that already have the runtime. Self-contained bundles the runtime so it runs
anywhere for a given RID, at the cost of size and taking on runtime patching yourself.

**Q: ReadyToRun vs Native AOT?**
A: R2R precompiles IL to native alongside the IL, so startup does less JIT work but the runtime
keeps full JIT, reflection and re-optimization — **no compatibility cost**. AOT compiles
everything ahead of time with no JIT at all: instant startup, small binary, but no runtime code
generation, limited reflection, and no UI frameworks. R2R when you can't accept AOT's
restrictions.

**Q: Why does trimming break reflection?**
A: The trimmer removes what static analysis can't prove is reachable, and a reflective lookup by
string name is invisible to that analysis. So the type is trimmed and you get a
`MissingMethodException` at runtime — in the published build only, never under `dotnet run`.
`IL2xxx` warnings predict it; source generators are the real fix.

**Q: Why copy the csproj before the rest of the source in a Dockerfile?**
A: Docker caches layers. If you copy everything first, any source change invalidates the `restore`
layer and every build re-downloads all packages. Copying just the project files and restoring
first means restore is cached until dependencies actually change.

**Q: What are chiseled images and what's the trade-off?**
A: Microsoft's distroless-style base images — no shell, no package manager, minimal libraries,
non-root by default. You get a much smaller image and a far smaller attack surface. The trade-off
is you can't `exec` into a shell to debug, so you rely on logs, telemetry and ephemeral debug
containers; and some native dependencies may be missing.

**Q: How do ASP.NET Core health checks map to Kubernetes probes?**
A: Liveness → `/alive`: cheap, app-only, and failing it **restarts** the pod. Readiness →
`/health`: checks dependencies, and failing it **stops traffic** being routed. Startup probe gives
a slow-booting app grace before liveness applies. Getting liveness and readiness the wrong way
round causes restart storms.

**Q: How do you get a zero-downtime deploy?**
A: Rolling update gated by the readiness probe, plus graceful shutdown: on SIGTERM the app fails
readiness first so no new traffic arrives, drains in-flight requests, then exits. On App Service,
the equivalent is deployment slots — deploy to staging, warm it, and swap.

**Q: How does .NET configuration work in Kubernetes?**
A: ConfigMaps and Secrets are injected as environment variables, and the environment configuration
provider reads them — nested keys use `__` as the separator. No Kubernetes-specific code. Note
that Kubernetes Secrets are base64-encoded, not encrypted, so add encryption at rest or an
external store.

**Q: What does Helm add over raw `kubectl`?**
A: Templating with per-environment values instead of duplicated YAML, and versioned releases —
`upgrade --install`, `history`, and a real `rollback`, which raw manifests don't give you.

**Q: What belongs in a .NET CI pipeline?**
A: Restore, build with warnings-as-errors, the test suite including integration tests where Docker
is available, analyzers, coverage, and a dependency vulnerability scan. Then package: build the
image, tag it with the commit SHA, push. Secrets come from the CI store or OIDC, never the repo.

---

## 🎓 Senior Extra

- **Aspire's real value is the inner loop.** One `F5` starts eight services, a database, a broker
  and a cache, all wired, with a trace waterfall across them. The manifest and deployment story
  matter, but the reason teams adopt it is that onboarding drops from a day to a command.
- **Aspire is incrementally adoptable** — add an AppHost to an existing solution and reference one
  project. You don't have to convert everything, which is what makes it a realistic proposal in an
  interview.
- **Startup time is a cost function of your platform.** On Kubernetes with long-lived pods it barely
  matters; on serverless or a scale-to-zero Container App, cold start is user-visible latency — and
  that's the argument that decides R2R versus AOT.
- **AOT's throughput trade is the part people miss**: no dynamic PGO, no OSR, no tiered
  re-optimization. For a service running for days, the JIT wins on steady-state throughput.
- **`InvariantGlobalization` plus `runtime-deps` chiseled plus AOT** is the recipe for a genuinely
  tiny container — but each of those three narrows what your app can do, so decide them
  deliberately, not as a checklist ([02](02-BCL-Essentials.md)).
- **`terminationGracePeriodSeconds` and .NET's `ShutdownTimeout` must agree.** Both default to 30
  seconds, so a worker that drains for 30 gets SIGKILLed exactly as it finishes. Set them
  explicitly and make the work resumable anyway.
- **Externalize everything a second replica can't share**: session state, in-memory cache, uploaded
  files, and the **Data Protection key ring** — the last one being the most commonly forgotten and
  the most confusing to diagnose ([10](10-Identity-Security.md)).
- **Supply chain is part of deployment now**: SBOM generation, image signing, pinned base image
  digests, and a vulnerability gate in CI. It's increasingly a question asked at senior level.
- **Deployment slots are not a staging environment.** They share the App Service plan's resources,
  so a load test against staging degrades production — useful for warm-up and swap, not for
  isolation.
- **The manifest is the interesting part of Aspire's deployment story**, because it decouples the
  app model from the target. The same graph publishes to Container Apps or Kubernetes; what changes
  is the publisher, not your code.

→ Deeper: [`../18-Aspire/`](../18-Aspire/README.md) ·
[`../19-Deployment/`](../19-Deployment/README.md)
