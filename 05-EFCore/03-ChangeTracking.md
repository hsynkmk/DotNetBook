# Change Tracking

## How EF knows what to save

When you query tracked entities, modify them, and call `SaveChanges`, EF Core figures out exactly which INSERTs, UPDATEs, and DELETEs to emit. The mechanism is the **change tracker**: it holds a snapshot of each tracked entity and compares current values to detect changes.

```csharp
var product = await db.Products.FindAsync(42);   // tracked; snapshot taken
product.Price = 19.99m;                            // you mutate the entity
await db.SaveChangesAsync();                        // EF detects Price changed → UPDATE Products SET Price=...
```

You never write the UPDATE — you mutate the object and save. The tracker diffs the snapshot against the entity and generates minimal SQL (only changed columns, by default).

---

## Entity states

Every tracked entity has a **state** that determines what `SaveChanges` does:

| State | Meaning | `SaveChanges` emits |
|---|---|---|
| `Unchanged` | tracked, no changes since load | nothing |
| `Added` | new, not yet in DB | INSERT |
| `Modified` | tracked, has changes | UPDATE |
| `Deleted` | marked for removal | DELETE |
| `Detached` | not tracked at all | nothing |

```csharp
db.Products.Add(p);              // state = Added
var existing = await db.Products.FindAsync(id);   // state = Unchanged
existing.Name = "New";           // state = Modified (detected on access/SaveChanges)
db.Products.Remove(existing);    // state = Deleted

var entry = db.Entry(p);
Console.WriteLine(entry.State);  // inspect/override the state explicitly
entry.State = EntityState.Modified;   // force-mark all properties modified
```

`SaveChanges` walks the tracked entities, generates SQL per state, and runs it in one transaction.

---

## Snapshot change detection & `DetectChanges`

By default EF uses **snapshot tracking**: when an entity is tracked, EF stores a copy of its property values. On `SaveChanges` (and certain other operations), it calls **`DetectChanges`**, comparing each tracked entity's current values to its snapshot to find modifications.

```csharp
// DetectChanges runs automatically on SaveChanges, but you can call it manually if needed:
db.ChangeTracker.DetectChanges();
```

This is why mutating a tracked entity "just works" — no `Update()` call needed; the diff finds it. The cost: detecting changes is **O(tracked entities × properties)**. With thousands of tracked entities, `DetectChanges` becomes expensive — a reason to use `AsNoTracking` for reads ([02-Querying.md](02-Querying.md)) and keep contexts short-lived ([01-DbContext.md](01-DbContext.md)). For bulk scenarios, you can disable auto-detect (`ChangeTracker.AutoDetectChangesEnabled = false`) and call `DetectChanges` manually — advanced, error-prone; usually unnecessary.

---

## `Update`, `Attach`, and disconnected entities

In web apps, an entity often comes from outside the context (deserialized from a request body) — it's **disconnected** (not tracked). To save it you must tell EF its state:

```csharp
// Attach — start tracking as Unchanged (then mutate to mark specific props Modified)
db.Products.Attach(product);
db.Entry(product).Property(p => p.Price).IsModified = true;   // update only Price

// Update — start tracking as Modified (marks ALL properties modified → updates every column)
db.Products.Update(product);
await db.SaveChangesAsync();
```

- **`Attach`** — tracks as `Unchanged`; nothing is updated unless you mark properties modified. Use when you want fine-grained control.
- **`Update`** — tracks as `Modified` with **all** properties modified → UPDATE sets every column (even unchanged ones). Convenient but can overwrite fields and clobber concurrent changes.

The common, safer pattern for updates from a request: **load, then mutate** the tracked entity, so only real changes are saved:

```csharp
// ✓ — load + apply changes → minimal, correct UPDATE; also runs concurrency checks
var entity = await db.Products.FindAsync(dto.Id);
if (entity is null) return Results.NotFound();
entity.Name = dto.Name;            // only changed props are updated
entity.Price = dto.Price;
await db.SaveChangesAsync();
```

