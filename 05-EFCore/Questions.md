# Chapter 05 — EF Core — Q & A

---

### Q1. What is a `DbContext`?

A session with the database that acts as a **unit of work** (batches changes, commits on `SaveChanges`) and a set of **repositories** (`DbSet<T>` per entity). It queries (LINQ→SQL), tracks changes, and saves them atomically. Because it's already a UoW + repository, wrapping it in another repository layer is often redundant.

---

### Q2. Why must a `DbContext` be short-lived and not shared across threads?

It's **not thread-safe** (concurrent use → "second operation started on this context" / corruption) and accumulates tracked entities (memory growth, slower change detection). It's registered scoped (one per request); for parallelism use `IDbContextFactory`, and never inject it into a singleton (captive dependency).

---

### Q3. When does a LINQ query against a `DbSet` execute?

At a **terminal** operator (`ToListAsync`, `FirstAsync`, `CountAsync`, `AnyAsync`, etc.). Before that, LINQ builds an expression tree that EF translates to SQL — deferred execution. Always use the async terminals and pass the `CancellationToken`.

---

### Q4. What happens when a query can't be translated to SQL?

Modern EF Core (3.0+) **throws** ("could not be translated") rather than silently evaluating on the client (the old footgun that fetched everything and filtered in memory). If you need client logic, materialize first (`ToListAsync`) then apply it deliberately.

---

### Q5. What are the biggest easy query performance wins?

**Project to DTOs** (select only needed columns, untracked), use **`AsNoTracking`** for read-only entity queries, and **eliminate N+1** (Include/projection, disable lazy loading). And ensure the database has **indexes** on filtered/sorted columns.

---

### Q6. What is the N+1 problem and how do you fix it?

One query for parents, then a separate query per parent for its children (often via lazy loading or a loop) — the dominant EF perf bug. Fix with **`Include`** (eager loading) or **projection** (compute aggregates in SQL). Detect by logging the generated SQL (repeated query shapes).

---

### Q7. Tracking vs `AsNoTracking`?

Tracked queries build change-tracking snapshots so `SaveChanges` can detect modifications — necessary only if you'll update the entities. **`AsNoTracking`** skips snapshots and identity resolution → faster, lighter, for read-only queries. DTO projections are already untracked.

---

### Q8. Single vs split queries?

Including multiple **collection** navigations in one query causes a **cartesian explosion** (rows multiply). **`AsSplitQuery`** runs a separate query per collection — more round-trips but no duplicated data. Use it for multiple collection includes; single query for one collection.

---

### Q9. How does change tracking know what to save?

The change tracker snapshots tracked entities' values; on `SaveChanges` it runs **`DetectChanges`**, diffing current values against the snapshot to find modifications, then emits minimal INSERT/UPDATE/DELETE based on each entity's **state** (Added/Modified/Deleted/Unchanged) — all in one transaction.

---

### Q10. `Attach` vs `Update` for disconnected entities?

`Attach` tracks as `Unchanged` (nothing updated unless you mark properties modified). `Update` tracks as `Modified` with **all** properties modified → updates every column (can over-write unchanged fields and clobber concurrent changes). Prefer **load-then-mutate** for safe, minimal partial updates.

---

### Q11. What are `ExecuteUpdate`/`ExecuteDelete`?

EF 7+ methods that translate a LINQ filter directly to a single SQL UPDATE/DELETE **without loading or tracking** entities — far faster for bulk operations. Caveat: they bypass the change tracker and `SaveChanges` (and its interceptors), executing immediately.

---

### Q12. How do you configure relationships?

By **convention** (FK + navigation property names — simplest), **data annotations** (`[ForeignKey]`), or the **fluent API** (`HasOne/WithMany/HasForeignKey` in `OnModelCreating` — most powerful, keeps entities clean). Prefer convention for simple cases, fluent for non-trivial.

---

### Q13. What's the danger with cascade delete?

Required relationships default to `OnDelete(Cascade)` — deleting a principal silently deletes all dependents (e.g., deleting a customer deletes all their orders). Set `OnDelete` deliberately; consider `Restrict` (force explicit handling) or **soft delete**.

---

### Q14. How should migrations be applied in production?

Not via auto-`Database.Migrate()` on multi-instance startup (instances race; couples deploy to migration). Apply a reviewed **idempotent SQL script** (`--idempotent`) or a **migration bundle** as a controlled, single-run deploy step, **before** the new app serves traffic.

---

### Q15. How do you make migrations zero-downtime safe?

Make them **backward-compatible** with the currently-running app version (additive changes are safe). For breaking changes use **expand/contract**: add the new column, deploy code writing both, backfill, then later drop the old column — spreading the change across safe deploys.

---

### Q16. Why can a property rename cause data loss in a migration?

EF often can't distinguish a rename from a delete+create, so it may generate `DropColumn` + `AddColumn` — losing the data. **Review** generated migrations and hand-edit to `RenameColumn`/`RenameTable` when needed.

