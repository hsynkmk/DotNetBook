# Chapter 19 — Deployment — Coding Problems

Containerize an ASP.NET Core app, publish as AOT, and deploy to Kubernetes. Each problem has a hidden solution — attempt it first.

---

### Problem 1 — Publish framework-dependent vs self-contained

Show the commands for (a) a framework-dependent publish and (b) a self-contained single-file publish for Linux x64.

<details>
<summary>Solution</summary>

```bash
# (a) Framework-dependent (needs .NET on the target):
dotnet publish -c Release

# (b) Self-contained, single-file (runs without .NET installed):
dotnet publish -c Release -r linux-x64 --self-contained -p:PublishSingleFile=true
```

FDD is smaller (controlled environments); SCD bundles the runtime (needs a RID). Single-file packs it into one executable ([01-PublishModes.md](01-PublishModes.md)).
</details>

---

### Problem 2 — Write a multi-stage Dockerfile

Containerize an ASP.NET Core API with proper layer caching and a small runtime base.

<details>
<summary>Solution</summary>

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY ["MyApi.csproj", "./"]
RUN dotnet restore                                   # cached unless csproj changes
COPY . .
RUN dotnet publish -c Release -o /app --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled AS final
WORKDIR /app
COPY --from=build /app .
USER $APP_UID                                        # non-root
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

Build on the SDK image, run on a small **chiseled** runtime image; copy csproj + `restore` before the source for cache reuse ([02-Docker.md](02-Docker.md), [03-ChiseledImages.md](03-ChiseledImages.md)).
</details>

---

### Problem 3 — Dockerfile-less container build

Produce a container image without a Dockerfile, using a chiseled base.

<details>
<summary>Solution</summary>

```xml
<!-- .csproj -->
<PropertyGroup>
  <EnableSdkContainerSupport>true</EnableSdkContainerSupport>
  <ContainerBaseImage>mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled</ContainerBaseImage>
</PropertyGroup>
```
```bash
dotnet publish -c Release --os linux --arch x64 /t:PublishContainer
```

The SDK builds an OCI image directly (non-root, multi-layer) — no Dockerfile to maintain ([02-Docker.md](02-Docker.md)).
</details>

---

### Problem 4 — Publish as Native AOT

Compile a Minimal API to a native binary and note what you must avoid.

<details>
<summary>Solution</summary>

```xml
<PropertyGroup>
  <PublishAot>true</PublishAot>
</PropertyGroup>
```
```bash
dotnet publish -c Release -r linux-x64
```

Avoid `Reflection.Emit`, `Expression.Compile`, dynamic loading, and reflection-based serialization — use **System.Text.Json source generation** and resolve all AOT/trim warnings. Gains: fast startup, low memory, no runtime needed ([06-NativeAOT.md](06-NativeAOT.md)).
</details>

---

### Problem 5 — Make JSON serialization AOT/trim-safe

Replace reflection-based `JsonSerializer` use with source generation.

<details>
<summary>Solution</summary>

```csharp
[JsonSerializable(typeof(Order))]
[JsonSerializable(typeof(List<Order>))]
internal partial class AppJsonContext : JsonSerializerContext { }

// Use the generated context — no runtime reflection:
var json = JsonSerializer.Serialize(order, AppJsonContext.Default.Order);
var order = JsonSerializer.Deserialize("...", AppJsonContext.Default.Order);
```

The source generator emits serializers at compile time, so the trimmer/AOT can see the references — trim/AOT-safe, no reflection ([07-Trimming.md](07-Trimming.md), [Ch02 §05](../02-BCL/05-Serialization.md)).
</details>

---

### Problem 6 — Choose a publish mode

For each, pick FDD/SCD + (AOT/R2R/none) and justify: (a) an Azure Function with cold-start sensitivity, (b) a WPF desktop app, (c) an API in a container with a .NET runtime base image.

<details>
<summary>Solution</summary>

