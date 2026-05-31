# Relationships

## Modeling how entities relate

Relationships map foreign keys and navigations between entities: one-to-many (a customer has many orders), one-to-one (a user has one profile), many-to-many (products have many tags). EF Core configures these by **convention**, **data annotations**, or the **fluent API** — with the fluent API giving the most control.

```csharp
public class Customer {
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public List<Order> Orders { get; set; } = [];      // navigation: one customer → many orders
}

public class Order {
    public int Id { get; set; }
    public int CustomerId { get; set; }                 // foreign key
    public Customer Customer { get; set; } = null!;     // inverse navigation: order → its customer
}
```

EF sees `Order.CustomerId` + the navigations and infers a **one-to-many** by convention. The terms: the **principal** (Customer — the "one") and the **dependent** (Order — the "many", which holds the FK).

---

## Convention, annotations, fluent API

```csharp
// 1. Convention — EF infers from property names (Id/CustomerId/navigations). Least code.

// 2. Data annotations — attributes on the model
public class Order {
    [ForeignKey(nameof(Customer))] public int CustomerId { get; set; }
    public Customer Customer { get; set; } = null!;
}

// 3. Fluent API — in OnModelCreating; most powerful, keeps the model clean
protected override void OnModelCreating(ModelBuilder b) {
    b.Entity<Order>()
        .HasOne(o => o.Customer)
        .WithMany(c => c.Orders)
        .HasForeignKey(o => o.CustomerId)
        .OnDelete(DeleteBehavior.Restrict);   // control cascade behavior
}
```

