# Interceptors

## Hooking into EF's pipeline

Interceptors let you observe and modify EF Core's operations — SQL commands, `SaveChanges`, connections, transactions — at well-defined points. They're the clean way to implement cross-cutting data concerns (auditing, soft delete, query logging, multi-tenancy stamping) without scattering that logic through your code.

```csharp
public class AuditInterceptor : SaveChangesInterceptor {
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData, InterceptionResult<int> result, CancellationToken ct = default) {
        var context = eventData.Context!;
        foreach (var entry in context.ChangeTracker.Entries<IAuditable>()) {
            if (entry.State == EntityState.Added) entry.Entity.CreatedAt = DateTimeOffset.UtcNow;
            if (entry.State == EntityState.Modified) entry.Entity.UpdatedAt = DateTimeOffset.UtcNow;
        }
        return base.SavingChangesAsync(eventData, result, ct);
    }
}

// Register on the context
o.UseNpgsql(cs).AddInterceptors(new AuditInterceptor());
```

---

## The interceptor types

| Interceptor base | Hooks | Use for |
|---|---|---|
| `SaveChangesInterceptor` | before/after `SaveChanges` | auditing, soft delete, domain events, validation |
| `DbCommandInterceptor` | before/after SQL command execution | query logging, SQL rewriting, command timing |
| `DbConnectionInterceptor` | connection open/close | multi-tenant connection switching, diagnostics |
| `DbTransactionInterceptor` | transaction begin/commit/rollback | transaction diagnostics |

Each base class provides `override`-able methods for the sync and async, "before" (`...ing`) and "after" (`...ed`) variants. You implement only the ones you need.

---

## `ISaveChangesInterceptor` — the most useful

`SavingChanges`/`SavingChangesAsync` runs **before** the database write, with access to the **change tracker** — so you can inspect and modify pending entities. The two killer use cases:

### Auditing (stamp created/updated)

```csharp
public interface IAuditable { DateTimeOffset CreatedAt { get; set; } DateTimeOffset? UpdatedAt { get; set; } }

public class AuditInterceptor(IUserContext user) : SaveChangesInterceptor {
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData e, InterceptionResult<int> result, CancellationToken ct = default) {
        foreach (var entry in e.Context!.ChangeTracker.Entries<IAuditable>()) {
            switch (entry.State) {
                case EntityState.Added:    entry.Entity.CreatedAt = DateTimeOffset.UtcNow; break;
                case EntityState.Modified: entry.Entity.UpdatedAt = DateTimeOffset.UtcNow; break;
            }
        }
        return base.SavingChangesAsync(e, result, ct);
    }
}
```

Every entity implementing `IAuditable` gets `CreatedAt`/`UpdatedAt` stamped automatically on save — no per-entity, per-service code. (Inject `IUserContext` to also stamp `CreatedBy`/`ModifiedBy`.)

### Soft delete (convert deletes to updates)

```csharp
public interface ISoftDelete { bool IsDeleted { get; set; } }

public class SoftDeleteInterceptor : SaveChangesInterceptor {
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData e, InterceptionResult<int> result, CancellationToken ct = default) {
        foreach (var entry in e.Context!.ChangeTracker.Entries<ISoftDelete>()) {
            if (entry.State == EntityState.Deleted) {
                entry.State = EntityState.Modified;        // turn DELETE into UPDATE
                entry.Entity.IsDeleted = true;
            }
        }
        return base.SavingChangesAsync(e, result, ct);
    }
}
```

This intercepts deletes and converts them to "set `IsDeleted = true`" updates — combined with a **global query filter** ([13-GlobalQueryFilters.md](13-GlobalQueryFilters.md)) that hides soft-deleted rows, you get transparent soft delete across the whole app.

---

## `IDbCommandInterceptor` — at the SQL level

Hooks the actual command execution — for logging, timing, or (rarely) rewriting SQL:

```csharp
public class SlowQueryInterceptor(ILogger<SlowQueryInterceptor> log) : DbCommandInterceptor {
    public override async ValueTask<DbDataReader> ReaderExecutedAsync(
        DbCommand command, CommandExecutedEventData e, DbDataReader result, CancellationToken ct = default) {
        if (e.Duration > TimeSpan.FromMilliseconds(500))
            log.LogWarning("Slow query ({Ms}ms): {Sql}", e.Duration.TotalMilliseconds, command.CommandText);
        return result;
    }
}
```

