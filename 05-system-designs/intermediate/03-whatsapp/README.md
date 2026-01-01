# Design WhatsApp

Design a real-time messaging application like WhatsApp.

## 1. Requirements

### Functional Requirements
- One-on-one messaging
- Group messaging (up to 256 members)
- Sent/delivered/read receipts
- Online/offline status
- Media sharing (images, videos, documents)
- End-to-end encryption

### Non-Functional Requirements
- Real-time message delivery (<100ms when online)
- High availability (99.99%)
- Message ordering guaranteed
- At-least-once delivery
- Scale: 2B users, 100B messages/day

### Capacity Estimation

```
Users: 2B total, 500M DAU
Messages: 100B/day ≈ 1.15M messages/sec
Average message: 100 bytes
Media: 10% messages have media (avg 100KB)

Storage (5 years):
Messages: 100B × 365 × 5 × 100 bytes = 18 PB
Media: 10B × 365 × 5 × 100 KB = 1.8 EB (with dedup: ~200 PB)

Bandwidth:
Messages: 1.15M × 100 bytes = 115 MB/s
Media: 115K × 100 KB = 11.5 GB/s
```

## 2. High-Level Design

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌────────┐     ┌──────────────┐     ┌────────────────────┐   │
│  │ Client │────▶│ Load Balancer│────▶│   Chat Servers     │   │
│  │        │◀────│              │◀────│   (WebSocket)      │   │
│  └────────┘     └──────────────┘     └─────────┬──────────┘   │
│                                                 │              │
│         ┌───────────────────────────────────────┼───────────┐ │
│         │                                       │           │ │
│         ▼                                       ▼           │ │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │ │
│  │   Redis     │  │  Message    │  │    User/Group       │ │ │
│  │ (Sessions)  │  │   Queue     │  │     Database        │ │ │
│  └─────────────┘  │  (Kafka)    │  └─────────────────────┘ │ │
│                   └──────┬──────┘                           │ │
│                          │                                   │ │
│                          ▼                                   │ │
│                   ┌─────────────┐                           │ │
│                   │  Message DB │                           │ │
│                   │ (Cassandra) │                           │ │
│                   └─────────────┘                           │ │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 3. Connection Management

### WebSocket Connections

```
┌────────────────────────────────────────────────────────┐
│              WebSocket Architecture                    │
│                                                        │
│  Each chat server handles ~500K concurrent connections│
│                                                        │
│  ┌──────────┐                     ┌──────────────┐   │
│  │  Client  │◀═══WebSocket═══════▶│ Chat Server  │   │
│  └──────────┘                     └──────┬───────┘   │
│                                          │           │
│                                          ▼           │
│                                   ┌─────────────┐    │
│                                   │   Redis     │    │
│                                   │  Session    │    │
│                                   │  Store      │    │
│                                   └─────────────┘    │
│                                                        │
│  Session data:                                         │
│  - user_id → server_id (which server has connection) │
│  - connection status (online/offline)                │
│  - last_seen timestamp                                │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Session Management

```python
class ChatServer:
    def __init__(self, server_id):
        self.server_id = server_id
        self.connections = {}  # user_id -> websocket

    async def handle_connect(self, user_id, websocket):
        self.connections[user_id] = websocket

        # Register in Redis
        redis.hset(f"session:{user_id}", {
            "server_id": self.server_id,
            "status": "online",
            "connected_at": now()
        })

        # Deliver pending messages
        await self.deliver_pending_messages(user_id)

        # Notify presence
        await self.broadcast_presence(user_id, "online")

    async def handle_disconnect(self, user_id):
        del self.connections[user_id]

        redis.hset(f"session:{user_id}", {
            "status": "offline",
            "last_seen": now()
        })

        await self.broadcast_presence(user_id, "offline")
