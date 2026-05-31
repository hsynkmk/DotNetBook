# Azure SDK and Identity

## The modern Azure SDK pattern

The modern Azure SDKs for .NET (the `Azure.*` packages — `Azure.Storage.Blobs`, `Azure.Messaging.ServiceBus`, `Azure.Security.KeyVault.Secrets`, etc.) share a **consistent design**: each service has a strongly-typed **client** you construct with an **endpoint URI + a credential**, register in DI, and call with async methods returning `Response<T>`. The single most important concept across all of them is **how you authenticate** — and the answer, almost always, is **`DefaultAzureCredential`** with **managed identity**: keyless authentication where Azure itself proves your app's identity, so there are **no secrets to store or leak** ([Ch13 §07](../13-Configuration/07-Secrets.md)).

```csharp
// The pattern, everywhere: endpoint + credential → client, registered in DI
builder.Services.AddSingleton(new BlobServiceClient(
    new Uri("https://myaccount.blob.core.windows.net"),
    new DefaultAzureCredential()));            // keyless — no connection string/secret
```

---

## `DefaultAzureCredential` — one credential, every environment

`DefaultAzureCredential` is a **chained credential** that tries multiple authentication mechanisms in order and uses the first that works — so the **same code authenticates** in development and production without changes:

```
DefaultAzureCredential tries, in order:
  1. Environment variables (service principal)        ← CI / explicit config
  2. Managed Identity                                  ← production (App Service, AKS, VM, Container Apps)
  3. Visual Studio / VS Code signed-in account         ← local dev
  4. Azure CLI / Azure PowerShell login                ← local dev
  ...
```

In **production**, it picks up the app's **managed identity** (no credential in code/config). **Locally**, it falls back to *your* developer login (Azure CLI `az login`, Visual Studio). The result: you write `new DefaultAzureCredential()` once, and it "just works" everywhere — the recommended default for all Azure SDK clients.

```csharp
var credential = new DefaultAzureCredential();
var secrets = new SecretClient(new Uri("https://myvault.vault.azure.net"), credential);
var blobs   = new BlobServiceClient(new Uri("https://acct.blob.core.windows.net"), credential);
var bus     = new ServiceBusClient("myns.servicebus.windows.net", credential);
```

---

## Managed identity — keyless authentication

The core security win: **managed identity** gives your Azure-hosted app (App Service, AKS, VM, Container Apps, Functions) an **identity managed by Azure AD (Entra ID)** that authenticates **without any credential stored in your app**. You grant that identity **RBAC roles** on the resources it needs (e.g., "Storage Blob Data Contributor" on a storage account), and the SDK obtains tokens automatically:

- **No connection strings, no access keys, no secrets** in config or code — nothing to leak, nothing to rotate.
- **RBAC, not keys** — access is governed by role assignments (auditable, least-privilege, revocable) instead of shared secret keys.
- **Two flavors**: **system-assigned** (tied to one resource's lifecycle) and **user-assigned** (a standalone identity shared across resources).

This is the antidote to the classic Azure footgun — connection strings/account keys checked into config or leaked. With managed identity + RBAC, the credential never exists in your app at all ([Ch13 §07](../13-Configuration/07-Secrets.md)).

---

## Registering clients with the Azure SDK DI extensions

`Microsoft.Extensions.Azure` provides a clean DI integration so you register all Azure clients in one place, sharing a single credential and getting proper lifetimes:

```csharp
builder.Services.AddAzureClients(clients => {
    clients.AddBlobServiceClient(new Uri("https://acct.blob.core.windows.net"));
    clients.AddServiceBusClient("myns.servicebus.windows.net");
    clients.AddSecretClient(new Uri("https://myvault.vault.azure.net"));
    clients.UseCredential(new DefaultAzureCredential());   // one credential for all
});

// Inject the clients anywhere:
public class FileService(BlobServiceClient blobs) { ... }
```

This registers the clients as **singletons** (the right lifetime — they're thread-safe and pool connections — [Ch03 §03](../03-HostingAndDI/03-Lifetimes.md)), wires the shared credential, and integrates with configuration. It's the idiomatic way to set up Azure clients in an ASP.NET Core/host app.

---

## Client design conventions

All `Azure.*` clients follow shared conventions, so once you learn one you know them all:

- **Clients are thread-safe and meant to be reused** (register as singletons; don't create per-operation).
- **Async-first** — methods return `Task`/`ValueTask` and `Response<T>` (with status + value); honor `CancellationToken`.
- **Built-in resilience** — the SDK retries transient failures with backoff by default ([Ch11](../11-Resilience/README.md)).
- **`Azure.Core`** underpins them all (auth, retries, pipelines, diagnostics) — and integrates with OpenTelemetry ([Ch12](../12-Observability/README.md)), so Azure calls appear in your distributed traces.

---

## Common gotchas

### Using connection strings/account keys instead of managed identity

Keys in config are a leak risk and require rotation. Prefer **`DefaultAzureCredential` + managed identity + RBAC** — keyless, auditable, nothing to leak ([Ch13 §07](../13-Configuration/07-Secrets.md)).

### Creating a client per operation

Azure clients are thread-safe, pool connections, and are designed for reuse. Creating one per call wastes connections/handshakes. Register as **singletons** (via `AddAzureClients`).

### Missing RBAC role assignment

Managed identity needs the right **RBAC role** on the target resource (e.g., "Key Vault Secrets User"). Without it, the SDK authenticates but gets **403**. Grant least-privilege roles to the identity.

### `DefaultAzureCredential` slow/ambiguous locally

The chain probes multiple sources; if several are configured it can be slow or pick an unexpected one. For predictability, scope it (exclude unused sources) or use a specific credential in known environments.

### Forgetting local login

Locally, `DefaultAzureCredential` falls back to your `az login`/Visual Studio account — if you're not logged in (or lack the role), it fails. Run `az login` and ensure your account has access.

---

## Summary

- The modern **Azure SDKs** (`Azure.*`) share a consistent pattern: a **client** built from an **endpoint URI + credential**, registered in DI (as a **singleton** — thread-safe, connection-pooling), called via async `Response<T>` methods with built-in retries and OpenTelemetry.
- **`DefaultAzureCredential`** is a **chained** credential that works **the same in dev and prod**: managed identity in Azure, your developer login (Azure CLI/Visual Studio) locally — write it once, no code changes per environment.
- **Managed identity + RBAC** is the keyless gold standard: Azure AD authenticates your app with **no secret stored** (nothing to leak/rotate), access governed by **role assignments** (least-privilege, auditable) — the antidote to leaked connection strings/keys ([Ch13 §07](../13-Configuration/07-Secrets.md)).
- Register clients with **`AddAzureClients`** (`Microsoft.Extensions.Azure`) sharing one credential; grant the identity the right **RBAC roles** (or get 403s), and reuse clients rather than creating them per call.

→ Next: [02-AppService.md](02-AppService.md)
