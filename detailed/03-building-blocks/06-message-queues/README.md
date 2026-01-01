# Message Queues

Message queues enable asynchronous communication between services, providing decoupling, reliability, and scalability.

## Table of Contents
1. [Why Message Queues?](#why-message-queues)
2. [Messaging Patterns](#messaging-patterns)
3. [Queue vs Stream](#queue-vs-stream)
4. [Popular Solutions](#popular-solutions)
5. [Interview Questions](#interview-questions)

---

## Why Message Queues?

```
┌─────────────────────────────────────────────────────────────┐
│                  Why Message Queues?                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Synchronous (Tight Coupling):                              │
│  ┌────────┐     ┌────────┐     ┌────────┐                  │
│  │Service │────▶│Service │────▶│Service │                  │
│  │   A    │     │   B    │     │   C    │                  │
│  └────────┘     └────────┘     └────────┘                  │
│  If B is slow or down, A is blocked!                        │
│                                                              │
│  Asynchronous (Decoupled):                                  │
│  ┌────────┐     ┌─────────┐     ┌────────┐                 │
│  │Service │────▶│  Queue  │────▶│Service │                 │
│  │   A    │     └─────────┘     │   B    │                 │
│  └────────┘                     └────────┘                 │
│  A continues even if B is slow or down!                     │
│                                                              │
│  Benefits:                                                   │
│  • Decoupling: Services don't need to know each other      │
│  • Resilience: Messages saved even if consumer is down     │
│  • Scalability: Add more consumers for more throughput     │
│  • Load leveling: Absorb traffic spikes                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Messaging Patterns

```
┌─────────────────────────────────────────────────────────────┐
│                  Messaging Patterns                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Point-to-Point (Queue)                                  │
│     One producer, one consumer per message                  │
│     ┌──────────┐    ┌───────┐    ┌──────────┐              │
│     │ Producer │───▶│ Queue │───▶│ Consumer │              │
│     └──────────┘    └───────┘    └──────────┘              │
│     Message processed by exactly one consumer               │
│     Use: Task distribution, work queues                     │
│                                                              │
│  2. Publish-Subscribe (Pub/Sub)                             │
│     One producer, many consumers per message                │
│     ┌──────────┐    ┌───────┐    ┌──────────┐              │
│     │ Producer │───▶│ Topic │───▶│Consumer 1│              │
│     └──────────┘    │       │───▶│Consumer 2│              │
│                     └───────┘───▶│Consumer 3│              │
│                                  └──────────┘              │
│     All subscribers receive all messages                    │
│     Use: Events, notifications, data sync                  │
│                                                              │
│  3. Fan-Out                                                  │
│     One message to multiple queues                          │
│     ┌──────────┐    ┌─────────┐    ┌──────────┐            │
│     │ Producer │───▶│Exchange │───▶│ Queue A  │            │
│     └──────────┘    │         │───▶│ Queue B  │            │
│                     └─────────┘───▶│ Queue C  │            │
│                                    └──────────┘            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Queue vs Stream

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        Queue vs Stream                                      │
├────────────────────────────────────┬───────────────────────────────────────┤
│         Queue (RabbitMQ)           │         Stream (Kafka)                 │
├────────────────────────────────────┼───────────────────────────────────────┤
│ Message deleted after consume     │ Messages persisted (log)               │
│ Consumer tracks nothing           │ Consumer tracks offset                  │
│ At-most-once or at-least-once    │ Replay from any point                   │
│ Simple routing                    │ Partitions for parallelism             │
│ Lower throughput                  │ Higher throughput                       │
├────────────────────────────────────┼───────────────────────────────────────┤
│ Use for:                          │ Use for:                                │
│ • Task queues                     │ • Event streaming                       │
│ • RPC                             │ • Log aggregation                       │
│ • Simple pub/sub                  │ • Real-time analytics                   │
│                                    │ • Event sourcing                        │
└────────────────────────────────────┴───────────────────────────────────────┘
```

---

## Popular Solutions

| Solution | Type | Best For |
|----------|------|----------|
| **Kafka** | Stream | High throughput, event streaming |
| **RabbitMQ** | Queue | Traditional messaging, routing |
| **AWS SQS** | Queue | Managed, simple queuing |
| **Redis Streams** | Stream | Real-time, already using Redis |
| **Pulsar** | Both | Multi-tenancy, geo-replication |

---

## Interview Questions

### Basic
1. What is a message queue and why is it used?
2. Explain pub/sub messaging pattern.

### Intermediate
3. How would you ensure messages are processed exactly once?
4. When would you choose Kafka vs RabbitMQ?

### Advanced
5. Design a notification system using message queues.

---

[← Previous: Caching](../05-caching/README.md) | [Next: Blob Storage →](../07-blob-storage/README.md)
