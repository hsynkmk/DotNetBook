# Multi-Tenancy

## Serving many customers from one system

A **multi-tenant** application serves multiple customers (**tenants**) from shared infrastructure, while keeping each tenant's data **isolated**. It's the foundation of SaaS. The central design decision is the **isolation model** — how strongly tenants' data is separated, which trades off **isolation, cost, and operational complexity**: **tenant-per-row** (shared everything, a discriminator column), **tenant-per-schema** (shared database, separate schemas), or **tenant-per-database** (separate databases). Alongside that sits **tenant resolution** (how a request determines which tenant it's for) and the ever-present risk of **data leakage** between tenants — the cardinal sin of multi-tenancy.

---

## The three isolation models

| Model | Isolation | Cost (per tenant) | Complexity | Best for |
|---|---|---|---|---|
| **Tenant per row** | weakest (logical) | **lowest** (shared) | low | many small tenants, max density |
| **Tenant per schema** | medium | medium | medium | middle ground |
| **Tenant per database** | **strongest** (physical) | **highest** | higher (many DBs) | few large/regulated tenants |

### Tenant per row (shared database, shared schema)

Every tenant's rows live in the **same tables**, distinguished by a **`TenantId` discriminator column**. Every query must filter by `TenantId`:

```csharp
// A global query filter ensures EVERY query is scoped to the current tenant ([Ch05 §13]):
modelBuilder.Entity<Order>().HasQueryFilter(o => o.TenantId == _tenantProvider.TenantId);
```

- **Pros**: lowest cost (one DB, shared tables — maximum density), simplest infrastructure, easy cross-tenant analytics.
- **Cons**: **weakest isolation** — a single missing `TenantId` filter leaks one tenant's data to another (catastrophic). Mitigate with **EF Core global query filters** ([Ch05 §13](../05-EFCore/13-GlobalQueryFilters.md)) so the filter is applied **automatically** to every query, not by hand each time.

### Tenant per schema

Each tenant gets its **own schema** within a shared database (same tables, different schema namespaces). Stronger logical isolation than per-row, shared DB infrastructure — a middle ground, though schema management/migrations across many schemas add complexity.

### Tenant per database

Each tenant has a **separate database** (the connection string is selected per tenant at runtime). 

- **Pros**: **strongest isolation** (physical separation — a query simply can't reach another tenant's DB), easy per-tenant backup/restore/scaling, fits **compliance/regulatory** needs.
- **Cons**: highest cost and operational complexity (many databases to provision, migrate, monitor), lower density.

The choice depends on tenant count/size, isolation/compliance requirements, and cost tolerance — many SaaS systems even **mix** (shared DB for small tenants, dedicated DB for large/regulated ones).

---

## Tenant resolution

Every request must determine **which tenant** it belongs to, early in the pipeline, so the rest of the app operates in that tenant's context:

```csharp
// Resolve the tenant per request (middleware), then make it available via DI:
app.Use(async (ctx, next) => {
    var tenantId = ResolveTenant(ctx);   // from one of the strategies below
    ctx.RequestServices.GetRequiredService<ITenantContext>().SetTenant(tenantId);
    await next();
});
```

Common resolution strategies:
- **Subdomain** — `acme.app.com` → tenant "acme".
- **Path** — `/tenants/acme/...`.
- **Header / claim** — a tenant id in a request header or in the authenticated user's **claims** ([Ch10 §07](../10-Identity/07-Claims.md)) (common — the token carries the tenant).
- **Host mapping** — a custom domain mapped to a tenant.

