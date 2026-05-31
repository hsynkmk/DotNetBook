# Chapter 19 — Deployment

> Publishing .NET apps: Docker, Kubernetes, single-file, Native AOT, ReadyToRun. Choosing between framework-dependent and self-contained. CI/CD basics.

**Prerequisites**: Chapter 04 (ASP.NET Core).

**Time to read**: ~6-8 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-PublishModes.md](01-PublishModes.md) | Framework-dependent, self-contained, single-file, trimmed, AOT — when to use which. |
| [02-Docker.md](02-Docker.md) | Multi-stage Dockerfile, build caching, base image choices (Alpine, Chiseled, Mariner). |
| [03-ChiseledImages.md](03-ChiseledImages.md) | Microsoft's minimal Chiseled base images. Why and when. |
| [04-Kubernetes.md](04-Kubernetes.md) | Deployments, services, configmaps, secrets, probes. |
| [05-Helm.md](05-Helm.md) | Helm chart basics for .NET apps. |
| [06-NativeAOT.md](06-NativeAOT.md) | The full AOT story — what works, what doesn't, what changes. |
| [07-Trimming.md](07-Trimming.md) | Linker, IsTrimmable, DynamicallyAccessedMembers, trim warnings. |
| [08-ReadyToRun.md](08-ReadyToRun.md) | R2R for fast startup without full AOT. |
| [09-CICD.md](09-CICD.md) | GitHub Actions, Azure Pipelines patterns; build → test → containerize. |
| [10-AppService.md](10-AppService.md) | Azure App Service deployment, slots, configuration. |
| [Questions.md](Questions.md) | Drilling. |
| [Coding.md](Coding.md) | Containerize an ASP.NET Core app; publish as AOT; deploy to K8s (locally). |

→ Begin: [01-PublishModes.md](01-PublishModes.md)
