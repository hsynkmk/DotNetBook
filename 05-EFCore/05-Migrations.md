# Migrations

## Evolving the database schema with your model

Migrations are versioned, code-generated scripts that bring the database schema in line with your EF model as it changes over time. You modify your entities, generate a migration (a diff), and apply it — keeping schema changes in source control, reviewable, and repeatable across environments.

```bash
dotnet tool install -g dotnet-ef          # the EF CLI tool (once)

dotnet ef migrations add AddProductTable  # generate a migration from model changes
dotnet ef database update                 # apply pending migrations to the database
dotnet ef migrations remove               # undo the last (unapplied) migration
dotnet ef migrations list                 # show all migrations and which are applied
```

Each `migrations add` compares your current model to the model snapshot from the last migration and emits the **delta** as an `Up()` (apply) and `Down()` (revert) method.

---

## What a migration looks like

```csharp
public partial class AddProductTable : Migration {
    protected override void Up(MigrationBuilder b) {
        b.CreateTable("Products", c => new {
            Id = c.Column<int>(nullable: false).Annotation("Npgsql:ValueGenerationStrategy", ...),
            Name = c.Column<string>(maxLength: 100, nullable: false),
            Price = c.Column<decimal>(precision: 18, scale: 2, nullable: false),
        }, constraints: t => t.PrimaryKey("PK_Products", x => x.Id));
    }
    protected override void Down(MigrationBuilder b) => b.DropTable("Products");
}
```

`Up` applies the change; `Down` reverts it (for rollback). EF also maintains a **model snapshot** (`ModelSnapshot.cs`) representing the cumulative model, and a **`__EFMigrationsHistory`** table in the database recording which migrations have been applied — so `database update` knows what's pending. **Commit all three** (migration, snapshot, and your model changes) together.

---

## The migration workflow

```
1. Change your entities / OnModelCreating (add a property, table, relationship)
2. dotnet ef migrations add <DescriptiveName>   → generates the diff
3. REVIEW the generated migration (don't trust it blindly — see gotchas)
4. dotnet ef database update (dev) or apply in CI/CD (prod)
5. Commit model + migration + snapshot together
```

Always **review** the generated migration before applying — EF's diff is usually right but can miss intent (e.g., a rename looks like drop+add, losing data — below). Treat migrations as code: review them in PRs.

---

## Applying migrations in production

**Do not call `Database.Migrate()` automatically on app startup in production** for multi-instance deployments — concurrent instances racing to migrate, or a bad migration taking down all instances, is dangerous. Prefer **explicit, controlled** application:

```bash
# Generate an idempotent SQL script (safe to run repeatedly; checks history table)
dotnet ef migrations script --idempotent -o migrate.sql

# Or a bundle (a self-contained executable that applies migrations)
dotnet ef migrations bundle -o migrate
```

Production strategies (best → simplest):
1. **Idempotent SQL script** reviewed and run by a DBA / deployment pipeline (`--idempotent` makes it safe to re-run; it checks `__EFMigrationsHistory`).
2. **Migration bundle** — a small executable run as a deploy step (no SDK needed on the target).
3. **`Database.Migrate()` in a controlled, single-instance migration job** (e.g., a Kubernetes init container/job, not in the app's normal startup across N replicas).

The goal: migrations run **once**, **reviewed**, **before** the new app version serves traffic — never racing across instances or unreviewed.

---

## Idempotent scripts & zero-downtime

```bash
dotnet ef migrations script --idempotent FromMigration ToMigration
```

An **idempotent** script wraps each migration in a check against the history table, so running it when some migrations are already applied is safe — ideal for environments where you're unsure of the current state.

For **zero-downtime** deploys, migrations must be **backward-compatible** with the currently-running app version (which briefly runs alongside the new one during a rolling deploy):
- **Additive changes** (new nullable column, new table) are safe — old code ignores them.
- **Destructive changes** (drop/rename a column the old code uses) break the old version mid-deploy.
- Use the **expand/contract** pattern: (1) add the new column (expand), deploy code that writes both; (2) backfill; (3) later, after all instances use the new column, drop the old one (contract). This spreads a breaking schema change across multiple safe deploys.

---

## Common operations

```bash
dotnet ef migrations add Name --output-dir Data/Migrations   # custom location
dotnet ef database update PreviousMigrationName              # roll BACK to a specific migration
dotnet ef migrations script 0 InitialCreate                 # script a range
dotnet ef dbcontext info                                      # inspect the context/provider
dotnet ef dbcontext optimize                                  # compiled model for faster startup (large models)
```

Specify the context if you have several: `--context AppDbContext`. For a separate startup project: `--startup-project ../Api`.

---

## Seeding data

```csharp
// Model seed data (applied via migrations)
b.Entity<Category>().HasData(
    new Category { Id = 1, Name = "Tools" },
    new Category { Id = 2, Name = "Electronics" });
```

`HasData` declares seed rows that EF includes in migrations (INSERT in `Up`, DELETE in `Down`). Good for reference/lookup data with stable keys. For runtime/conditional seeding (creating an admin user, environment-specific data), do it in code at startup (in a controlled migration/seed job, not racing across instances) rather than `HasData`.

---

## Common gotchas

### Auto-migrate on startup across instances

`Database.Migrate()` in app startup with multiple replicas races (concurrent schema changes) and couples deploys to migration success. Apply migrations as a controlled, single-run deploy step.

### Renames generate drop + add (data loss!)

EF often can't tell a rename from a delete+create, so a property rename may generate `DropColumn` + `AddColumn` — **losing the data**. Review and hand-edit the migration to use `RenameColumn`/`RenameTable` instead.

### Not reviewing generated migrations

The diff can be wrong or destructive. Always read it before applying; treat it as reviewable code.

### Destructive change during a rolling deploy

Dropping/renaming a column the still-running old version uses breaks it mid-deploy. Use expand/contract for backward-compatible migrations.

### Forgetting to commit the snapshot

The `ModelSnapshot` must match the migrations; committing the migration without the updated snapshot corrupts the next diff. Commit model + migration + snapshot together.

### Editing applied migrations

Changing a migration already applied to other environments causes drift. Create a **new** migration to fix things, rather than editing an applied one.

---

## Summary

- **Migrations** are versioned code (Up/Down + a model snapshot + `__EFMigrationsHistory`) that evolve the schema with your model: `migrations add` → **review** → `database update`; commit model + migration + snapshot together.
- In **production**, don't auto-migrate on multi-instance startup — apply via a reviewed **idempotent SQL script** or **migration bundle** as a controlled, single-run deploy step, **before** the new app serves traffic.
- For **zero-downtime**, make migrations **backward-compatible** (additive); use **expand/contract** to split breaking changes across deploys.
- **Review** generated migrations — renames can become data-losing drop+add (fix to `RenameColumn`); never edit an already-applied migration (add a new one).
- Use **`HasData`** for stable seed/reference data; do conditional/runtime seeding in a controlled job.

→ Next: [06-RawSQL.md](06-RawSQL.md)
