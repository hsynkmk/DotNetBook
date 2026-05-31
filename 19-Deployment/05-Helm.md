# Helm

## Packaging Kubernetes manifests

Raw Kubernetes YAML ([04-Kubernetes.md](04-Kubernetes.md)) has a problem at scale: you end up with near-identical manifests duplicated per environment (dev/staging/prod), differing only in a few values (image tag, replica count, connection strings). **Helm** is the package manager for Kubernetes — it templates your manifests into a reusable **chart** with **values** you override per environment, and manages installs/upgrades/rollbacks as versioned **releases**. For a .NET app, a Helm chart turns "edit five YAML files for each environment" into "one chart, three values files."

```bash
helm install orderapi ./orderapi-chart -f values.prod.yaml
helm upgrade orderapi ./orderapi-chart --set image.tag=1.4.3
helm rollback orderapi 2          # roll back to a previous release revision
```

---

## What a chart contains

A Helm chart is a directory of templated manifests plus metadata and default values:

```
orderapi-chart/
├── Chart.yaml            # chart name, version, app version
├── values.yaml           # default values (overridable)
├── templates/
│   ├── deployment.yaml   # templated Deployment
│   ├── service.yaml      # templated Service
│   ├── configmap.yaml
│   ├── ingress.yaml
│   └── _helpers.tpl      # reusable template snippets
└── charts/               # dependency (sub-)charts
```

The `templates/` files are Kubernetes YAML with **placeholders**; `values.yaml` supplies defaults; you override values per environment at install/upgrade time.

---

## Templating with values

Templates reference values via `{{ .Values.x }}`, so one template serves all environments — only the values differ:

```yaml
# templates/deployment.yaml
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: {{ .Values.environment }}
```

```yaml
# values.yaml (defaults)
replicaCount: 2
image: { repository: myregistry/orderapi, tag: latest }
environment: Development
```

```yaml
# values.prod.yaml (overrides)
replicaCount: 5
image: { tag: "1.4.2" }
environment: Production
```

```bash
helm upgrade --install orderapi ./orderapi-chart -f values.prod.yaml
```

The same chart deploys to dev (2 replicas, latest) and prod (5 replicas, pinned tag) — **no duplicated manifests**. This is Helm's core value: parameterize once, instantiate per environment.

---

## Releases, upgrades, and rollbacks

Helm tracks each install/upgrade as a **release revision**, giving you lifecycle management:

- **`helm install`** — create a release; **`helm upgrade`** — apply changes as a new revision.
- **`helm rollback <release> <revision>`** — revert to a previous revision if a deploy goes wrong — a fast, reliable undo (something raw `kubectl apply` doesn't track).
- **`helm history`** — see all revisions; **`helm uninstall`** — remove the release and its resources.

This versioned release model makes deploys auditable and reversible — important for production change management.

---

## Helm in a .NET workflow

A typical CI/CD flow ([09-CICD.md](09-CICD.md)) for a .NET service:

1. Build & test the app, build the container image, push it to a registry (tagged with the build/version).
2. `helm upgrade --install <release> ./chart --set image.tag=<version> -f values.<env>.yaml`.
3. Helm renders the manifests with the new image tag and applies them; K8s performs a rolling update ([04-Kubernetes.md](04-Kubernetes.md)).

The chart lives in your repo alongside the app (or in a chart repository); the pipeline passes the freshly-built image tag and the environment's values. This decouples *what to deploy* (image tag) from *how it's configured* (values per env).

---

## Alternatives and notes

- **Kustomize** is a template-free alternative (overlay-based patching of base manifests) built into `kubectl` — some teams prefer it for its simplicity (no templating language). Helm vs Kustomize is a style choice; both solve per-environment variation.
- **Aspir8** ([Ch18 §08](../18-Aspire/08-Deployment.md)) can generate Kubernetes manifests/Helm from a .NET Aspire app model — so Aspire users may get charts generated rather than hand-writing them.
- Keep **secrets out of values files** committed to git ([Ch13 §07](../13-Configuration/07-Secrets.md)) — use sealed secrets, an external secret store (Key Vault CSI), or inject at deploy time, not plaintext in `values.prod.yaml`.

---

## Common gotchas

### Secrets in committed values files

Putting real secrets in `values.prod.yaml` in git leaks them. Keep secrets in an external store / sealed secrets and reference them; never commit plaintext secrets.

### `latest` image tags

Deploying `image.tag: latest` makes releases non-reproducible and breaks rollback (the tag's content changes). **Pin a specific version** (`1.4.2`) per release.

### Over-templating

Templating every conceivable field makes charts unreadable. Template the values that actually vary per environment; keep the rest concrete.

### Forgetting `--install` on upgrade

`helm upgrade <release>` fails if the release doesn't exist. Use `helm upgrade --install` to create-or-update idempotently in CI.

### Drift from manual `kubectl` edits

Editing resources with `kubectl` outside Helm causes drift (Helm doesn't know about the change). Manage Helm-deployed resources **through Helm**, not ad-hoc `kubectl edit`.

---

## Summary

- **Helm** is Kubernetes' package manager: it **templates** manifests into a reusable **chart** with **values** overridden per environment, eliminating duplicated YAML across dev/staging/prod.
- A **chart** = `Chart.yaml` (metadata) + `values.yaml` (defaults) + `templates/` (manifests with `{{ .Values.x }}` placeholders); you override values via `-f values.<env>.yaml` or `--set key=value`.
- Helm tracks installs as versioned **releases** — **`upgrade --install`**, **`rollback`**, **`history`** — giving auditable, reversible deploys (a fast undo raw `kubectl` lacks).
- In .NET CI/CD: build/push the image, then `helm upgrade --install` with the new **pinned** image tag and the env's values — decoupling *what* (image) from *how configured* (values). Keep **secrets out of committed values**; **Kustomize** is a template-free alternative, and **Aspir8** can generate charts from Aspire ([Ch18](../18-Aspire/README.md)).

→ Next: [06-NativeAOT.md](06-NativeAOT.md)
