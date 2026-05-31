# Kubernetes

## Orchestrating containers at scale

Once your app is a container ([02-Docker.md](02-Docker.md)), **Kubernetes (K8s)** is the dominant platform for running it in production at scale: it schedules containers across machines, restarts failed ones, scales replicas up/down, load-balances traffic, manages configuration/secrets, and performs rolling updates. This isn't a Kubernetes deep-dive — it's the **.NET-relevant essentials**: how your app maps to K8s objects (Deployment, Service, ConfigMap, Secret) and how the runtime concerns you've built (health probes, config, graceful shutdown) plug into the platform.

```yaml
# A minimal Deployment + Service for an ASP.NET Core API
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
          image: myregistry/orderapi:1.4.2
          ports: [ { containerPort: 8080 } ]
          livenessProbe:  { httpGet: { path: /alive,  port: 8080 } }
          readinessProbe: { httpGet: { path: /health, port: 8080 } }
```

---

## The core objects

| Object | Role |
|---|---|
| **Pod** | the smallest unit — one (or more) running containers |
| **Deployment** | manages a set of identical Pods: replicas, rolling updates, self-healing |
| **Service** | a stable network endpoint load-balancing across a Deployment's Pods |
| **ConfigMap** | non-secret configuration, injected as env vars or files |
| **Secret** | sensitive configuration (base64-encoded), injected like ConfigMaps |
| **Ingress** | HTTP routing from outside the cluster to Services |

You rarely create Pods directly — you declare a **Deployment** (desired replica count + the Pod template), and K8s continuously reconciles reality to match (restarting crashed Pods, replacing them on node failure). A **Service** gives those Pods a stable name/IP so other services reach them (the platform-level analog of service discovery — [Ch18 §04](../18-Aspire/04-ServiceDiscovery.md)).

---

## Probes — liveness and readiness

This is where your app's health checks ([Ch04 §16](../04-AspNetCore/16-HealthChecks.md)) meet the platform. K8s uses two probes, and the distinction matters:

```yaml
livenessProbe:  { httpGet: { path: /alive,  port: 8080 }, periodSeconds: 10 }
readinessProbe: { httpGet: { path: /health, port: 8080 }, periodSeconds: 5 }
startupProbe:   { httpGet: { path: /alive,  port: 8080 }, failureThreshold: 30 }
```

- **Liveness** (`/alive`) — "is the process healthy?" If it fails, K8s **restarts** the container. Map it to a *cheap* check (process is up); a liveness check that depends on a downed database would cause needless restart loops.
- **Readiness** (`/health`) — "can it serve traffic *right now*?" If it fails, K8s **stops routing** to the Pod (but doesn't restart it). Map it to dependency health — when the DB is unreachable, stop taking traffic until it recovers.
- **Startup** — gives slow-starting apps time to boot before liveness kicks in (avoids killing a still-initializing app).

Wiring these to ASP.NET Core's `/alive` (liveness) and `/health` (readiness) endpoints (which Aspire's `MapDefaultEndpoints` provides — [Ch18 §03](../18-Aspire/03-ServiceDefaults.md)) lets K8s manage your app correctly.

---

## Configuration and secrets

K8s injects configuration into containers, which .NET reads via the environment-variable provider ([Ch13 §02](../13-Configuration/02-Providers.md)):

```yaml
envFrom:
  - configMapRef: { name: orderapi-config }
  - secretRef:    { name: orderapi-secrets }
env:
  - name: ConnectionStrings__Db
    valueFrom: { secretKeyRef: { name: orderapi-secrets, key: db-connection } }
```

- **ConfigMaps** hold non-secret settings (feature flags, URLs); **Secrets** hold sensitive values (connection strings, API keys — [Ch13 §07](../13-Configuration/07-Secrets.md)).
- Use `__` (double underscore) in env var names to express nested config keys (`ConnectionStrings__Db` → `ConnectionStrings:Db`) ([Ch13 §03](../13-Configuration/03-Layering.md)).
- K8s Secrets are only **base64-encoded** (not encrypted by default) — enable encryption-at-rest and/or use an external secret store (Key Vault via CSI, sealed secrets) for real protection.

This is why .NET apps need **no Kubernetes-specific config code** — they read env vars; K8s supplies them from ConfigMaps/Secrets.

---

## Rolling updates and scaling

- **Rolling update**: change the Deployment's image tag and K8s **gradually** replaces old Pods with new ones (respecting readiness probes so traffic only goes to ready Pods) — zero-downtime deploys. It can **roll back** if the new version fails.
- **Scaling**: `replicas: N` or a **HorizontalPodAutoscaler** scales Pods by CPU/memory/custom metrics. Your app should be **stateless** (or externalize state to a DB/cache/distributed cache — [Ch06](../06-DataAndCaching/README.md)) so any replica can handle any request — a key design constraint for K8s.

---

## Graceful shutdown

On update/scale-down, K8s sends **SIGTERM**, waits a grace period, then **SIGKILL**. The .NET Generic Host ([Ch03 §01](../03-HostingAndDI/01-GenericHost.md)) handles SIGTERM by triggering graceful shutdown — stopping new work and draining in-flight requests via the `ApplicationStopping` token. Ensure long operations respect **cancellation** so they finish or stop cleanly within the grace period; otherwise K8s kills them mid-flight. Set `terminationGracePeriodSeconds` to accommodate your drain time.

---

## Common gotchas

### Liveness probe that depends on dependencies

A liveness check failing because the database is down causes K8s to **restart** the app pointlessly (restarting won't fix the DB). Liveness = cheap "process up"; put dependency checks in **readiness** (stop routing, don't restart).

### Stateful app + scaling

In-memory state (sessions, caches) breaks when requests hit different replicas. Externalize state (distributed cache/DB — [Ch06](../06-DataAndCaching/README.md)); design stateless services for horizontal scaling.

### Secrets assumed encrypted

K8s Secrets are base64, not encrypted by default. Enable encryption-at-rest or use an external secret store (Key Vault CSI driver) for sensitive data ([Ch13 §07](../13-Configuration/07-Secrets.md)).

### Ignoring graceful shutdown

Not honoring SIGTERM/cancellation means in-flight requests are killed on deploy/scale-down. Respect the host's shutdown and cancellation tokens; set an adequate grace period.

### Missing/incorrect resource limits

Without CPU/memory requests/limits, Pods can be scheduled poorly or OOM-killed unpredictably. Set `resources.requests`/`limits` based on profiling ([Ch21 Performance](../21-Performance/README.md)).

---

## Summary

- **Kubernetes** runs your containers at scale — scheduling, self-healing, scaling, load-balancing, rolling updates. Your app maps to **Deployment** (replicas + Pod template, self-healing), **Service** (stable load-balanced endpoint), **ConfigMap/Secret** (config injection), and **Ingress** (external routing).
- **Probes** connect to your health checks: **liveness** (`/alive`, cheap — failing **restarts**), **readiness** (`/health`, dependency-aware — failing **stops routing**), **startup** (grace for slow boot) — wired to ASP.NET Core's endpoints ([Ch04 §16](../04-AspNetCore/16-HealthChecks.md)).
- **Config/secrets** are injected as env vars (use `__` for nested keys); .NET reads them via the env provider — no K8s-specific code. K8s **Secrets are base64, not encrypted** — add encryption/an external store.
- **Rolling updates** give zero-downtime deploys (gated by readiness, with rollback); **scaling** requires **stateless** apps (externalize state); honor **SIGTERM/cancellation** for graceful shutdown and set **resource limits**.

→ Next: [05-Helm.md](05-Helm.md)