```

## 4. Message Flow

### One-on-One Messaging

```
┌────────────────────────────────────────────────────────┐
│              Message Flow (User A → User B)           │
│                                                        │
│  ┌───────┐  1. Send   ┌───────────┐                  │
│  │User A │──────────▶│Chat Server│                   │
│  └───────┘           │    A      │                   │
│                      └─────┬─────┘                   │
│                            │ 2. Lookup B's server    │
│                            ▼                          │
│                      ┌───────────┐                   │
│                      │   Redis   │                   │
│                      └─────┬─────┘                   │
│                            │ 3. B is on Server B     │
│                            ▼                          │
│                      ┌───────────┐  4. Forward      │
│                      │Chat Server│──────────────────▶│
│                      │    B      │           ┌───────┐
│                      └───────────┘           │User B │
│                                              └───────┘
│                                                        │
│  5. If B offline: Store in Kafka → Cassandra         │
│  6. Return ack to A with message status              │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Message States

```
┌────────────────────────────────────────────────────────┐
│                Message States                          │
│                                                        │
│  ✓   SENT       - Server received message            │
│  ✓✓  DELIVERED  - Recipient's device received        │
│  ✓✓  READ       - Recipient opened chat (blue)       │
│                                                        │
│  Implementation:                                       │
│                                                        │
│  1. A sends message → Server stores, returns SENT    │
│  2. Server forwards to B → B acks → DELIVERED        │
│  3. B opens chat → sends read receipt → READ         │
│                                                        │
│  def send_message(from_user, to_user, content):      │
│      msg_id = generate_id()                          │
│      msg = {                                          │
│          'id': msg_id,                               │
│          'from': from_user,                          │
│          'to': to_user,                              │
│          'content': content,                         │
│          'status': 'SENT',                           │
│          'timestamp': now()                          │
│      }                                                │
│      store_message(msg)                              │
│      deliver_or_queue(msg)                           │
│      return msg_id                                   │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## 5. Group Messaging

```
┌────────────────────────────────────────────────────────┐
│               Group Message Flow                       │
│                                                        │
│  User sends to group:                                 │
│  1. Server receives message                           │
│  2. Lookup group members                              │
│  3. For each member:                                  │
│     - If online: forward via WebSocket               │
│     - If offline: queue for later delivery           │
│                                                        │
│  Optimization for large groups:                       │
│  - Use Kafka partition per group                     │
│  - Multiple consumers process deliveries            │
│                                                        │
│  ┌─────────┐                                         │
│  │  User   │────▶ Kafka Topic: group-{group_id}     │
│  └─────────┘                                         │
│                    │                                   │
│        ┌──────────┼──────────┐                       │
│        ▼          ▼          ▼                       │
│     Consumer   Consumer   Consumer                   │
│     (Member    (Member    (Member                    │
│      1-100)    101-200)   201-256)                   │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Group Data Model

```sql
CREATE TABLE groups (
    group_id BIGINT PRIMARY KEY,
    name VARCHAR(255),
    created_by BIGINT,
    created_at TIMESTAMP,
    member_count INT
);

CREATE TABLE group_members (
    group_id BIGINT,
    user_id BIGINT,
    role VARCHAR(20),  -- admin, member
    joined_at TIMESTAMP,
    PRIMARY KEY (group_id, user_id)
);

-- Cassandra for messages (write-optimized)
CREATE TABLE messages (
    conversation_id TEXT,  -- user1_user2 or group_id
    message_id TIMEUUID,
    sender_id BIGINT,
    content TEXT,
    media_url TEXT,
    status TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY (conversation_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);
```

## 6. Message Storage

### Cassandra Design

