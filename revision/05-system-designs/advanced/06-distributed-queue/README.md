# Design a Distributed Message Queue

Design a distributed message queue like Kafka or RabbitMQ.

## 1. Requirements

### Functional Requirements
- Publish messages to topics
- Subscribe and consume messages
- Message ordering (within partition)
- At-least-once delivery
- Message retention
- Consumer groups

### Non-Functional Requirements
- High throughput (millions msgs/sec)
- Low latency (<10ms publish)
- Durability (no message loss)
- Horizontal scalability
- Fault tolerance

### Capacity Estimation

```
Messages: 1M messages/sec
Average message size: 1 KB
Data rate: 1 GB/sec
Retention: 7 days
Storage: 1 GB/s × 7 × 86400 = 600 TB

Replication factor: 3
Total storage: 600 TB × 3 = 1.8 PB
```

## 2. High-Level Design

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌───────────┐           ┌───────────┐                        │
│  │ Producer  │           │ Producer  │                        │
│  └─────┬─────┘           └─────┬─────┘                        │
│        │                       │                               │
│        └───────────┬───────────┘                               │
│                    ▼                                           │
│             ┌─────────────┐                                   │
│             │   Router    │                                   │
│             └──────┬──────┘                                   │
│                    │                                           │
│    ┌───────────────┼───────────────┐                          │
│    ▼               ▼               ▼                          │
│ ┌──────────┐  ┌──────────┐  ┌──────────┐                     │
│ │ Broker 1 │  │ Broker 2 │  │ Broker 3 │                     │
│ │┌────────┐│  │┌────────┐│  │┌────────┐│                     │
│ ││Topic A ││  ││Topic A ││  ││Topic A ││                     │
│ ││Part 0  ││  ││Part 1  ││  ││Part 2  ││                     │
│ │└────────┘│  │└────────┘│  │└────────┘│                     │
│ └──────────┘  └──────────┘  └──────────┘                     │
│        │               │               │                       │
│        └───────────────┼───────────────┘                       │
│                    ▼                                           │
│  ┌───────────┐           ┌───────────┐                        │
│  │ Consumer  │           │ Consumer  │                        │
│  │  Group A  │           │  Group B  │                        │
│  └───────────┘           └───────────┘                        │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 3. Topic and Partitions

```
┌────────────────────────────────────────────────────────────────┐
│                Topic and Partitions                            │
│                                                                │
│  Topic: orders                                                 │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  Partition 0           Partition 1           Partition 2  │ │
│  │  ┌───┬───┬───┬───┐    ┌───┬───┬───┐        ┌───┬───┐    │ │
│  │  │ 0 │ 1 │ 2 │ 3 │    │ 0 │ 1 │ 2 │        │ 0 │ 1 │    │ │
│  │  └───┴───┴───┴───┘    └───┴───┴───┘        └───┴───┘    │ │
│  │      Broker 1             Broker 2            Broker 3    │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  Partitioning:                                                 │
│  - Key-based: hash(key) % num_partitions                     │
│  - Round-robin: if no key specified                          │
│                                                                │
│  Ordering:                                                     │
│  - Guaranteed within partition                                │
│  - Not guaranteed across partitions                          │
│                                                                │
│  Example: order_id as key                                     │
│  → All events for same order go to same partition            │
│  → Preserves order for that order                            │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 4. Storage Layer

```
┌────────────────────────────────────────────────────────────────┐
│                 Log-Based Storage                              │
│                                                                │
│  Each partition = append-only log                             │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Segment Files                                           │  │
│  │                                                          │  │
│  │  Partition 0:                                            │  │
│  │  segment_0.log    [0-999]     (closed, immutable)       │  │
│  │  segment_1000.log [1000-1999] (closed, immutable)       │  │
│  │  segment_2000.log [2000-2342] (active, writable)        │  │
│  │                                                          │  │
│  │  Each segment has:                                       │  │
│  │  - .log file (messages)                                  │  │
│  │  - .index file (offset → position mapping)              │  │
│  │  - .timeindex (timestamp → offset mapping)              │  │
│  │                                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  Message format:                                               │
│  ┌────────┬────────┬─────┬───────┬─────────┬──────────────┐  │
│  │ Offset │ Size   │ CRC │ Magic │ Attrs   │ Key + Value  │  │
│  │ 8 bytes│ 4 bytes│ 4 B │ 1 B   │ 1 B     │ variable     │  │
│  └────────┴────────┴─────┴───────┴─────────┴──────────────┘  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Storage Implementation

