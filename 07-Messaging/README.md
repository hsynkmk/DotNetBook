# Chapter 07 — Messaging

> In-process producer-consumer with `Channel<T>`, then out-of-process messaging with RabbitMQ, Azure Service Bus, and Kafka. The patterns that decouple services from each other.

**Prerequisites**: CSharpBook Chapter 08 (Concurrency). Familiarity with async.

**Time to read**: ~6-8 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-ChannelOfT.md](01-ChannelOfT.md) | In-process producer-consumer: bounded vs unbounded, completion, back-pressure. |
| [02-BackgroundQueues.md](02-BackgroundQueues.md) | Hosted services as queue workers; pattern for background jobs. |
| [03-RabbitMQ.md](03-RabbitMQ.md) | RabbitMQ.Client basics, exchanges, queues, bindings, acknowledgments. |
| [04-MassTransit.md](04-MassTransit.md) | The abstraction layer — consumers, sagas, retry, fault tolerance. |
| [05-AzureServiceBus.md](05-AzureServiceBus.md) | Azure.Messaging.ServiceBus — queues, topics, sessions, dead-letter. |
| [06-Kafka.md](06-Kafka.md) | Confluent.Kafka — producers, consumers, partitions, consumer groups. |
| [07-Patterns.md](07-Patterns.md) | Outbox pattern, idempotent consumers, ordered processing, sagas. |
| [Questions.md](Questions.md) | Drilling questions. |
| [Coding.md](Coding.md) | Build a Channel-based queue, then port to RabbitMQ. |

→ Begin: [01-ChannelOfT.md](01-ChannelOfT.md)
