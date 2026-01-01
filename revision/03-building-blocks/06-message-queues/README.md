# Message Queues

Message queues enable asynchronous communication between services, improving reliability and scalability.

## Table of Contents
- [Why Message Queues?](#why-message-queues)
- [Messaging Patterns](#messaging-patterns)
- [Message Queue Features](#message-queue-features)
- [Popular Message Queues](#popular-message-queues)
- [Kafka Deep Dive](#kafka-deep-dive)
- [Key Takeaways](#key-takeaways)

## Why Message Queues?

### Synchronous vs Asynchronous

```
Synchronous:
┌────────┐     ┌────────┐     ┌────────┐
│ Client │────▶│  API   │────▶│ Email  │
│        │◀────│        │◀────│ Service│
└────────┘     └────────┘     └────────┘
                              (slow, blocks)

Asynchronous:
┌────────┐     ┌────────┐     ┌────────┐     ┌────────┐
│ Client │────▶│  API   │────▶│ Queue  │────▶│ Email  │
│        │◀────│        │     │        │     │ Service│
└────────┘     └────────┘     └────────┘     └────────┘
              (returns fast)  (buffered)    (processes later)
```

### Benefits

| Benefit | Description |
|---------|-------------|
| Decoupling | Services don't need to know about each other |
| Reliability | Messages persist if consumer is down |
| Scalability | Add consumers to process faster |
| Peak handling | Buffer spikes, process at steady rate |
| Async processing | Don't block on slow operations |

## Messaging Patterns

### Point-to-Point (Queue)

One message delivered to one consumer.

```
┌────────────┐     ┌─────────────────┐     ┌────────────┐
│  Producer  │────▶│      Queue      │────▶│  Consumer  │
└────────────┘     │  ┌───┬───┬───┐  │     └────────────┘
                   │  │ 1 │ 2 │ 3 │  │
                   │  └───┴───┴───┘  │
                   └─────────────────┘

Each message processed by exactly one consumer
Use case: Task distribution, work queues
```

### Publish-Subscribe (Topic)

One message delivered to all subscribers.

```
┌────────────┐     ┌─────────────────┐     ┌────────────┐
│  Publisher │────▶│      Topic      │────▶│Subscriber 1│
└────────────┘     │                 │────▶│Subscriber 2│
                   │                 │────▶│Subscriber 3│
                   └─────────────────┘     └────────────┘

Each message received by all subscribers
Use case: Events, notifications, broadcasts
```

### Fan-Out

Combine both patterns.

```
┌────────────┐     ┌─────────────────┐
│  Producer  │────▶│     Exchange    │
└────────────┘     └───────┬─────────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌─────────┐  ┌─────────┐  ┌─────────┐
        │ Queue 1 │  │ Queue 2 │  │ Queue 3 │
        └────┬────┘  └────┬────┘  └────┬────┘
             │            │            │
        ┌────▼────┐  ┌────▼────┐  ┌────▼────┐
        │Consumer │  │Consumer │  │Consumer │
        └─────────┘  └─────────┘  └─────────┘
```

## Message Queue Features

### Delivery Guarantees

```
┌────────────────────────────────────────────────────────┐
│              Delivery Guarantees                       │
├────────────────────────────────────────────────────────┤
│                                                        │
│  At-most-once:                                         │
│  - Message may be lost                                │
│  - No duplicates                                       │
│  - Use: Metrics, logs (loss acceptable)              │
│                                                        │
│  At-least-once:                                        │
│  - Message never lost                                 │
│  - May have duplicates                                │
│  - Use: Most applications (with idempotency)         │
│                                                        │
│  Exactly-once:                                         │
│  - Never lost, no duplicates                          │
│  - Hard to achieve, expensive                         │
│  - Use: Financial transactions                        │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Ordering

```
Strict ordering (per partition):
Producer: A → B → C → D
Consumer: A → B → C → D

Best-effort ordering:
Producer: A → B → C → D
Consumer: A → C → B → D (possible with parallel consumers)
```

### Dead Letter Queue

```
Main Queue ────┐                    ┌──── DLQ
               │                    │
               ▼                    ▼
          ┌─────────┐         ┌─────────┐
          │ Message │──fail──▶│ Failed  │
          │         │ ×3      │ Message │
          └─────────┘         └─────────┘
               │
               ▼
          Processed

After N failures, move to DLQ for investigation
```

## Popular Message Queues

### Comparison

| Feature | RabbitMQ | Kafka | SQS | Redis Streams |
|---------|----------|-------|-----|---------------|
| Model | Queue | Log | Queue | Log |
| Ordering | Per queue | Per partition | Best effort | Per stream |
| Retention | Until consumed | Time/size based | 14 days max | Configurable |
| Throughput | Medium | Very high | Medium | High |
| Latency | Low | Medium | Medium | Very low |
| Complexity | Medium | High | Low | Low |

### RabbitMQ

Traditional message broker with flexible routing.

```
┌────────────────────────────────────────────────────────┐
│                     RabbitMQ                           │
│                                                        │
│  Producer ──▶ Exchange ──▶ Queue ──▶ Consumer         │
│                   │                                    │
│            Routing rules:                              │
│            - Direct (exact match)                     │
│            - Topic (pattern match)                    │
│            - Fanout (broadcast)                       │
│            - Headers (header match)                   │
│                                                        │
└────────────────────────────────────────────────────────┘

Use cases:
- Task queues
- RPC
- Complex routing
```

### Amazon SQS

Fully managed, simple queue service.

```python
import boto3

sqs = boto3.client('sqs')

# Send message
sqs.send_message(
    QueueUrl='https://sqs.../my-queue',
    MessageBody='{"task": "process_image", "id": 123}'
)

# Receive messages
response = sqs.receive_message(
    QueueUrl='https://sqs.../my-queue',
    MaxNumberOfMessages=10
)

# Delete after processing
sqs.delete_message(
    QueueUrl='https://sqs.../my-queue',
    ReceiptHandle=message['ReceiptHandle']
)
```

## Kafka Deep Dive

Kafka is a distributed streaming platform for high-throughput, fault-tolerant messaging.

### Architecture

```
┌────────────────────────────────────────────────────────┐
│                      Kafka Cluster                     │
│                                                        │
│  Topic: orders                                         │
│  ┌─────────────┬─────────────┬─────────────┐          │
│  │ Partition 0 │ Partition 1 │ Partition 2 │          │
│  │ ┌─┬─┬─┬─┬─┐ │ ┌─┬─┬─┬─┐   │ ┌─┬─┬─┬─┬─┐ │          │
│  │ │0│1│2│3│4│ │ │0│1│2│3│   │ │0│1│2│3│4│ │          │
│  │ └─┴─┴─┴─┴─┘ │ └─┴─┴─┴─┘   │ └─┴─┴─┴─┴─┘ │          │
│  └─────────────┴─────────────┴─────────────┘          │
│        │              │              │                 │
│        ▼              ▼              ▼                 │
│    Broker 1       Broker 2       Broker 3             │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Key Concepts

```
Topic: Category of messages (like a table)
Partition: Ordered, immutable sequence
Offset: Position in partition
Consumer Group: Set of consumers that share work
```

### Partitioning

```python
# Messages with same key go to same partition
producer.send('orders', key='user-123', value=order_data)

# Ordering guaranteed within partition
Partition 0: user-123's orders in order
Partition 1: user-456's orders in order
```

### Consumer Groups

```
Topic with 3 partitions, Consumer Group with 3 consumers:

┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Partition 0 ────────────▶ Consumer A                  │
│  Partition 1 ────────────▶ Consumer B                  │
│  Partition 2 ────────────▶ Consumer C                  │
│                                                         │
│  Each partition consumed by one consumer in group      │
│  Scale by adding partitions + consumers                │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Kafka Example

```python
from kafka import KafkaProducer, KafkaConsumer

# Producer
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode()
)

producer.send('orders', key=b'user-123', value={
    'order_id': 'ord-456',
    'amount': 99.99
})

# Consumer
consumer = KafkaConsumer(
    'orders',
    bootstrap_servers=['localhost:9092'],
    group_id='order-processor',
    auto_offset_reset='earliest'
)

for message in consumer:
    process_order(message.value)
    # Offset committed automatically (or manually)
```

### Kafka vs Traditional Queues

| Aspect | Kafka | RabbitMQ/SQS |
|--------|-------|--------------|
| Model | Log (append-only) | Queue (delete after consume) |
| Retention | Keep messages | Remove after consume |
| Replay | Yes (seek to offset) | No |
| Throughput | Millions/sec | Thousands/sec |
| Ordering | Per partition | Per queue |
| Use case | Event streaming | Task queues |

## Key Takeaways

1. **Queues for tasks** (one consumer), **topics for events** (many subscribers)
2. **At-least-once** is the sweet spot (with idempotency)
3. **Kafka for high throughput** and event streaming
4. **RabbitMQ for routing** flexibility
5. **SQS for simplicity** (managed, no operations)
6. **Dead letter queues** for failure handling

## Practice Questions

1. Design a notification system using message queues.
2. How would you handle message ordering across partitions?
3. When would you choose Kafka over RabbitMQ?
4. How do you ensure exactly-once processing?

## Further Reading

- [Kafka: The Definitive Guide](https://www.confluent.io/resources/kafka-the-definitive-guide/)
- [RabbitMQ Tutorials](https://www.rabbitmq.com/getstarted.html)
- [AWS SQS Developer Guide](https://docs.aws.amazon.com/sqs/)

---

Next: [Blob Storage](../07-blob-storage/README.md)
