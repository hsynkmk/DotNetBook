# Owned Types & Value Objects

## Modeling parts of an entity that aren't their own entity

Some types are conceptually **part of** an entity, not independent things with their own identity — an `Address` belongs to a `Customer`; `Money` is a value, not a row to look up. **Owned types** let EF map these as components of the owner, with no separate key or `DbSet` — supporting clean domain modeling (value objects) and reducing table sprawl.

```csharp
public class Customer {
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public Address ShippingAddress { get; set; } = null!;   // owned — part of Customer
    public Address BillingAddress { get; set; } = null!;
}

public class Address {                 // a value object — no Id, no identity
    public string Street { get; set; } = "";
    public string City { get; set; } = "";
    public string PostalCode { get; set; } = "";
}

// Configure as owned
b.Entity<Customer>().OwnsOne(c => c.ShippingAddress);
b.Entity<Customer>().OwnsOne(c => c.BillingAddress);
```

By default an owned type's properties map to **extra columns in the owner's table** — `ShippingAddress_Street`, `ShippingAddress_City`, `BillingAddress_Street`, etc. The `Address` has no key and no `DbSet`; it only exists as part of a `Customer`.

---

## Owned vs related entity

The distinction is **identity and lifecycle**:

| | Owned type | Related entity (one-to-one) |
|---|---|---|
| Identity | none (a value) | own primary key |
| Lifecycle | tied to the owner (created/deleted with it) | independent |
| Table | owner's table (default) or its own | always its own table |
| Queried independently | no (no `DbSet`) | yes (`DbSet`) |
| Concept | value object (`Address`, `Money`) | an entity (`Profile`, `Account`) |

Use an **owned type** when the thing is a *value* with no independent existence (an address makes no sense without its customer). Use a **related entity** ([04-Relationships.md](04-Relationships.md)) when it has identity and lifecycle of its own. This mirrors DDD's value-object vs entity distinction.

---

## Value objects (DDD)

Owned types are EF's mechanism for **value objects** — immutable, equality-by-value domain types that model concepts without identity:

```csharp
public record Money(decimal Amount, string Currency);   // value object: equal if same amount+currency

public class Order {
    public int Id { get; set; }
    public Money Total { get; set; } = null!;            // owned value object
}
b.Entity<Order>().OwnsOne(o => o.Total);                 // → Total_Amount, Total_Currency columns
```

Records ([CSharpBook Ch03 §03]) are perfect value objects (immutable, value equality). Modeling `Money`/`Address`/`DateRange` as owned value objects keeps the domain expressive (`order.Total.Currency`) while persisting cleanly. Value objects encapsulate validation and behavior (a `Money` can reject negative amounts, do currency-checked arithmetic).

---

## Owned collections

An entity can own a **collection** of value objects:

```csharp
public class Order {
    public int Id { get; set; }
    public List<OrderLine> Lines { get; set; } = [];     // owned collection
}
public class OrderLine {                                  // value object, no identity of its own
    public string Product { get; set; } = "";
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
}
b.Entity<Order>().OwnsMany(o => o.Lines);                 // → a separate OrderLines table with a FK + shadow key
```

`OwnsMany` maps an owned collection to its own table (with a foreign key to the owner and a synthesized key) — but the lines are still **owned** (no independent `DbSet`, lifecycle tied to the order). Use this for components like order lines that belong wholly to their parent. (If lines need independent identity/querying, model them as a related entity instead.)

---

## JSON columns — owned types as JSON

Modern databases support JSON columns; EF can map an owned type (or collection) to a **single JSON column** instead of separate columns/tables:

```csharp
b.Entity<Customer>().OwnsOne(c => c.ShippingAddress, a => a.ToJson());      // stored as one JSON column
b.Entity<Order>().OwnsMany(o => o.Lines, l => l.ToJson());                  // collection as JSON array
```

`ToJson()` (EF 7+) stores the owned data as JSON in one column — and EF can **query into it** (`Where(c => c.ShippingAddress.City == "Boston")` translates to JSON path SQL). This is great for:
- Semi-structured data that's read/written as a unit (settings, metadata, document-like nested data).
- Avoiding extra tables for owned collections.

Trade-offs: JSON columns are harder to index/query than relational columns for some operations, and schema-on-read (no column constraints). Use them for cohesive nested data accessed as a whole; use relational columns/tables when you need indexing, constraints, or relational queries on the parts.

---

## Querying owned types

Owned types come along with their owner automatically (no `Include` needed) — they're part of the entity:

```csharp
var customer = await db.Customers.FirstAsync(c => c.Id == id, ct);
var city = customer.ShippingAddress.City;                 // already loaded (it's part of Customer)

// Query/filter on owned properties
var bostonCustomers = await db.Customers
    .Where(c => c.ShippingAddress.City == "Boston")        // translates to WHERE ShippingAddress_City = 'Boston'
    .ToListAsync(ct);
```

Because owned data is part of the owner's record (or JSON column), it's loaded with the owner — no separate query, no N+1. You filter on owned properties naturally.

---

## Common gotchas

### Using owned types for things with identity

If the "owned" thing is actually an independent entity (queried on its own, shared, has its own lifecycle), model it as a related entity, not owned. Owned = value with no identity.

### Sharing an owned instance between owners

Owned-type instances belong to one owner. Don't assign the same `Address` object to two customers — each owner needs its own instance (they're values, not shared references).

### Required vs optional owned types

An owned type maps to the owner's columns; making it optional (nullable) requires care (EF needs to distinguish "no address" from "address with empty fields"). Configure nullability deliberately.

### JSON columns expecting relational indexing

Data in a JSON column isn't indexed/constrained like relational columns by default. If you frequently filter/sort on a nested property, relational mapping (separate columns) may perform better. Choose JSON for cohesive whole-value access.

### Migrations on owned types

Adding/removing owned properties changes the owner's table (or JSON shape). Review the generated migration like any schema change ([05-Migrations.md](05-Migrations.md)).

---

## Summary

- **Owned types** map a component of an entity (a **value object** with no identity/lifecycle of its own) into the owner's table by default — e.g., `Address`, `Money`. Configure with **`OwnsOne`**/**`OwnsMany`**.
- Choose **owned** for values without independent identity (loaded with the owner, no `DbSet`); a **related entity** when it has its own key/lifecycle/queries.
- Owned types are EF's **value-object** mechanism (use records) — keeping the domain expressive while persisting cleanly; **owned collections** (`OwnsMany`) map to a child table tied to the parent.
- **`ToJson()`** (EF 7+) stores an owned type/collection as a single **JSON column** (queryable via JSON paths) — ideal for cohesive nested data; weigh indexing/constraint trade-offs vs relational mapping.
- Owned data loads **with** the owner (no `Include`, no N+1) and is filterable; don't share owned instances or use owned types for things that really have identity.

→ Next: [13-GlobalQueryFilters.md](13-GlobalQueryFilters.md)
