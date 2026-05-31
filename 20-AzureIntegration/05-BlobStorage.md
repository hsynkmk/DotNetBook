# Azure Blob Storage

## Object storage for files and unstructured data

**Azure Blob Storage** is Azure's massively-scalable object store — for files, images, backups, logs, video, and any unstructured data. It's organized as **storage account → container → blob**, and accessed via `Azure.Storage.Blobs` following the standard SDK pattern ([01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)). Blobs are the go-to for "store a file in the cloud": user uploads, document storage, static assets, data lakes. Key concerns are **uploading/downloading efficiently** (streaming, not buffering), **access control** (managed identity + RBAC, or SAS tokens for delegated access), and **cost tiers**.

```csharp
builder.Services.AddAzureClients(c =>
    c.AddBlobServiceClient(new Uri("https://myaccount.blob.core.windows.net"))
     .UseCredential(new DefaultAzureCredential()));        // keyless

// Upload:
var container = blobService.GetBlobContainerClient("documents");
await container.CreateIfNotExistsAsync();
var blob = container.GetBlobClient("report.pdf");
await blob.UploadAsync(stream, overwrite: true);
```

---

## The hierarchy

```
Storage Account  (myaccount)
  └── Container   (documents)        — like a top-level folder; access scope
        └── Blob  (2026/report.pdf)  — the object (the "/" is just a name convention, not real folders)
```

- **Account** — the storage namespace (with an endpoint and access config).
- **Container** — a grouping with its own access level; the unit of RBAC/SAS scope.
- **Blob** — the actual object. There are three types: **Block blobs** (the common case — files, streamed in blocks), **Append blobs** (append-only, for logs), and **Page blobs** (random-access, for VM disks).

The "folders" in a blob name (`2026/report.pdf`) are **virtual** — blob storage is flat; the path is part of the name, and you list "by prefix" to simulate folders.

---

## Uploading and downloading — stream, don't buffer

The performance rule: **stream** large blobs rather than loading them fully into memory ([Ch02 §04](../02-BCL/04-IO.md)). The SDK uploads/downloads in chunks:

```csharp
// Stream an upload (e.g., from an HTTP request body) — doesn't buffer the whole file in memory:
await blob.UploadAsync(requestStream, new BlobUploadOptions {
    HttpHeaders = new BlobHttpHeaders { ContentType = "application/pdf" }
});

// Stream a download to a response:
await blob.DownloadToAsync(responseStream);

// Open a readable stream for processing:
await using var stream = await blob.OpenReadAsync();
```

For large files the SDK handles **parallel block upload/download** automatically. Avoid `UploadAsync(BinaryData)` of a multi-GB file held in memory — stream it. This matters for both memory ([Ch01 §04](../01-Runtime/04-GCDeepDive.md)) and throughput.

---

## Access control: managed identity vs SAS

Two ways to grant access:

- **Managed identity + RBAC** ([01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)) — your *app* accesses the account with no keys, governed by roles ("Storage Blob Data Contributor"). The default for app-to-storage access.
- **SAS (Shared Access Signature)** — a **time-limited, scoped URL** that grants a *specific* permission (read/write) to a *specific* blob/container for a *limited* time — to hand to a **client** (browser, mobile app) so it can upload/download **directly** without routing bytes through your server:

```csharp
// Generate a read SAS valid for 1 hour, give the URL to a client to download directly:
var sasUri = blob.GenerateSasUri(BlobSasPermissions.Read, DateTimeOffset.UtcNow.AddHours(1));
```

**User-delegation SAS** (signed with the managed identity, not the account key) is the secure modern form — it doesn't require the account key at all. SAS is the pattern for **direct client access** (offload uploads/downloads from your server), with least-privilege, expiring access.

---

## Access tiers (cost)

Blob storage has **access tiers** trading storage cost vs access cost/latency — match the tier to data temperature:

| Tier | For | Storage cost | Access cost/latency |
|---|---|---|---|
| **Hot** | frequently accessed | higher | low |
| **Cool** | infrequent (≥30 days) | lower | higher |
| **Cold** | rarely accessed (≥90 days) | lower still | higher |
| **Archive** | long-term backup | lowest | **must rehydrate** (hours) before read |

**Lifecycle management** policies auto-transition blobs between tiers by age (e.g., Hot → Cool after 30 days → Archive after a year) — significant cost savings for data that ages. Archive is offline-ish: you must **rehydrate** before reading (slow), so only for true cold storage.

---

## Other features

- **Metadata & tags** — attach key/value metadata to blobs; **blob index tags** are queryable.
- **Concurrency** — **ETags** + conditional headers (`If-Match`) for optimistic concurrency ([Ch05 §07](../05-EFCore/07-Concurrency.md) covers the concept) to avoid lost updates.
- **Versioning & soft delete** — recover overwritten/deleted blobs.
- **Events** — blob create/delete events via Event Grid ([08-EventGrid-EventHubs.md](08-EventGrid-EventHubs.md)) — e.g., trigger a Function on upload ([03-Functions.md](03-Functions.md)).

---

## Common gotchas

### Buffering large blobs in memory

Loading a multi-GB blob fully into memory (`byte[]`/`BinaryData`) causes memory spikes/OOM. **Stream** uploads/downloads ([Ch02 §04](../02-BCL/04-IO.md)).

### Routing client uploads through your server

Proxying large user uploads/downloads through your app wastes bandwidth/CPU. Issue a **SAS** so clients upload/download **directly** to/from storage.

### Account keys instead of identity/user-delegation SAS

Account keys are powerful and leakable. Use **managed identity + RBAC** for app access and **user-delegation SAS** (key-less) for client access ([01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)).

### Forgetting Archive rehydration

Archive-tier blobs can't be read directly — you must rehydrate (hours). Don't archive data you might need quickly; use lifecycle policies thoughtfully.

### Treating blob "folders" as real

Blob storage is flat; paths are virtual (part of the name). List by **prefix** to enumerate a "folder" — there's no real directory object.

---

## Summary

- **Azure Blob Storage** is scalable object storage (**account → container → blob**; block/append/page blob types) for files and unstructured data, via `Azure.Storage.Blobs` with **managed identity** ([01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)); "folders" are virtual (flat namespace, list by prefix).
- **Stream** uploads/downloads (the SDK does parallel block transfer) — never buffer large blobs fully in memory ([Ch02 §04](../02-BCL/04-IO.md)).
- **Access**: **managed identity + RBAC** for app access; **SAS** (ideally **user-delegation SAS** — keyless) for time-limited, scoped **direct client** upload/download, offloading bytes from your server.
- **Access tiers** (Hot/Cool/Cold/Archive) trade storage vs access cost — use **lifecycle policies** to auto-tier aging data; Archive requires **rehydration**. Use ETags for concurrency, and blob **events** to trigger Functions ([03-Functions.md](03-Functions.md)).

→ Next: [06-CosmosDB.md](06-CosmosDB.md)