Prefer **convention** for simple cases (less code, fewer mistakes); use the **fluent API** for anything non-trivial (composite keys, delete behavior, custom names, relationships convention can't infer). Data annotations are a middle ground but mix persistence concerns into your domain types — fluent keeps the entity clean.

---

## One-to-many

The most common relationship — a principal with a collection of dependents:

```csharp
b.Entity<Customer>()
    .HasMany(c => c.Orders)
    .WithOne(o => o.Customer)
    .HasForeignKey(o => o.CustomerId);
```

The dependent (`Order`) holds the FK (`CustomerId`). A **required** relationship uses a non-nullable FK (`int CustomerId` — every order must have a customer); an **optional** one uses a nullable FK (`int? CustomerId`). This nullability drives the default delete behavior.

---

## One-to-one

```csharp
public class User { public int Id { get; set; } public UserProfile? Profile { get; set; } }
public class UserProfile {
    public int Id { get; set; }
    public int UserId { get; set; }                  // FK + unique → one-to-one
    public User User { get; set; } = null!;
}

b.Entity<User>().HasOne(u => u.Profile).WithOne(p => p.User).HasForeignKey<UserProfile>(p => p.UserId);
```

For one-to-one you must specify which side is the dependent (holds the FK) via `HasForeignKey<TDependent>`. The FK gets a unique index so only one dependent per principal. Consider whether a one-to-one should instead be an **owned type** ([12-OwnedTypes.md](12-OwnedTypes.md)) — often a profile/address is better modeled as owned (same table or a clearly-owned table) than a separate entity.

---

## Many-to-many

```csharp
public class Product { public int Id { get; set; } public List<Tag> Tags { get; set; } = []; }
public class Tag { public int Id { get; set; } public List<Product> Products { get; set; } = []; }

// EF Core 5+ infers the join table automatically (no join entity needed):
b.Entity<Product>().HasMany(p => p.Tags).WithMany(t => t.Products);
```

EF Core 5+ supports **implicit join tables** — declare collection navigations on both sides and EF creates the `ProductTag` join table for you. When the join needs **extra columns** (e.g., `AddedDate`, `Order`), model the join entity explicitly:

```csharp
public class ProductTag {
    public int ProductId { get; set; }  public Product Product { get; set; } = null!;
    public int TagId { get; set; }      public Tag Tag { get; set; } = null!;
    public DateTime AddedDate { get; set; }   // payload on the join
}
b.Entity<ProductTag>().HasKey(pt => new { pt.ProductId, pt.TagId });   // composite key
```

Use the implicit join for pure associations; an explicit join entity when the relationship itself carries data.

---

## Cascade delete behavior

When you delete a principal, what happens to its dependents? `OnDelete` controls it:

| Behavior | Effect on dependents when principal deleted |
|---|---|
| `Cascade` | dependents are **deleted** too |
| `Restrict` | **prevent** deleting the principal if dependents exist (throws) |
| `SetNull` | dependent FKs set to null (requires nullable FK) |
| `NoAction` | leave to the database (may error on FK constraint) |

```csharp
b.Entity<Order>().HasOne(o => o.Customer).WithMany(c => c.Orders)
    .OnDelete(DeleteBehavior.Restrict);   // don't let a customer be deleted while orders exist
```

Default: **required** relationships default to `Cascade`, **optional** to `ClientSetNull`. Cascade delete can be dangerous (deleting a customer silently deletes all their orders) — set it deliberately. Many teams prefer `Restrict` (force explicit handling) or **soft delete** ([13-GlobalQueryFilters.md](13-GlobalQueryFilters.md)) over hard cascades.

---

## Required vs optional & nullability

EF infers required/optional from **reference nullability** (with NRT enabled) and FK nullability:

```csharp
public Customer Customer { get; set; } = null!;   // required navigation (non-nullable)
public int CustomerId { get; set; }                // required FK
// vs
public Customer? Customer { get; set; }            // optional navigation
public int? CustomerId { get; set; }               // optional FK (nullable)
```

With Nullable Reference Types ([CSharpBook Ch10 §01]), a non-nullable navigation/FK = required; nullable = optional. This drives schema (NULL/NOT NULL columns) and delete behavior. Keep NRT on so the model's intent is explicit.

---

## Loading related data (recap)

How you *load* relationships is a query concern ([02-Querying.md](02-Querying.md)):

```csharp
// Eager — Include
await db.Customers.Include(c => c.Orders).ThenInclude(o => o.Items).ToListAsync(ct);
// Projection — often best for reads
await db.Customers.Select(c => new { c.Name, OrderCount = c.Orders.Count }).ToListAsync(ct);
// Explicit — load on demand
await db.Entry(customer).Collection(c => c.Orders).LoadAsync(ct);
```

Avoid lazy loading (N+1 — [02-Querying.md](02-Querying.md)). Use `Include` or projection. Multiple collection includes → consider `AsSplitQuery`.

---

## Common gotchas

### Cascade delete surprises

A required relationship defaults to `Cascade` — deleting a principal silently deletes all dependents. Set `OnDelete` deliberately; consider `Restrict` or soft delete.

### Forgetting the FK property

Without a `CustomerId` FK property, EF uses a **shadow** FK (a column not in your class). Explicit FK properties are usually clearer and let you set the relationship by id without loading the principal.

### Many-to-many needing payload

Implicit join tables can't carry extra columns. If the relationship has data (date, order, role), model the join entity explicitly with a composite key.

### Mixing annotations and fluent inconsistently

Conflicting configuration between attributes and fluent API is confusing. Pick a primary approach (fluent for non-trivial) and be consistent.

### One-to-one without specifying the dependent

EF can't guess which side holds the FK in a one-to-one. Specify `HasForeignKey<TDependent>`.

### Required navigation `= null!` confusion

`null!` silences NRT for a navigation EF will populate — it doesn't mean the relationship is optional. Nullability of the FK/navigation is what defines required vs optional.

---

## Summary

- Relationships map FKs + navigations: **one-to-many** (dependent holds the FK), **one-to-one** (specify the dependent via `HasForeignKey<T>`), **many-to-many** (implicit join table in EF 5+, or an explicit join entity when it carries data).
- Configure by **convention** (simple cases), **annotations**, or the **fluent API** (`HasOne/WithMany/HasForeignKey` — preferred for non-trivial config; keeps entities clean).
- **`OnDelete`** controls cascade behavior (`Cascade`/`Restrict`/`SetNull`/`NoAction`) — required relationships default to **Cascade** (dangerous); set it deliberately or use soft delete.
- **NRT nullability** of navigations/FKs defines required vs optional (keep NRT on); explicit FK properties beat shadow FKs.
- Loading is a separate concern — use `Include`/projection, avoid lazy loading ([02-Querying.md](02-Querying.md)).

→ Next: [05-Migrations.md](05-Migrations.md)
