# Chapter 19 — Deployment — Q & A

---

### Q1. Framework-dependent vs self-contained?

**Framework-dependent (FDD)** needs the .NET runtime on the target (small; runtime patched by host) — ideal in controlled environments/base images. **Self-contained (SCD)** bundles the runtime (larger, runs anywhere for a RID, you ship runtime updates) — for machines without .NET or to pin an exact version.

---

### Q2. What is a RID and when is it required?

A **Runtime Identifier** (`linux-x64`, `win-x64`, `linux-arm64`, …) names the target platform. It's required for self-contained, single-file, trimming, and AOT (which need to know which runtime/native code to produce). Framework-dependent **portable** builds don't need one.

---

### Q3. What does single-file publish do?

`PublishSingleFile=true` packs the app (and, if self-contained, the runtime) into one executable for easy distribution. It's a packaging convenience; some native libs may sit alongside and certain assembly-loading patterns differ — test the output.

---

### Q4. What does trimming do and what's the danger?

It removes statically-unreachable code to shrink self-contained output. The danger: **reflection is invisible to static analysis**, so reflection-reached code gets trimmed and **fails at runtime** (not in dev). Heed trim warnings; annotate or use source generators.

---

### Q5. What's the difference between Native AOT and ReadyToRun?

**Native AOT** compiles fully to native (no JIT, no runtime needed) — fastest startup/smallest/lowest memory, but heavy **restrictions** (no dynamic codegen, limited reflection, no UI). **ReadyToRun** precompiles IL to native *alongside* the IL — faster startup with **full JIT/reflection/UI** support and **no compatibility cost** (but larger artifact).

---

### Q6. When choose R2R over AOT?

When you want faster startup but need reflection/dynamic/JIT/UI or broad library compatibility — reflection-heavy services, MVC, WPF/WinUI, large legacy apps. R2R requires no code changes; AOT requires meeting its constraints.

---

### Q7. Why use a multi-stage Dockerfile?

To build with the large **SDK** image but ship the small **runtime** image — copying only the published output into the final stage. The shipped image has your app + runtime, not the compilers/SDK — much smaller and more secure.

---

### Q8. Why copy the csproj and `restore` before the source in a Dockerfile?

Docker caches layers; copying only the project file and running `restore` first means the (slow) NuGet restore layer is **cached** and only re-runs when dependencies change — not on every source edit. Order steps least- to most-frequently-changing.

---

### Q9. What are Chiseled images and why use them?

Microsoft's ultra-minimal "distroless"-style base images: **no shell, no package manager, minimal libs, non-root by default** — smaller image, smaller attack surface, far fewer CVEs. The production recommendation. `runtime-deps` chiseled pairs with self-contained/AOT (no runtime included).

---

### Q10. What's the downside of Chiseled images?

**No shell** — you can't `exec` in to debug interactively (use logs/telemetry/`kubectl debug`), and shell-based health checks/scripts won't run. Also possible **missing native libs** (use the `-extra` variant). Fine for most server apps.

---

### Q11. How does your app map to Kubernetes objects?

**Deployment** (replicas + Pod template, self-healing, rolling updates), **Service** (stable load-balanced endpoint), **ConfigMap/Secret** (config injection as env/files), **Ingress** (external HTTP routing). You declare a Deployment; K8s reconciles reality to match.

---

### Q12. Liveness vs readiness probes — and how to map them?

**Liveness** (`/alive`): "is the process healthy?" — failing **restarts** the container; map to a **cheap** check. **Readiness** (`/health`): "can it serve now?" — failing **stops routing** (no restart); map to **dependency** health. A liveness check depending on the DB causes pointless restart loops.

---

### Q13. How does config/secrets reach a .NET app in Kubernetes?

K8s injects ConfigMaps/Secrets as **environment variables** (use `__` for nested keys); .NET reads them via the env config provider — no K8s-specific code. K8s Secrets are only **base64** (not encrypted by default) — add encryption-at-rest or an external store.