```
┌────────────────────────────────────────────────────────┐
│              Message Storage (Cassandra)              │
│                                                        │
│  Why Cassandra?                                       │
│  - Write-optimized (LSM tree)                        │
│  - Linear scalability                                 │
│  - High availability (no single point of failure)   │
│                                                        │
│  Partition key: conversation_id                      │
│  - Groups messages for same chat together            │
│  - Enables efficient range queries                   │
│                                                        │
│  Clustering key: message_id (TimeUUID)               │
│  - Sorted by time within partition                   │
│  - DESC order for "most recent first"               │
│                                                        │
│  Query patterns:                                      │
│  1. Get recent messages for chat                     │
│     SELECT * FROM messages                           │
│     WHERE conversation_id = 'abc'                    │
│     LIMIT 50;                                        │
│                                                        │
│  2. Get messages after timestamp (sync)              │
│     SELECT * FROM messages                           │
│     WHERE conversation_id = 'abc'                    │
│     AND message_id > minTimeuuid('2024-01-01');     │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## 7. Presence System

```python
class PresenceService:
    def __init__(self):
        self.redis = Redis()
        self.heartbeat_interval = 30  # seconds

    def update_presence(self, user_id, status):
        """Called on connect/disconnect and periodically"""
        key = f"presence:{user_id}"
        self.redis.hset(key, {
            "status": status,
            "last_active": now()
        })
        self.redis.expire(key, self.heartbeat_interval * 2)

    def get_presence(self, user_id):
        """Get user's online status"""
        key = f"presence:{user_id}"
        data = self.redis.hgetall(key)

        if not data:
            return {"status": "offline", "last_seen": "unknown"}

        return data

    def get_bulk_presence(self, user_ids):
        """Get presence for multiple users (contact list)"""
        pipe = self.redis.pipeline()
        for user_id in user_ids:
            pipe.hgetall(f"presence:{user_id}")

        return dict(zip(user_ids, pipe.execute()))
```

## 8. Media Handling

```
┌────────────────────────────────────────────────────────┐
│                 Media Upload Flow                      │
│                                                        │
│  1. Client requests upload URL                        │
│  2. Server generates presigned S3 URL                │
│  3. Client uploads directly to S3                    │
│  4. Client sends message with media reference        │
│  5. Recipient downloads from CDN                      │
│                                                        │
│  ┌────────┐  1. Get URL  ┌──────────┐               │
│  │ Client │─────────────▶│  Server  │               │
│  └───┬────┘              └────┬─────┘               │
│      │                        │                       │
│      │ 2. Presigned URL       │                       │
│      │◀───────────────────────┘                       │
│      │                                                 │
│      │ 3. Upload    ┌──────────┐                     │
│      │─────────────▶│    S3    │                     │
│      │              └──────────┘                     │
│      │                                                 │
│      │ 4. Send message with media_id                 │
│      │─────────────▶ Server ─────────────▶ Recipient │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## 9. End-to-End Encryption

```
┌────────────────────────────────────────────────────────┐
│            End-to-End Encryption (E2EE)               │
│                                                        │
│  Signal Protocol:                                     │
│                                                        │
│  1. Key Exchange (X3DH):                              │
│     - Each user has identity key pair                │
│     - Signed pre-keys uploaded to server             │
│     - Derive shared secret without server knowing    │
│                                                        │
│  2. Message Encryption:                               │
│     - Double Ratchet algorithm                        │
│     - New key for each message (forward secrecy)     │
│     - Server sees only ciphertext                    │
│                                                        │
│  Server's role:                                       │
│  - Store encrypted messages                          │
│  - Relay key exchange messages                       │
│  - Cannot decrypt any content                        │
│                                                        │
│  ┌────────┐  Encrypted  ┌──────────┐  Encrypted     │
│  │ User A │────────────▶│  Server  │───────────────▶│
│  └────────┘             └──────────┘     ┌──────────┐
│                                          │  User B  │
│  Only A and B have keys to decrypt       └──────────┘
│                                                        │
└────────────────────────────────────────────────────────┘
```

## 10. Key Takeaways

1. **WebSocket for real-time** - persistent connections for instant delivery
2. **Session routing** - Redis tracks which server has each user
3. **Message queue for reliability** - Kafka ensures no message loss
4. **Cassandra for messages** - write-optimized, scalable storage
5. **Presence with TTL** - heartbeat-based online status
6. **E2EE with Signal Protocol** - server cannot read messages

---

Next: [Notification System](../04-notification-system/README.md)