This avoids over-writing untouched columns and respects concurrency tokens ([07-Concurrency.md](07-Concurrency.md)).

---

## Identity resolution

The tracker maintains **identity resolution**: within one context, an entity with a given key is represented by a **single instance**. Query the same row twice → you get the same tracked object (the second query doesn't overwrite your in-memory changes by default):

```csharp
var a = await db.Products.FindAsync(42);
var b = await db.Products.FirstAsync(p => p.Id == 42);
ReferenceEquals(a, b);   // true — same tracked instance (identity resolution)
```

This keeps your object graph consistent within a unit of work. Note: `AsNoTracking` queries **don't** do identity resolution (each query materializes fresh instances) — use `AsNoTrackingWithIdentityResolution()` if you need de-duplication without tracking.

---

## Bulk updates without loading — `ExecuteUpdate`/`ExecuteDelete`

Change tracking requires **loading** entities to modify them — wasteful for bulk operations. EF 7+ adds **`ExecuteUpdate`/`ExecuteDelete`** that translate directly to a single SQL UPDATE/DELETE without loading or tracking:

```csharp
// Update many rows in ONE SQL statement — no loading, no tracking
await db.Products.Where(p => p.Category == "Obsolete")
    .ExecuteUpdateAsync(s => s.SetProperty(p => p.IsActive, false), ct);

await db.Orders.Where(o => o.Created < cutoff).ExecuteDeleteAsync(ct);
```

For updating/deleting **many rows**, these are far faster than load-modify-save (no round-trips to fetch, no tracking overhead, one SQL statement). Caveat: they **bypass the change tracker** and `SaveChanges` — they execute immediately and won't run interceptors/validation tied to `SaveChanges`. (Raw/bulk SQL: [06-RawSQL.md](06-RawSQL.md).)

---

## Common gotchas

### Forgetting that mutating a tracked entity is enough

You don't call `Update()` on a tracked entity — just mutate it and `SaveChanges`. Calling `Update` on an already-tracked entity needlessly marks all columns modified.

### `Update()` overwriting unchanged columns

`Update` marks every property modified → UPDATE writes all columns, potentially clobbering fields the client didn't intend to change. Prefer load-then-mutate for partial updates.

### Expensive `DetectChanges` with many tracked entities

Tracking thousands of entities makes change detection slow. Use `AsNoTracking` for reads, keep contexts short, and use `ExecuteUpdate/Delete` for bulk.

### Concurrency clobbering with disconnected `Update`

Blindly `Update`-ing a disconnected entity can overwrite changes another user made. Use concurrency tokens ([07-Concurrency.md](07-Concurrency.md)) and/or load-then-mutate.

### Expecting tracking on projections / no-tracking

DTO projections and `AsNoTracking` queries aren't tracked — mutating those results and calling `SaveChanges` does nothing. Track (load entities) when you intend to modify.

### Loading to bulk-delete/update

Loading thousands of rows just to delete them is wasteful. Use `ExecuteDelete`/`ExecuteUpdate`.

---

## Summary

- The **change tracker** snapshots tracked entities and diffs them on **`SaveChanges`** (via `DetectChanges`) to emit minimal INSERT/UPDATE/DELETE — you mutate objects, not write SQL.
- Each tracked entity has a **state** (`Unchanged`/`Added`/`Modified`/`Deleted`/`Detached`) driving what `SaveChanges` does.
- For **disconnected** entities, use **`Attach`** (track as Unchanged) or **`Update`** (all props modified — can over-write); prefer **load-then-mutate** for safe, minimal partial updates.
- **Identity resolution** gives one instance per key within a context (tracking queries); `AsNoTracking` skips it.
- Change detection is O(entities × properties) — use **`AsNoTracking`** for reads, short contexts, and **`ExecuteUpdate`/`ExecuteDelete`** (EF 7+) for bulk operations (no loading/tracking, but bypass `SaveChanges`/interceptors).

→ Next: [04-Relationships.md](04-Relationships.md)