Useful for slow-query logging, adding query hints, or correlating SQL with traces ([Ch12 Observability](../12-Observability/README.md)). For ordinary SQL logging, EF's built-in `LogTo` ([09-Performance.md](09-Performance.md)) is simpler — use a command interceptor when you need to *act* on commands (timing thresholds, rewriting), not just log them.

---

## Registering interceptors with DI

Interceptors often need dependencies (a logger, the current user). Register them in DI and add them via the options:

```csharp
builder.Services.AddScoped<AuditInterceptor>();
builder.Services.AddSingleton<SoftDeleteInterceptor>();

builder.Services.AddDbContext<AppDbContext>((sp, o) => o
    .UseNpgsql(cs)
    .AddInterceptors(
        sp.GetRequiredService<AuditInterceptor>(),       // scoped (needs per-request IUserContext)
        sp.GetRequiredService<SoftDeleteInterceptor>())); // singleton (stateless)
```

Resolve interceptors from the service provider in the options factory so they get their dependencies. Mind lifetimes: a stateless interceptor can be a singleton; one needing per-request context (the current user) should be scoped (and resolved from the request's `sp`).

---

## Interceptors vs alternatives

| Concern | Interceptor | Alternative |
|---|---|---|
| Audit stamps (created/updated) | `SaveChangesInterceptor` ✓ | overriding `SaveChanges` (works, but interceptor is cleaner/reusable) |
| Soft delete | `SaveChangesInterceptor` + query filter ✓ | manual flag-setting everywhere (error-prone) |
| Hide deleted/tenant rows on read | **global query filter** ([13](13-GlobalQueryFilters.md)) | manual `Where` everywhere |
| SQL logging | EF `LogTo` ([09](09-Performance.md)) | command interceptor (when you need to act, not just log) |
| Domain events on save | `SaveChangesInterceptor` ✓ | dispatch in a service after save |

You can also **override `SaveChangesAsync` on the `DbContext`** for save-time logic — equivalent for context-specific behavior, but interceptors are **reusable** across contexts and composable (register several). Use interceptors for cross-cutting concerns; override `SaveChanges` for context-specific one-offs.

---

## Common gotchas

### Interceptor lifetime mismatch

An interceptor needing per-request state (current user) registered as a singleton captures stale/wrong state. Register it scoped and resolve per request; stateless interceptors can be singletons.

### Heavy work in `SavingChanges`

It runs on every save, synchronously blocking the write. Keep it light (stamp fields, flip flags) — don't do I/O or expensive computation in the interceptor.

### Soft delete without a query filter

Converting deletes to updates (interceptor) but not filtering reads (query filter) means "deleted" rows still appear in queries. You need **both** — the interceptor for writes, the global filter for reads ([13-GlobalQueryFilters.md](13-GlobalQueryFilters.md)).

### Forgetting to call `base`

Not calling `base.SavingChangesAsync(...)` (returning the `result`) can skip EF's own processing. Return the base call (or the appropriate `InterceptionResult`).

### Modifying state incorrectly

Changing entity state in `SavingChanges` (like soft delete's Deleted→Modified) must be done carefully and consistently — test it; an incorrect state transition can skip or corrupt the write.

---

## Summary

- **Interceptors** hook EF's pipeline (`SaveChanges`, SQL commands, connections, transactions) for cross-cutting data concerns — clean and reusable vs scattering logic.
- **`SaveChangesInterceptor`** is the most useful: `SavingChanges` runs before the write with change-tracker access — ideal for **auditing** (stamp created/updated) and **soft delete** (convert DELETE → UPDATE).
- **`DbCommandInterceptor`** acts at the SQL level (slow-query logging, hints) — use EF's `LogTo` for plain logging.
- **Register via DI** (resolve in the options factory) and **mind lifetimes** (scoped for per-request state, singleton for stateless).
- **Soft delete needs both** an interceptor (writes) and a **global query filter** (reads); keep interceptor work light; call `base`.

→ Next: [12-OwnedTypes.md](12-OwnedTypes.md)