The resolved tenant is stored in a **scoped** ([Ch03 §03](../03-HostingAndDI/03-Lifetimes.md)) `ITenantContext` so it flows to query filters, connection-string selection, caching keys, etc. **Resolve once, early**, and never trust client-supplied tenant ids without verifying them against the authenticated identity (or a user could request another tenant's data).

---

## Cross-cutting tenant concerns

Tenancy touches more than the database:

- **Caching** — cache keys **must** include the tenant id, or one tenant sees another's cached data ([Ch06](../06-DataAndCaching/README.md)). A tenant-blind cache key is a leak.
- **Configuration/features** — per-tenant settings/feature flags ([08-FeatureFlags.md](08-FeatureFlags.md)).
- **Background work** — jobs/messages ([Ch08](../08-BackgroundProcessing/README.md), [Ch07](../07-Messaging/README.md)) must carry and restore the tenant context (the ambient `ITenantContext` isn't automatically present off the request thread).
- **Connection-string selection** (per-database model) — choose the tenant's DB connection at runtime from the resolved tenant.
- **Observability** — tag logs/traces/metrics with the tenant id ([Ch12](../12-Observability/README.md)) so you can diagnose per-tenant.

---

## The data-leakage risk

The defining danger of multi-tenancy is **cross-tenant data leakage** — one tenant accessing another's data. It's catastrophic (privacy/compliance breach), and in the **per-row** model it's one forgotten `WHERE TenantId = ...` away. Defenses:

- **Automate the filter** — EF Core **global query filters** ([Ch05 §13](../05-EFCore/13-GlobalQueryFilters.md)) apply tenant scoping to every query by default (don't rely on remembering it per query).
- **Verify tenant from a trusted source** — derive it from the authenticated identity/claim, not an unverified client value.
- **Defense in depth** — stronger isolation models (schema/DB) make leakage structurally harder; combine with row filters.
- **Test for it** — explicitly test that tenant A cannot read/write tenant B's data ([Ch17](../17-Testing/README.md)).

Treat tenant isolation as a **security boundary**, not just a data-modeling detail.

---

## Common gotchas

### A forgotten tenant filter (per-row leakage)

One query missing the `TenantId` filter leaks data across tenants. Use **automatic global query filters** ([Ch05 §13](../05-EFCore/13-GlobalQueryFilters.md)) rather than hand-filtering each query.

### Tenant-blind cache keys

Caching without the tenant id in the key serves one tenant's data to another. Always include the tenant in cache keys ([Ch06](../06-DataAndCaching/README.md)).

### Trusting client-supplied tenant ids

Accepting a tenant id from a header/parameter without verifying it against the authenticated identity lets a user request another tenant's data. Derive/verify tenant from a **trusted** source ([Ch10 §07](../10-Identity/07-Claims.md)).

### Losing tenant context off the request thread

Background jobs/message handlers don't have the request's ambient tenant context — they must **carry and restore** it explicitly ([Ch08](../08-BackgroundProcessing/README.md)), or they operate with no/wrong tenant.

### Choosing the wrong isolation model

Per-row for highly-regulated tenants (insufficient isolation) or per-database for thousands of tiny tenants (unmanageable cost/ops). Match the model to tenant count/size and compliance needs (and consider mixing).

---

## Summary

- **Multi-tenancy** serves many customers from shared infrastructure with **isolated** data; the core choice is the **isolation model**: **per-row** (shared tables + `TenantId` — lowest cost/density, weakest isolation), **per-schema** (middle ground), or **per-database** (strongest/physical isolation, highest cost — for few large/regulated tenants); systems often **mix**.
- **Resolve the tenant** early per request (subdomain/path/header/**claim**) into a **scoped `ITenantContext`** that flows to queries, connection selection, caching, and logging — and **verify** it against the authenticated identity, never trust unverified client values.
- The defining risk is **cross-tenant data leakage** (one forgotten filter in per-row); defend with **automatic EF Core global query filters** ([Ch05 §13](../05-EFCore/13-GlobalQueryFilters.md)), trusted tenant resolution, stronger isolation models, and explicit tests — treat isolation as a **security boundary**.
- Tenancy is cross-cutting: include the tenant in **cache keys**, **propagate tenant context to background work/messages** ([Ch08](../08-BackgroundProcessing/README.md)), tag **observability** ([Ch12](../12-Observability/README.md)), and support per-tenant config/features ([08-FeatureFlags.md](08-FeatureFlags.md)).

→ Next: [08-FeatureFlags.md](08-FeatureFlags.md)