- **(a) Azure Function → self-contained + Native AOT**: cold start is dominated by startup; AOT gives near-instant launch and low memory (if the code is AOT-safe) ([06-NativeAOT.md](06-NativeAOT.md)).
- **(b) WPF app → framework-dependent/self-contained + ReadyToRun**: AOT isn't supported for WPF; R2R speeds launch with full compatibility ([Ch16 §07](../16-Desktop/07-Packaging.md), [08-ReadyToRun.md](08-ReadyToRun.md)).
- **(c) Containerized API → framework-dependent (on a runtime base image)**, optionally R2R: the base image supplies the runtime (small layers, runtime patched by the image); R2R if startup matters ([01-PublishModes.md](01-PublishModes.md)).
</details>

---

### Problem 7 — Kubernetes Deployment + Service

Write a Deployment (3 replicas) and Service for the containerized API, with probes.

<details>
<summary>Solution</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: orderapi }
spec:
  replicas: 3
  selector: { matchLabels: { app: orderapi } }
  template:
    metadata: { labels: { app: orderapi } }
    spec:
      containers:
        - name: orderapi
          image: myregistry/orderapi:1.4.2     # pinned, not :latest
          ports: [ { containerPort: 8080 } ]
          livenessProbe:  { httpGet: { path: /alive,  port: 8080 } }
          readinessProbe: { httpGet: { path: /health, port: 8080 } }
          resources:
            requests: { cpu: "100m", memory: "128Mi" }
            limits:   { cpu: "500m", memory: "256Mi" }
---
apiVersion: v1
kind: Service
metadata: { name: orderapi }
spec:
  selector: { app: orderapi }
  ports: [ { port: 80, targetPort: 8080 } ]
```

Liveness=`/alive` (restart), readiness=`/health` (route), pinned image, resource limits ([04-Kubernetes.md](04-Kubernetes.md)).
</details>

---

### Problem 8 — Inject config and secrets in K8s

Wire a connection string from a Secret and a feature flag from a ConfigMap into the Deployment.

<details>
<summary>Solution</summary>

```yaml
        env:
          - name: ConnectionStrings__Db
            valueFrom: { secretKeyRef: { name: orderapi-secrets, key: db-connection } }
          - name: Features__NewCheckout
            valueFrom: { configMapKeyRef: { name: orderapi-config, key: new-checkout } }
```

`__` maps to nested config keys (`ConnectionStrings:Db`). .NET reads these via the env provider — no K8s-specific code. (K8s Secrets are base64 — add encryption/an external store — [04-Kubernetes.md](04-Kubernetes.md), [Ch13 §07](../13-Configuration/07-Secrets.md).)
</details>

---

### Problem 9 — Helm-template the Deployment

Parameterize the image tag and replica count so one chart serves dev and prod.

<details>
<summary>Solution</summary>

```yaml
# templates/deployment.yaml
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```
```yaml
# values.yaml (defaults)         # values.prod.yaml (overrides)
replicaCount: 2                   replicaCount: 5
image: { repository: myreg/orderapi, tag: latest }   image: { tag: "1.4.2" }
```
```bash
helm upgrade --install orderapi ./chart -f values.prod.yaml
```

One template, per-environment values — no duplicated YAML ([05-Helm.md](05-Helm.md)).
</details>

---

### Problem 10 — A CI pipeline (GitHub Actions)

Build, test, and run on every PR/push to main.

<details>
<summary>Solution</summary>

```yaml
name: ci
on: { push: { branches: [main] }, pull_request: {} }
jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '10.0.x' }
      - run: dotnet restore
      - run: dotnet build -c Release --no-restore
      - run: dotnet test -c Release --no-build --collect:"XPlat Code Coverage"
```

Each stage is a gate; `--no-restore`/`--no-build` avoid redundant work. Add vulnerability scanning (`dotnet list package --vulnerable`) and analyzers ([09-CICD.md](09-CICD.md)).
</details>

---

### Problem 11 — A CD job that builds, pushes, and deploys

Extend the pipeline to containerize and deploy via Helm with a pinned tag.

<details>
<summary>Solution</summary>

```yaml
  deploy:
    needs: build-test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          IMG=myregistry.azurecr.io/orderapi:${{ github.sha }}
          docker build -t "$IMG" .
          docker push "$IMG"
      - run: helm upgrade --install orderapi ./chart --set image.tag=${{ github.sha }} -f values.prod.yaml
