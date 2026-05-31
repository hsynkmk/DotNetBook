# Configuration and Secrets

## Configuration flows from the AppHost

In a plain app, each service reads its own configuration ([Ch13](../13-Configuration/README.md)). In Aspire, the **AppHost becomes the orchestrator of configuration**: it computes connection strings, resolves endpoints, and injects them into each service as environment variables ([04-ServiceDiscovery.md](04-ServiceDiscovery.md)). On top of that, the AppHost manages **parameters** (named config values, including secrets) and passes them to the services that need them. The result is a single place that wires *what each service is configured with* — instead of duplicating connection strings and settings across many `appsettings.json` files.

```csharp
// AppHost: a parameter (value supplied externally), passed to a service
var apiKey = builder.AddParameter("payment-api-key", secret: true);

builder.AddProject<Projects.Api>("api")
    .WithReference(db)                       // injects ConnectionStrings:ordersdb
    .WithEnvironment("PaymentApiKey", apiKey); // injects the parameter as an env var
```

---

## Where service config comes from

A service in an Aspire app gets configuration from the usual providers ([Ch13 §02](../13-Configuration/02-Providers.md)) **plus** what the AppHost injects:

1. The service's own `appsettings.json` / `appsettings.{Env}.json` (its static settings).
2. **Environment variables injected by the AppHost** — connection strings (`ConnectionStrings__name`), discovery endpoints (`services__name__...`), and parameters you passed with `WithEnvironment`.
3. User secrets (local dev), real config providers (production).

Because the AppHost-injected env vars layer over the service's own config, an Aspire-managed connection string overrides whatever (if anything) the service had — so you **don't put Aspire-managed connection strings in the service's `appsettings.json`** ([05-Integrations.md](05-Integrations.md)). The service just reads `ConnectionStrings:ordersdb` and gets the injected value.

---

## Parameters

**`AddParameter`** declares a named external value the AppHost (and the services it passes it to) can use — for settings that vary by environment or that are secrets:

```csharp
var dbPassword = builder.AddParameter("db-password", secret: true);
var region     = builder.AddParameter("region");                       // non-secret

var pg = builder.AddPostgres("pg", password: dbPassword);              // use a parameter as the password
builder.AddProject<Projects.Api>("api").WithEnvironment("Region", region);
```

A parameter's **value** comes from the AppHost's configuration (e.g., its `appsettings.json`, **user secrets** locally, or environment/parameters in deployment). `secret: true` marks it sensitive so it's treated carefully (masked in the dashboard, surfaced as a secret in deployment manifests — [08-Deployment.md](08-Deployment.md)). Parameters are how you thread environment-specific and sensitive values through the app model without hardcoding them.

---

## Secrets — local vs production

The secret story follows the general rules ([Ch13 §07](../13-Configuration/07-Secrets.md)) — never in the repo:

- **Local development**: parameter values (including secrets) come from the AppHost's **user secrets** (`dotnet user-secrets`) — stored outside the repo, not committed. A secret parameter reads its value from there.

```bash
# In the AppHost project:
dotnet user-secrets set "Parameters:db-password" "s3cr3t"
```

- **Production**: secret parameters map to the platform's secret mechanism — e.g., Azure Key Vault / Container Apps secrets via `azd` ([08-Deployment.md](08-Deployment.md)), or Kubernetes secrets. The deployment manifest marks them as secrets so the tooling provisions them securely rather than baking them into images/config.

So a secret parameter is sourced from user secrets locally and from a secret store in production — the *same* AppHost declaration, different backing per environment, and **never** committed.

---

## Per-environment values

Different environments need different values (connection strings to different databases, feature flags, region). In Aspire you express environment differences via:

- **Parameters** whose values differ per environment (supplied by the AppHost's environment-specific config or by the deployment).
- The AppHost reacting to the environment (`builder.Environment` / `builder.ExecutionContext`) to add resources or set values conditionally:

```csharp
if (builder.Environment.IsDevelopment())
    builder.AddRedis("cache");                 // local container in dev
else
    builder.AddConnectionString("cache");      // reference an existing managed cache in prod
```

`AddConnectionString("name")` references an **externally-provided** connection string (from the AppHost's config / deployment) rather than provisioning a container — the typical way to point at a real managed resource in production while running a container locally.

---

## Common gotchas

### Putting Aspire-managed connection strings in `appsettings.json`

The AppHost injects connection strings; duplicating them in the service's config causes confusion/override surprises. Let the integration read the injected value; don't hardcode it ([05-Integrations.md](05-Integrations.md)).

### Secrets in the repo

A secret parameter's value must come from **user secrets** (local) or a **secret store** (prod) — never from committed config. Mark sensitive parameters `secret: true` and source them appropriately ([Ch13 §07](../13-Configuration/07-Secrets.md)).

### Forgetting `secret: true`

A sensitive parameter without `secret: true` may be shown in the dashboard and emitted as a plain value in deployment manifests. Mark secrets so they're masked/provisioned securely.

### Expecting the AppHost to read the service's config

The flow is AppHost → service (the AppHost injects config *into* services). The AppHost has its own config (for parameters); it doesn't read each service's `appsettings.json`.

### Hardcoding environment-specific resources

Branch on `builder.Environment`/`ExecutionContext` (local container vs `AddConnectionString` to a managed resource) instead of hardcoding — keeps the model portable across dev and prod.

---

## Summary

- In Aspire the **AppHost orchestrates configuration**: it injects **connection strings** and **discovery endpoints** into services as env vars (layered over the service's own config), so you read them by name and **don't** hardcode Aspire-managed connection strings in `appsettings.json`.
- **`AddParameter`** declares named external values (with `secret: true` for sensitive ones) threaded to services via `WithEnvironment` or used in resource config (e.g., a DB password); parameter values come from the AppHost's config.
- **Secrets**: sourced from the AppHost's **user secrets** locally and from a **platform secret store** (Key Vault / Container Apps / Kubernetes secrets) in production via the deployment manifest — the same declaration, different backing, **never committed** ([Ch13 §07](../13-Configuration/07-Secrets.md)).
- Handle **per-environment** differences with environment-varying parameters and `builder.Environment`/`ExecutionContext` branching (e.g., a local **container** in dev vs **`AddConnectionString`** to a managed resource in prod) — keeping the app model portable.

→ Next: [08-Deployment.md](08-Deployment.md)