---

### Q14. What does graceful shutdown require?

K8s sends **SIGTERM** then SIGKILL after a grace period. The .NET Generic Host triggers graceful shutdown on SIGTERM (drains in-flight requests via `ApplicationStopping`). Your long operations must respect **cancellation** to finish/stop within the grace period; set `terminationGracePeriodSeconds` adequately.

---

### Q15. What problem does Helm solve?

Duplicated Kubernetes YAML across environments. Helm **templates** manifests into a **chart** with **values** overridden per environment (one chart, many values files), and manages installs/upgrades/**rollbacks** as versioned **releases**.

---

### Q16. How does Helm enable rollback?

It tracks each install/upgrade as a **release revision**; `helm rollback <release> <revision>` reverts to a previous one — a fast, reliable undo that raw `kubectl apply` doesn't track. `helm history` shows revisions.

---

### Q17. What does Native AOT give up?

Runtime code generation (`Reflection.Emit`, `Expression.Compile` to IL), much reflection (trim-limited), dynamic assembly loading/plugins, the `dynamic` keyword, and most UI frameworks (WPF/WinUI). It implies trimming, inheriting trim constraints.

---

### Q18. How do you make code AOT/trim-safe?

Prefer **source generators** that emit statically-analyzable code (System.Text.Json source gen, configuration binding gen, Minimal API RDG) instead of runtime reflection; use `[DynamicallyAccessedMembers]`/`[DynamicDependency]` to preserve necessary reflection; resolve all trim/AOT warnings; and test the published artifact.

---

### Q19. What are CI and CD?

**CI** (Continuous Integration) builds and tests every change for fast feedback. **CD** (Continuous Delivery/Deployment) takes a passing build, packages it (container → registry), and ships it to environments — Delivery gates production behind approval, Deployment ships straight through.

---

### Q20. What's the canonical .NET pipeline?

restore → build → test → publish/containerize → push → deploy, where each stage is a **gate** (failures stop the pipeline). CI runs build+tests (+coverage, analyzers, vulnerability scans) on every push/PR; CD tags/pushes the image and deploys via Helm/`azd`/`kubectl`/App Service.

---

### Q21. Why never deploy `:latest` image tags?

`latest` makes releases non-reproducible (the tag's content changes) and breaks rollback. Tag images with the **commit SHA or semantic version** and deploy that specific tag.

---

### Q22. Compare rolling, blue/green, and canary deploys.

**Rolling**: gradually replace old instances with new (zero-downtime, K8s default). **Blue/green**: run new alongside old, switch all traffic at once (instant rollback by switching back). **Canary**: route a small % to the new version, watch metrics, then ramp up (limited blast radius). The safer ones rely on observability.

---

### Q23. What is Azure App Service and when use it?

Managed PaaS for web apps/APIs — Azure handles OS/runtime/scaling/TLS; deploy via zip/container/CI-CD, no Dockerfile needed. Use it for a **single web app** (vs Aspire + Container Apps/Kubernetes for multi-service systems). Standout: **deployment slots** for zero-downtime swaps and instant rollback.

---

### Q24. How do deployment slots work?

Separate live instances (e.g., `staging`) you deploy to, **warm up**, verify, then **swap** into production (near-instant, warmed so no cold start) — blue/green without Kubernetes; **swap back** to roll back. Slot settings can stay with the slot or swap.

---

### Q25. How should secrets be handled across deployment (containers/K8s/App Service/CI)?

Never in the repo/image/plaintext config. Inject at runtime: env vars from a secret store; **K8s Secrets** (with encryption-at-rest or external store), **App Service Key Vault references + managed identity**, **CI secret stores/OIDC**. Use managed identity to avoid storing the credential that unlocks the vault ([Ch13 §07](../13-Configuration/07-Secrets.md)).
