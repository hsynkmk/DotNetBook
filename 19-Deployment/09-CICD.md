# CI/CD

## Automating build → test → ship

**CI/CD** automates the path from a commit to a deployed app. **Continuous Integration (CI)** builds and tests every change so problems surface immediately; **Continuous Delivery/Deployment (CD)** takes a passing build and packages/ships it (to a registry, then to an environment). For .NET, the pipeline is a sequence of `dotnet` commands plus container build/push and a deploy step — expressed in **GitHub Actions**, **Azure Pipelines**, GitLab CI, etc. The shape is the same everywhere: **restore → build → test → publish/containerize → push → deploy**, with quality gates that stop a bad change before it ships.

```yaml
# GitHub Actions — build, test, and containerize a .NET app
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
      - run: dotnet test -c Release --no-build --logger trx --collect:"XPlat Code Coverage"
```

---

## The canonical .NET pipeline

| Stage | Command / action | Purpose |
|---|---|---|
| **Restore** | `dotnet restore` | fetch NuGet packages (cache for speed) |
| **Build** | `dotnet build -c Release --no-restore` | compile; fail on errors/warnings-as-errors |
| **Test** | `dotnet test --no-build` | run unit/integration tests ([Ch17](../17-Testing/README.md)); fail on red |
| **Publish** | `dotnet publish -c Release` | produce the deployable artifact ([01-PublishModes.md](01-PublishModes.md)) |
| **Containerize** | `docker build`/`/t:PublishContainer` | build the image ([02-Docker.md](02-Docker.md)) |
| **Push** | `docker push` | publish the image to a registry, tagged with the version |
| **Deploy** | `helm upgrade`/`azd deploy`/`kubectl` | roll out to an environment ([04](04-Kubernetes.md), [05](05-Helm.md)) |

Each stage is a **gate**: build failures, test failures, or (optionally) coverage/security thresholds stop the pipeline before a broken change advances. This is the value of CI — fast, automatic feedback on every commit/PR.

---

## CI: build and test on every change

The CI half runs on every push/PR and is about **catching problems early**:

- **Build with warnings-as-errors** (where practical) so quality issues fail the build.
- **Run the test suite** — unit tests always; integration tests (`WebApplicationFactory`/Testcontainers — [Ch17 §05–06](../17-Testing/05-IntegrationTests.md)) where the runner has Docker. Publish test results and **coverage** ([Ch17 §10](../17-Testing/10-TestStrategy.md)).
- **Static analysis / security**: run analyzers, `dotnet format --verify-no-changes`, and dependency/vulnerability scans (e.g., `dotnet list package --vulnerable`).
- **Cache** NuGet packages and build outputs to keep the pipeline fast.

A green CI run means the change compiles, passes tests, and meets quality gates — the precondition for shipping.

---

## CD: package and deploy

The CD half takes a passing build and ships it:

- **Build & tag the container image** with the commit SHA or a semantic version (never just `latest` — pin for reproducibility/rollback — [05-Helm.md](05-Helm.md)).
- **Push** to a registry (ACR, Docker Hub, GHCR).
- **Deploy** to the target: `helm upgrade --install` to Kubernetes, `azd deploy` for Aspire/Azure ([Ch18 §08](../18-Aspire/08-Deployment.md)), `kubectl apply`, or App Service deploy ([10-AppService.md](10-AppService.md)).
- **Environments & approvals**: deploy to staging automatically, gate production behind a manual approval (Continuous *Delivery*) or deploy straight through (Continuous *Deployment*).

```yaml
  deploy:
    needs: build-test
    runs-on: ubuntu-latest
    steps:
      - run: |
          docker build -t myregistry.azurecr.io/orderapi:${{ github.sha }} .
          docker push  myregistry.azurecr.io/orderapi:${{ github.sha }}
      - run: helm upgrade --install orderapi ./chart --set image.tag=${{ github.sha }} -f values.prod.yaml
```

---

## Deployment strategies

How the new version replaces the old (often handled by the platform — [04-Kubernetes.md](04-Kubernetes.md)):

- **Rolling update** — gradually replace old Pods with new (default in K8s; zero-downtime, gated by readiness probes).
- **Blue/green** — run the new version alongside the old, switch traffic at once, roll back instantly by switching back.
- **Canary** — route a small % of traffic to the new version, watch metrics ([Ch12](../12-Observability/README.md)), then ramp up — catches regressions with limited blast radius.

The safer strategies (blue/green, canary) rely on good **observability** to decide whether to proceed or roll back — wiring CD to your telemetry closes the loop.

---

## Secrets and supply-chain in pipelines

- **Secrets**: store registry creds, cloud credentials, and deploy secrets in the CI system's **secret store** (GitHub Actions secrets, Azure Pipelines variable groups / Key Vault) — never in the repo ([Ch13 §07](../13-Configuration/07-Secrets.md)). Prefer **OIDC/federated** credentials over long-lived secrets where supported.
- **Supply chain**: scan dependencies for vulnerabilities, generate an **SBOM**, and ideally **sign** images/artifacts so deployments verify provenance.
- **Least privilege**: the pipeline's deploy identity should have only the permissions it needs.

---

## Common gotchas

### `latest` image tags

Deploying `:latest` makes releases non-reproducible and breaks rollback. Tag with the **commit SHA / semantic version** and deploy that specific tag ([05-Helm.md](05-Helm.md)).

### Tests that need Docker on a runner without it

Integration tests using Testcontainers ([Ch17 §06](../17-Testing/06-TestContainers.md)) need Docker on the runner. Ensure the runner supports it, or gate those tests to a stage that does.

### Secrets in pipeline files

Hardcoding credentials in YAML leaks them. Use the CI secret store / OIDC federation; never commit secrets.

### No quality gates

A pipeline that builds and ships without failing on test/security issues defeats CI's purpose. Make failures **stop** the pipeline (tests, analyzers, vulnerability scans).

### Slow pipelines from no caching

Re-restoring NuGet and rebuilding everything each run is slow. Cache packages/build outputs and use `--no-restore`/`--no-build` between stages.

---

## Summary

- **CI/CD** automates **restore → build → test → publish/containerize → push → deploy** — CI builds/tests every change for fast feedback; CD packages a passing build and ships it (registry → environment), expressed in GitHub Actions/Azure Pipelines/etc.
- **CI** runs on every push/PR with **quality gates** (warnings-as-errors, the test suite incl. integration tests where Docker is available, coverage, analyzers, vulnerability scans) — a green run is the precondition to ship; **cache** for speed.
- **CD** builds and **version/SHA-tags** the image (never `latest`), pushes to a registry, and deploys via **Helm/`azd`/`kubectl`/App Service** — with **environments + approvals** (delivery) or straight-through (deployment) and strategies like **rolling/blue-green/canary** (driven by observability).
- Keep **secrets** in the CI secret store / OIDC (never in repo), scan the **supply chain** (vulnerabilities, SBOM, image signing), and use **least-privilege** deploy identities.

→ Next: [10-AppService.md](10-AppService.md)