```python
class Partition:
    def __init__(self, topic, partition_id, data_dir):
        self.topic = topic
        self.partition_id = partition_id
        self.data_dir = data_dir
        self.segments = []
        self.active_segment = None
        self.next_offset = 0
        self.max_segment_size = 1 * 1024 * 1024 * 1024  # 1 GB

    def append(self, messages):
        """Append messages to partition"""
        if self.active_segment is None or \
           self.active_segment.size > self.max_segment_size:
            self._roll_segment()

        offsets = []
        for msg in messages:
            offset = self.next_offset
            self.active_segment.append(offset, msg)
            offsets.append(offset)
            self.next_offset += 1

        return offsets

    def read(self, offset, max_messages=100):
        """Read messages starting from offset"""
        segment = self._find_segment(offset)
        if segment is None:
            return []

        return segment.read(offset, max_messages)

    def _roll_segment(self):
        """Create new segment"""
        if self.active_segment:
            self.active_segment.close()

        new_segment = Segment(
            self.data_dir,
            self.next_offset
        )
        self.segments.append(new_segment)
        self.active_segment = new_segment

class Segment:
    def __init__(self, data_dir, base_offset):
        self.base_offset = base_offset
        self.log_path = f"{data_dir}/{base_offset}.log"
        self.index_path = f"{data_dir}/{base_offset}.index"
        self.log_file = open(self.log_path, 'ab')
        self.index = SparseIndex()
        self.size = 0

    def append(self, offset, message):
        """Append message to segment"""
        position = self.log_file.tell()

        # Write message
        data = self._serialize(offset, message)
        self.log_file.write(data)
        self.size += len(data)

        # Update index (sparse - every N messages)
        if (offset - self.base_offset) % 4096 == 0:
            self.index.add(offset, position)

    def read(self, offset, max_messages):
        """Read messages from segment"""
        # Find position using index
        position = self.index.find_position(offset)

        messages = []
        with open(self.log_path, 'rb') as f:
            f.seek(position)
            while len(messages) < max_messages:
                msg = self._read_message(f)
                if msg is None:
                    break
                if msg['offset'] >= offset:
                    messages.append(msg)

        return messages
```

## 5. Replication

```
┌────────────────────────────────────────────────────────────────┐
│                    Replication                                 │
│                                                                │
│  ISR (In-Sync Replicas):                                      │
│  - Leader + followers that are caught up                      │
│  - Message is committed when written to all ISR              │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Topic: orders, Partition 0                              │  │
│  │                                                          │  │
│  │  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐│  │
│  │  │   Broker 1   │   │   Broker 2   │   │   Broker 3   ││  │
│  │  │   LEADER     │──▶│   FOLLOWER   │   │   FOLLOWER   ││  │
│  │  │   [0-100]    │   │   [0-100]    │   │   [0-98]     ││  │
│  │  └──────────────┘   └──────────────┘   └──────────────┘│  │
│  │                                                          │  │
│  │  ISR = {Broker1, Broker2}                               │  │
│  │  Broker3 is behind, not in ISR                          │  │
│  │                                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  Acks settings:                                                │
│  - acks=0: Fire and forget                                   │
│  - acks=1: Leader acknowledged                               │
│  - acks=all: All ISR acknowledged (most durable)            │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Replication Protocol

```python
class ReplicationManager:
    def __init__(self, broker_id):
        self.broker_id = broker_id
        self.leader_partitions = {}
        self.follower_partitions = {}

    async def replicate_as_leader(self, partition, messages):
        """Replicate messages to followers"""
        followers = self.get_followers(partition)

        # Send to all followers in parallel
        tasks = []
        for follower in followers:
            task = self.send_to_follower(follower, partition, messages)
            tasks.append(task)

        results = await asyncio.gather(*tasks, return_exceptions=True)

        # Check ISR acknowledgments
        acked = sum(1 for r in results if r is True)

        return acked >= self.min_isr

    async def replicate_as_follower(self, partition):
        """Fetch messages from leader"""
        leader = self.get_leader(partition)
        last_offset = self.get_last_offset(partition)

        while True:
            # Fetch from leader
            messages = await self.fetch_from_leader(
                leader, partition, last_offset
            )

            if messages:
                # Append to local log
                self.append_local(partition, messages)
                last_offset = messages[-1]['offset'] + 1

            await asyncio.sleep(0.1)  # Fetch interval
