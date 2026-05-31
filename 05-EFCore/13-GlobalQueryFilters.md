# Global Query Filters

## Automatic WHERE clauses on every query

A global query filter is a predicate EF Core **automatically adds to every query** for an entity type — so you don't have to remember to write `Where(x => !x.IsDeleted)` or `Where(x => x.TenantId == current)` everywhere. The two dominant uses: **soft delete** (hide deleted rows) and **multi-tenancy** (show only the current tenant's rows).

```csharp
protected override void OnModelCreating(ModelBuilder b) {
    // Every query for Product automatically gets: WHERE IsDeleted = false
    b.Entity<Product>().HasQueryFilter(p => !p.IsDeleted);
}

// Now this query is automatically filtered:
var products = await db.Products.ToListAsync(ct);   // → WHERE IsDeleted = false (added by EF)
```

The filter applies to direct queries, `Include`s, and navigations — consistently, everywhere — eliminating the risk of forgetting it on one query and leaking deleted/other-tenant data.

---

## Soft delete

Combine a query filter (hides deleted rows on **read**) with a `SaveChanges` interceptor (converts deletes to flag-updates on **write** — [11-Interceptors.md](11-Interceptors.md)) for transparent soft delete:

```csharp
public interface ISoftDelete { bool IsDeleted { get; set; } }

// 1. Filter reads — deleted rows are invisible to normal queries
b.Entity<Product>().HasQueryFilter(p => !p.IsDeleted);

// 2. Intercept writes — DELETE becomes "set IsDeleted = true" (the interceptor from §11)
//    (SoftDeleteInterceptor converts EntityState.Deleted → Modified + IsDeleted = true)

// Usage — looks like a normal delete, behaves as soft delete:
db.Products.Remove(product);
await db.SaveChangesAsync(ct);     // row stays, IsDeleted = true; future queries don't see it
```

You need **both halves**: the interceptor for the write, the filter for the read. With both, `Remove` + `SaveChanges` soft-deletes, and every subsequent query automatically excludes it — no per-query `Where`.

---

## Multi-tenancy (row-level)

For a shared-database, shared-schema multi-tenant app (tenant-per-row), a query filter scopes every query to the current tenant:

```csharp
public class AppDbContext(DbContextOptions o, ITenantContext tenant) : DbContext(o) {
    private readonly Guid _tenantId = tenant.TenantId;   // resolved per request

    protected override void OnModelCreating(ModelBuilder b) {
        b.Entity<Order>().HasQueryFilter(o => o.TenantId == _tenantId);
        b.Entity<Customer>().HasQueryFilter(c => c.TenantId == _tenantId);
    }
}

// Every query is automatically scoped to the current tenant:
var orders = await db.Orders.ToListAsync(ct);   // → WHERE TenantId = @currentTenant
```

The filter references a field captured from the (scoped) context's tenant context, so each request's `DbContext` filters to that request's tenant. This is the safest row-level multi-tenancy approach: a developer **can't accidentally** query across tenants because the filter is always applied. (Multi-tenancy strategies — per-row/per-schema/per-database — in [Ch22 §07](../22-BestPractices/README.md).) Pair with a `SaveChanges` interceptor that stamps `TenantId` on inserts so writes are tenant-scoped too.

---

## Bypassing the filter

Sometimes you legitimately need to see filtered-out rows — an admin restoring a soft-deleted record, a background job across all tenants, an audit. Use **`IgnoreQueryFilters`**:

```csharp
// See ALL rows, including soft-deleted / all tenants
var allIncludingDeleted = await db.Products.IgnoreQueryFilters().ToListAsync(ct);
var deletedOnly = await db.Products.IgnoreQueryFilters().Where(p => p.IsDeleted).ToListAsync(ct);
```

`IgnoreQueryFilters()` removes **all** global filters for that query. Use it deliberately and carefully — especially for multi-tenant filters, where bypassing means crossing tenant boundaries (a serious data-leak risk if misused). Restrict such queries to admin/system code paths with proper authorization.

---

## Multiple filters & combining

```csharp
// One filter per entity; combine conditions with && (EF 10 also supports multiple named filters)
b.Entity<Order>().HasQueryFilter(o => !o.IsDeleted && o.TenantId == _tenantId);
```

Historically an entity had a **single** query filter (combine conditions with `&&`). EF 10 adds support for **multiple, named filters** per entity so soft-delete and tenancy filters can be defined and ignored independently:

```csharp
// EF 10: named filters, independently ignorable
b.Entity<Order>().HasQueryFilter("SoftDelete", o => !o.IsDeleted);
b.Entity<Order>().HasQueryFilter("Tenant", o => o.TenantId == _tenantId);
// later: db.Orders.IgnoreQueryFilters(["SoftDelete"])  → ignore only soft-delete, keep tenant filter
```

Named filters let you bypass *one* filter (e.g., show deleted rows) while **keeping** another (e.g., still tenant-scoped) — important so "show deleted" doesn't accidentally also cross tenants.

---

## Filters and navigations / Includes

Query filters apply to **navigations and `Include`s** too — a `Customer.Orders` navigation only loads non-deleted, same-tenant orders. This consistency is the point: the filter follows the data everywhere. Be aware: a required relationship whose principal is filtered out can make dependents appear "orphaned" in queries — design filters consistently across related entities (filter both `Order` and its `Customer` on tenant, etc.).

---

## Common gotchas

### Soft delete filter without the write interceptor (or vice versa)

The filter hides deleted rows on read; the interceptor turns deletes into flag-updates on write. You need **both** — filter alone doesn't soft-delete; interceptor alone leaves deleted rows visible.

### `IgnoreQueryFilters` crossing tenants

Bypassing filters removes the tenant filter too (with a single combined filter), risking cross-tenant data exposure. Use EF 10 named filters to ignore only soft-delete while keeping tenancy; restrict bypass to authorized admin code.

### Forgetting filters apply to Includes/navigations

Filtered entities are also excluded from `Include`s and navigation loads. Usually desirable, but be aware when debugging "missing" related data.

### Inconsistent filters across related entities

Filtering `Order` by tenant but not `Customer` (or soft-deleting one side only) creates inconsistent graphs. Apply filters consistently across related types.

### Captured context state in the filter

A tenant filter references the context's tenant field — make sure the context is scoped per request so the captured tenant is correct (don't share/reuse a context across tenants).

### Performance of complex filters

A heavy filter predicate is added to every query — keep it simple (indexed columns: `IsDeleted`, `TenantId`) so it doesn't slow all queries.

---

## Summary

- **Global query filters** (`HasQueryFilter`) auto-add a `WHERE` predicate to **every query** (and Include/navigation) for an entity — so you never forget to filter, eliminating soft-deleted/cross-tenant data leaks.
- **Soft delete** = query filter (`!IsDeleted`, hides on read) **+** a `SaveChanges` interceptor (DELETE → flag update, on write) — you need both halves.
- **Multi-tenancy** (tenant-per-row) = a filter scoping every query to the current (scoped-context) tenant — the safest approach, since cross-tenant queries can't happen accidentally.
- Bypass with **`IgnoreQueryFilters()`** for admin/system paths — carefully (bypassing can cross tenants); **EF 10 named filters** let you ignore one filter (soft-delete) while keeping another (tenancy).
- Keep filter predicates simple and indexed (they run on every query); apply them consistently across related entities.

→ Next: [14-Testing.md](14-Testing.md)