```

Tag with the **commit SHA** (reproducible, rollback-able), push, then `helm upgrade --install` (create-or-update). Store registry/cluster creds in **secrets**/OIDC ([09-CICD.md](09-CICD.md), [05-Helm.md](05-Helm.md)).
</details>

---

### Problem 12 — Deploy to App Service with a slot swap

Deploy a new version to a staging slot and swap it to production with zero downtime.

<details>
<summary>Solution</summary>

```bash
dotnet publish -c Release -o ./publish
az webapp deploy --resource-group rg --name myapp --slot staging --src-path ./publish --type zip
# warm up / verify staging, then swap:
az webapp deployment slot swap --resource-group rg --name myapp --slot staging --target-slot production
```

Deploy to `staging`, verify, then **swap** (near-instant, warmed — no cold start). Swap back to roll back instantly — blue/green without Kubernetes ([10-AppService.md](10-AppService.md)).
</details>

---

### Problem 13 — Spot the trimming bug

```csharp
[PublishTrimmed]  // (conceptually) trimming enabled
public object Create(string typeName) {
    var type = Type.GetType(typeName);
    return Activator.CreateInstance(type!)!;   // works in dev, throws in the trimmed build
}
```

Why does it break when trimmed, and how do you fix it?

<details>
<summary>Solution</summary>

The trimmer can't see which types `Type.GetType(typeName)` will load (it's a runtime string), so those types/constructors look unreachable and get **trimmed** — `CreateInstance` then fails (null type / missing constructor). Fixes:

- Annotate so the trimmer preserves the needed members, e.g. `[DynamicDependency(DynamicallyAccessedMemberTypes.PublicConstructors, typeof(KnownType))]`, or constrain the API with `[DynamicallyAccessedMembers]`.
- **Better**: avoid runtime reflection — use a known factory/registration or a source generator so references are static and trim-safe.

And always **test the trimmed artifact**, since this bug doesn't appear in `dotnet run` ([07-Trimming.md](07-Trimming.md)).
</details>

---

### Problem 14 — Fix a liveness probe causing restart loops

Pods restart constantly during a brief database outage. The probes are:

```yaml
livenessProbe:  { httpGet: { path: /health, port: 8080 } }   # /health checks the DB
```

What's wrong and the fix?

<details>
<summary>Solution</summary>

The **liveness** probe points at `/health`, which includes a **database** check. When the DB is briefly down, liveness fails → K8s **restarts** the pod — but restarting doesn't fix the DB, so it loops pointlessly. Fix: liveness should be a **cheap "process up" check** (`/alive`), and dependency health belongs in **readiness** (stop routing, don't restart):

```yaml
livenessProbe:  { httpGet: { path: /alive,  port: 8080 } }   # cheap, process-up
readinessProbe: { httpGet: { path: /health, port: 8080 } }   # dependency-aware
```

Now a DB outage stops routing traffic (readiness) without killing the app (liveness) ([04-Kubernetes.md](04-Kubernetes.md)).
</details>

---

### Problem 15 — Smallest possible production container for an AOT app

Combine AOT + the right chiseled base for a minimal, secure image.

<details>
<summary>Solution</summary>

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -r linux-x64 -p:PublishAot=true -o /app

# runtime-deps chiseled: no .NET runtime (the AOT binary is self-contained), no shell, non-root
FROM mcr.microsoft.com/dotnet/runtime-deps:10.0-noble-chiseled AS final
WORKDIR /app
COPY --from=build /app/MyApi .
USER $APP_UID
ENTRYPOINT ["./MyApi"]
```

AOT produces a self-contained native binary, so the final image needs only **`runtime-deps` chiseled** (native OS deps, no .NET runtime) — the smallest, lowest-CVE production image ([03-ChiseledImages.md](03-ChiseledImages.md), [06-NativeAOT.md](06-NativeAOT.md)).
</details>