```

## 6. Consumer Groups

```
┌────────────────────────────────────────────────────────────────┐
│                  Consumer Groups                               │
│                                                                │
│  Group: payment-processors                                    │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Topic: payments (6 partitions)                         │  │
│  │                                                          │  │
│  │  P0  P1  P2  P3  P4  P5                                 │  │
│  │   │   │   │   │   │   │                                 │  │
│  │   └───┴───┘   │   └───┴───┘                             │  │
│  │       │       │       │                                  │  │
│  │       ▼       ▼       ▼                                  │  │
│  │  Consumer1  Consumer2  Consumer3                         │  │
│  │  (P0, P1)   (P2, P3)   (P4, P5)                         │  │
│  │                                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  Rebalancing triggers:                                        │
│  - Consumer joins                                             │
│  - Consumer leaves/crashes                                    │
│  - Partitions added                                           │
│                                                                │
│  Offset tracking:                                              │
│  - Each consumer group tracks offset per partition           │
│  - Stored in special __consumer_offsets topic               │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Consumer Implementation

```python
class Consumer:
    def __init__(self, group_id, topics):
        self.group_id = group_id
        self.topics = topics
        self.assigned_partitions = []
        self.offsets = {}

    async def poll(self, timeout_ms=1000):
        """Poll for new messages"""
        messages = []

        for partition in self.assigned_partitions:
            offset = self.offsets.get(partition, 0)

            # Fetch messages from broker
            batch = await self.fetch(partition, offset)
            messages.extend(batch)

            # Update local offset
            if batch:
                self.offsets[partition] = batch[-1]['offset'] + 1

        return messages

    async def commit(self):
        """Commit current offsets"""
        await self.coordinator.commit_offsets(
            self.group_id,
            self.offsets
        )

    async def subscribe(self):
        """Join consumer group"""
        # Register with group coordinator
        await self.coordinator.join_group(
            self.group_id,
            self.topics
        )

        # Receive partition assignment
        self.assigned_partitions = await self.coordinator.get_assignment()

        # Load committed offsets
        self.offsets = await self.coordinator.get_offsets(
            self.group_id,
            self.assigned_partitions
        )
```

## 7. Delivery Semantics

```
┌────────────────────────────────────────────────────────────────┐
│              Delivery Guarantees                               │
│                                                                │
│  At-most-once:                                                │
│  - Commit offset before processing                           │
│  - If crash, message may be lost                             │
│                                                                │
│  At-least-once:                                               │
│  - Commit offset after processing                            │
│  - If crash, message may be reprocessed                      │
│  - Consumer must be idempotent                               │
│                                                                │
│  Exactly-once:                                                │
│  - Transactional produce + consume                           │
│  - Atomic commit of offset + processing result              │
│  - Most complex, highest latency                             │
│                                                                │
│  Implementation (at-least-once):                              │
│                                                                │
│  while True:                                                   │
│      messages = consumer.poll()                               │
│      for msg in messages:                                     │
│          process(msg)  # Must be idempotent                  │
│      consumer.commit()  # After processing                   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 8. Key Takeaways

1. **Partitioned topics** - enable parallel processing
2. **Append-only log** - simple, fast, durable storage
3. **ISR replication** - balance durability and latency
4. **Consumer groups** - scalable consumption with rebalancing
5. **Offset tracking** - enable replay and exactly-once
6. **Zero-copy** - efficient data transfer

---

Next: [Payment System](../07-payment-system/README.md)
