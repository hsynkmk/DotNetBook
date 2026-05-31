# Azure Cosmos DB

## Globally-distributed NoSQL

**Azure Cosmos DB** is Azure's globally-distributed, multi-model NoSQL database, built for **low-latency, elastic scale, and global replication**. Unlike a relational database ([Ch05 EF Core](../05-EFCore/README.md)), Cosmos stores **schema-flexible documents** (JSON), scales horizontally by **partitioning**, and offers single-digit-millisecond reads/writes with tunable consistency and multi-region writes. The .NET client is `Microsoft.Azure.Cosmos`, following the standard SDK pattern ([01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)). The single most important design decision in Cosmos is the **partition key** — get it wrong and you sacrifice scale and pay more; get it right and it scales nearly limitlessly.

```csharp
var cosmos = new CosmosClient("https://acct.documents.azure.com", new DefaultAzureCredential());
var container = cosmos.GetContainer("ShopDb", "orders");

await container.CreateItemAsync(order, new PartitionKey(order.CustomerId));   // write
var read = await container.ReadItemAsync<Order>(order.Id, new PartitionKey(order.CustomerId));  // point read
```

---

## The data model

- **Account → Database → Container → Items.** A **container** holds JSON **items** (documents); it's the unit of scale/throughput.
- **Schema-flexible** — items in a container don't need the same shape (though you usually keep them consistent per type). No migrations for adding a field.
- **Throughput (RU/s)** — Cosmos meters operations in **Request Units**; you provision (or autoscale) RU/s per container or share across a database. Every read/write/query **costs RUs** based on size/complexity — cost and rate-limiting (429s) are RU-driven, a different mental model from relational DBs.

---

## Partitioning — the critical decision

Cosmos scales by **horizontal partitioning**: items are distributed across **physical partitions** by the **partition key** you choose. The partition key determines scalability, performance, and cost:

```csharp
// Choose a partition key with high cardinality and even access distribution:
await container.CreateItemAsync(order, new PartitionKey(order.CustomerId));   // partition by customer
```

A good partition key:
- **High cardinality** — many distinct values, so data/load spreads across partitions.
- **Even distribution** — no "hot partition" that takes disproportionate traffic.
- **Aligns with queries** — queries that filter by the partition key are **single-partition** (cheap, fast); queries *without* it are **cross-partition** (fan out to all partitions — slower, more RUs).

The worst choice is a low-cardinality or skewed key (e.g., a status field, or a single tenant dominating traffic) → a **hot partition** that bottlenecks throughput. Partition key is effectively **immutable** for a container, so choose it carefully up front based on access patterns.

---

## Queries — point reads vs cross-partition

The cheapest, fastest operation is a **point read** — fetching one item by **id + partition key** (1 RU-ish, no query engine). Prefer it whenever you know both:

```csharp
var item = await container.ReadItemAsync<Order>(id, new PartitionKey(customerId));   // cheapest
```

For sets, use the **SQL-like query** API; **always include the partition key** when you can to keep queries single-partition:

```csharp
var query = new QueryDefinition("SELECT * FROM c WHERE c.status = @s")
    .WithParameter("@s", "Pending");
var iterator = container.GetItemQueryIterator<Order>(query,
    requestOptions: new QueryRequestOptions { PartitionKey = new PartitionKey(customerId) });
while (iterator.HasMoreResults) foreach (var o in await iterator.ReadNextAsync()) { /* ... */ }
```

A query **without** a partition key is **cross-partition** — it fans out, costing more RUs and latency. Model your data and partition key so the *common* queries are single-partition; cross-partition queries should be the exception.

---

## Consistency levels

Cosmos offers **five tunable consistency levels** (a spectrum from strongest to fastest), letting you trade consistency for latency/availability/cost:

| Level | Guarantee | Trade-off |
|---|---|---|
| **Strong** | linearizable (always latest) | highest latency/cost, limits multi-region |
| **Bounded staleness** | lag bounded by time/versions | strong-ish, more available |
| **Session** (default) | consistent within a client session | great balance — read-your-writes per session |
| **Consistent prefix** | reads never see out-of-order writes | weaker |
| **Eventual** | converges eventually | lowest latency/cost, weakest |

**Session** (the default) is the common sweet spot — you read your own writes within a session, at good performance. Choose **Strong** only when you truly need it (it costs latency and constrains global distribution); **Eventual** for max performance where staleness is acceptable. This tunability is a Cosmos hallmark — most databases don't let you dial consistency per need.

---

## Change feed

Cosmos exposes a **change feed** — a persistent, ordered log of inserts/updates per container — that you can process to react to data changes (build materialized views, sync to another store, trigger workflows). It powers the **Cosmos DB trigger** in Azure Functions ([03-Functions.md](03-Functions.md)):

```csharp
[Function("OnOrderChanged")]
public void Run([CosmosDBTrigger("ShopDb", "orders",
    Connection = "Cosmos", LeaseContainerName = "leases")] IReadOnlyList<Order> changes) {
    foreach (var order in changes) { /* react to each changed item */ }
}
```

The change feed is the event-sourcing/CDC primitive of Cosmos — process changes reliably (with leases tracking progress) to drive downstream processing, denormalized views, or integrations.

---

## Common gotchas

### Poor partition key → hot partition

A low-cardinality/skewed key concentrates load on one partition, capping throughput and causing 429s. Choose a **high-cardinality, evenly-distributed** key aligned with queries — and you can't change it later.

### Cross-partition queries everywhere

Queries without the partition key fan out across all partitions (high RU, high latency). Design data so common queries are **single-partition**; reserve cross-partition for rare cases.

### Treating it like SQL

Cosmos is NoSQL — no joins across containers, RU-based cost, denormalization over normalization. Model for your **access patterns** (often denormalized/embedded), not relational normalization ([Ch05](../05-EFCore/README.md)).

### Ignoring RU costs / 429s

Every operation costs RUs; exceeding provisioned RU/s yields **429 (rate-limited)** responses. Monitor RU consumption, use autoscale, and the SDK's retry handles 429s with backoff ([Ch11](../11-Resilience/README.md)) — but design to stay within budget.

### Over-using Strong consistency

Strong consistency adds latency and constrains multi-region writes. Use **Session** (default) unless you genuinely need stronger guarantees.

---

## Summary

- **Azure Cosmos DB** is a globally-distributed, schema-flexible **NoSQL** document store with low latency, elastic **horizontal scale**, and multi-region replication; client is `Microsoft.Azure.Cosmos` with **managed identity** ([01-OverviewAndIdentity.md](01-OverviewAndIdentity.md)). Cost/throughput are metered in **Request Units (RU/s)**.
- The critical design choice is the **partition key** — pick **high-cardinality, evenly-distributed**, and **query-aligned** so common queries are **single-partition** (cheap/fast); a poor key creates a **hot partition** and it's effectively immutable.
- Prefer **point reads** (id + partition key — cheapest); for queries always include the partition key when possible (cross-partition queries fan out, costing more RUs/latency).
- Cosmos offers **five tunable consistency levels** (Strong → Eventual; **Session** is the default sweet spot) and a **change feed** (ordered change log powering Functions triggers, materialized views, CDC). Model for access patterns (denormalize), watch **RU/429s**.

→ Next: [07-KeyVault.md](07-KeyVault.md)