---

### Q17. How do you avoid SQL injection with raw SQL in EF?

Use **`FromSql`/`ExecuteSql`** with interpolated strings — they parameterize the holes (safe). **`FromSqlRaw`/`ExecuteSqlRaw`** take plain strings; interpolating into them concatenates *before* EF sees it → **injectable**. With `Raw` methods, pass explicit parameters.

---

### Q18. How does EF Core handle concurrency conflicts?

**Optimistic concurrency**: a **concurrency token** (ideally a rowversion) is added to the UPDATE/DELETE `WHERE` clause, so the statement affects 0 rows if the row changed since load → `DbUpdateConcurrencyException`. You resolve it (store wins / client wins / merge) instead of silently overwriting.

---

### Q19. Is `SaveChanges` transactional?

Yes — every `SaveChanges` wraps all its INSERT/UPDATE/DELETE statements in a single transaction (all commit or all roll back). So making all related changes and calling `SaveChanges` **once** gives atomicity. Multiple `SaveChanges` for one operation are separate transactions (risking partial state).

---

### Q20. When do you need an explicit transaction?

When a unit of work spans **multiple `SaveChanges`** calls or mixes EF with raw SQL — wrap them in `BeginTransactionAsync` + `CommitAsync` (with `await using` for auto-rollback). With a retry strategy enabled, wrap the whole transaction in `CreateExecutionStrategy().ExecuteAsync`.

---

### Q21. Why avoid distributed transactions, and what's the alternative?

They (2PC across multiple DBs/brokers) are complex, slow, and poorly supported in .NET. Keep each transaction within one database; for cross-resource consistency (DB write + message publish) use the **outbox pattern** (write the message in the same DB transaction, publish later) or sagas.

---

### Q22. When use `IDbContextFactory`?

When the scoped context doesn't fit: **parallel queries** (a context per task — not thread-safe), **Blazor Server** (components are long-lived; create a short-lived context per operation — the recommended pattern), and **singletons/background services** (avoid the captive dependency). You own and dispose the created context.

---

### Q23. `IDbContextFactory` vs `IServiceScopeFactory` in a background service?

`IServiceScopeFactory` creates a full DI scope — use when you need several scoped services (the context plus a scoped validator/repo). `IDbContextFactory` creates just a context — cleaner when the `DbContext` is all you need.

---

### Q24. What's the best use of a `SaveChangesInterceptor`?

Cross-cutting save-time concerns with change-tracker access: **auditing** (stamp `CreatedAt`/`UpdatedAt`/`CreatedBy` on Added/Modified entities) and **soft delete** (convert `EntityState.Deleted` → `Modified` + `IsDeleted = true`). Reusable across contexts, unlike overriding `SaveChanges`.

---

### Q25. Owned type vs related entity?

An **owned type** is a value object with no identity/lifecycle of its own (mapped into the owner's table by default; no `DbSet`) — e.g., `Address`, `Money`. A **related entity** has its own key, lifecycle, and `DbSet`, queried independently. Use owned for values, related entities for things with identity.

---

### Q26. What does `ToJson()` do for owned types?

Maps an owned type/collection to a **single JSON column** (EF 7+) instead of separate columns/tables, and EF can query into it via JSON paths. Good for cohesive nested/semi-structured data accessed as a whole; weigh the loss of relational indexing/constraints.

---

### Q27. What are global query filters used for?

A predicate EF **automatically adds to every query** for an entity — eliminating the risk of forgetting a `Where`. The two dominant uses: **soft delete** (`!IsDeleted`, hides deleted rows) and **multi-tenancy** (`TenantId == current`, scopes to the current tenant). Bypass with `IgnoreQueryFilters()`.

---

### Q28. What does transparent soft delete require?

**Both** a global query filter (`!IsDeleted`, hides deleted rows on read) **and** a `SaveChanges` interceptor (converts `Deleted` → `Modified` + `IsDeleted = true` on write). The filter alone doesn't soft-delete; the interceptor alone leaves deleted rows visible.

---

### Q29. Why not use the InMemory provider for testing?

It's **not a relational database** — no SQL translation (misses untranslatable queries), no constraints (FK/unique/not-null), no real transactions, different semantics. It gives false confidence (passes tests that fail in production). Microsoft advises against it; prefer **SQLite in-memory** (real SQL engine) or **Testcontainers** (real DB).

---

### Q30. Why shouldn't you mock `DbContext`/`DbSet`?

`IQueryable` over a mock runs LINQ-to-Objects, not LINQ-to-Entities — so it won't catch translation errors, differs for async, and tests the mock setup rather than real behavior. Mock a **repository interface** for unit tests, or use a real provider (SQLite/Testcontainers) for data-layer tests.

---

→ Next: [Coding.md](Coding.md)
