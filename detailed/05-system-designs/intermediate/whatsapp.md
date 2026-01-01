# Design WhatsApp

## 1. Requirements

### Functional Requirements
- One-on-one messaging
- Group messaging (up to 256 members)
- Online/offline status (presence)
- Message delivery status (sent, delivered, read)
- End-to-end encryption
- Media sharing (images, videos, documents)
- Voice/video calls

### Non-Functional Requirements
- Real-time messaging (< 100ms latency)
- High availability
- Message ordering guaranteed
- At-least-once delivery
- Support billions of users
- Low battery and data usage on mobile

### Scale Estimates
- 2B users
- 500M DAU
- 100B messages per day
- Average message size: 100 bytes

```
┌─────────────────────────────────────────────────────────────┐
│                   Scale Summary                             │
├─────────────────────────────────────────────────────────────┤
│  Messages: 100B/day = 1.15M/second                         │
│  Connections: 500M concurrent WebSocket connections        │
│  Storage: 100B × 100B = 10 TB/day (messages only)         │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. High-Level Design

```
┌─────────────────────────────────────────────────────────────┐
│                  High-Level Architecture                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   User A                                         User B     │
│  ┌────────┐                                   ┌────────┐   │
│  │ Phone  │                                   │ Phone  │   │
│  └───┬────┘                                   └───┬────┘   │
│      │                                            │         │
│      │ WebSocket                      WebSocket  │         │
│      │                                            │         │
│      ▼                                            ▼         │
│  ┌────────────────────────────────────────────────────────┐│
│  │                   Gateway Servers                       ││
│  │   (Manage WebSocket connections, route messages)       ││
│  └────────────────────────┬───────────────────────────────┘│
│                           │                                 │
│                           ▼                                 │
│  ┌────────────────────────────────────────────────────────┐│
│  │                   Message Queue                         ││
│  │                    (Kafka)                              ││
│  └────────────────────────┬───────────────────────────────┘│
│                           │                                 │
│         ┌─────────────────┼─────────────────┐              │
│         ▼                 ▼                 ▼              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │   Chat      │  │  Presence   │  │   Media     │        │
│  │  Service    │  │  Service    │  │  Service    │        │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘        │
│         │                │                │                │
│         ▼                ▼                ▼                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │  Cassandra  │  │   Redis     │  │     S3      │        │
│  │  (Messages) │  │ (Presence)  │  │   (Media)   │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Real-Time Messaging

```
┌─────────────────────────────────────────────────────────────┐
│               Real-Time Message Delivery                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Message Flow (A sends to B):                               │
│                                                              │
│  ┌────────┐  1. Send    ┌───────────┐  2. Publish          │
│  │ User A │───────────▶ │  Gateway  │──────────────┐       │
│  └────────┘  WebSocket  │  Server   │              │       │
│                         └───────────┘              │       │
│                                                    ▼       │
│                                            ┌─────────────┐ │
│                                            │   Message   │ │
│                                            │   Queue     │ │
│                                            │   (Kafka)   │ │
│                                            └──────┬──────┘ │
│                                                   │        │
│    3. Store message                              │        │
│       ┌───────────────────────────────────────────┘        │
│       │                                                     │
│       ▼                                                     │
│  ┌───────────┐  4. Look up B's gateway                     │
│  │ Cassandra │─────────────────────────────────────┐       │
│  └───────────┘                                     │       │
│                                                    ▼       │
│  ┌────────┐  5. Push     ┌───────────┐       ┌─────────┐  │
│  │ User B │◀────────────│  Gateway  │◀──────│ Session │  │
│  └────────┘  WebSocket  │  Server   │       │ Service │  │
│                         └───────────┘       └─────────┘  │
│                                                              │
│  Session Service: Maps user_id → gateway_server_id         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Connection Management

```
┌─────────────────────────────────────────────────────────────┐
│                Connection Management                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Each gateway server handles ~500K connections             │
│                                                              │
│  Connection Registry (Redis):                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Key: user:{user_id}:connection                        │ │
│  │  Value: {                                               │ │
│  │    "gateway_id": "gateway-1",                          │ │
│  │    "connected_at": 1640000000,                         │ │
│  │    "last_seen": 1640000100                             │ │
│  │  }                                                      │ │
│  │  TTL: 5 minutes (refreshed by heartbeat)               │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Handling offline users:                                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  1. Message arrives for offline user                   │ │
│  │  2. Store in "pending messages" queue                  │ │
│  │  3. Send push notification                             │ │
│  │  4. When user comes online, deliver pending messages  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Message Delivery Status

```
┌─────────────────────────────────────────────────────────────┐
│               Message Delivery Status                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Status Flow:                                                │
│                                                              │
│  ┌──────┐    ┌───────────┐    ┌───────────┐    ┌────────┐  │
│  │ Sent │───▶│ Delivered │───▶│   Read    │    │  ✓✓   │  │
│  │  ✓   │    │    ✓✓     │    │   ✓✓     │    │ (blue) │  │
│  └──────┘    └───────────┘    └───────────┘    └────────┘  │
│                                                              │
│  Implementation:                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. SENT: Message stored in server                     │ │
│  │     - ACK to sender                                    │ │
│  │     - Single checkmark                                 │ │
│  │                                                         │ │
│  │  2. DELIVERED: Message received by recipient's device  │ │
│  │     - Recipient sends delivery ACK                     │ │
│  │     - Server forwards to sender                        │ │
│  │     - Double checkmark (grey)                          │ │
│  │                                                         │ │
│  │  3. READ: Recipient opened the chat                    │ │
│  │     - Recipient sends read ACK                         │ │
│  │     - Server forwards to sender                        │ │
│  │     - Double checkmark (blue)                          │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Message Schema:                                            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  message_id      │ UUID                                 │ │
│  │  sender_id       │ BIGINT                               │ │
│  │  receiver_id     │ BIGINT                               │ │
│  │  content         │ BLOB (encrypted)                     │ │
│  │  status          │ ENUM(sent, delivered, read)         │ │
│  │  sent_at         │ TIMESTAMP                            │ │
│  │  delivered_at    │ TIMESTAMP                            │ │
│  │  read_at         │ TIMESTAMP                            │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Group Messaging

```
┌─────────────────────────────────────────────────────────────┐
│                   Group Messaging                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Group message delivery:                                    │
│                                                              │
│  ┌────────┐    ┌───────────────────────────────────────┐   │
│  │Sender  │    │             Group                      │   │
│  │        │───▶│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐     │   │
│  └────────┘    │  │User1│ │User2│ │User3│ │...  │     │   │
│                │  └─────┘ └─────┘ └─────┘ └─────┘     │   │
│                └───────────────────────────────────────┘   │
│                                                              │
│  Approach 1: Fan-out on write (small groups)               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  For each member in group:                             │ │
│  │    - Create message copy                               │ │
│  │    - Deliver individually                              │ │
│  │                                                         │ │
│  │  ✓ Simple delivery tracking per user                  │ │
│  │  ✗ Storage overhead for large groups                  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Approach 2: Store once, reference (large groups)          │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  - Store message once with group_id                    │ │
│  │  - Separate delivery status per member                │ │
│  │  - Members query group messages                       │ │
│  │                                                         │ │
│  │  ✓ Storage efficient                                  │ │
│  │  ✗ More complex delivery tracking                     │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Group Data Model:                                          │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  groups                                                 │ │
│  │  ├── group_id (PK)                                     │ │
│  │  ├── name                                               │ │
│  │  ├── created_by                                        │ │
│  │  └── created_at                                        │ │
│  │                                                         │ │
│  │  group_members                                          │ │
│  │  ├── group_id (partition key)                          │ │
│  │  ├── user_id                                            │ │
│  │  ├── role (admin, member)                              │ │
│  │  └── joined_at                                          │ │
│  │                                                         │ │
│  │  group_messages                                         │ │
│  │  ├── group_id (partition key)                          │ │
│  │  ├── message_id                                        │ │
│  │  ├── sender_id                                         │ │
│  │  ├── content                                            │ │
│  │  └── sent_at (clustering key)                          │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Presence (Online Status)

```
┌─────────────────────────────────────────────────────────────┐
│                   Presence System                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Tracking online status:                                    │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Heartbeat mechanism:                                   │ │
│  │                                                         │ │
│  │  Client → Server: heartbeat every 30 seconds          │ │
│  │  Server: Update last_seen in Redis                     │ │
│  │                                                         │ │
│  │  Redis key: presence:{user_id}                         │ │
│  │  Value: { "last_seen": 1640000000, "status": "online" }│ │
│  │  TTL: 60 seconds (auto-offline if no heartbeat)       │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Notifying contacts about status changes:                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Approach 1: Pull model (recommended)                  │ │
│  │  - Client queries presence when opening chat          │ │
│  │  - Reduces unnecessary updates                        │ │
│  │                                                         │ │
│  │  Approach 2: Push model                                │ │
│  │  - Subscribe to contacts' status                      │ │
│  │  - Push updates when status changes                   │ │
│  │  - More real-time but higher overhead                 │ │
│  │                                                         │ │
│  │  WhatsApp: Hybrid                                      │ │
│  │  - Push for active chats                              │ │
│  │  - Pull for chat list                                 │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  "Last seen" display:                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Status         │ Display                              │ │
│  │  ───────────────┼────────────────────────────────────  │ │
│  │  Now online     │ "online"                             │ │
│  │  < 5 minutes    │ "last seen recently"                │ │
│  │  < 1 hour       │ "last seen X minutes ago"           │ │
│  │  < 24 hours     │ "last seen today at HH:MM"          │ │
│  │  Older          │ "last seen DATE at HH:MM"           │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. End-to-End Encryption

```
┌─────────────────────────────────────────────────────────────┐
│                End-to-End Encryption                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Signal Protocol (used by WhatsApp):                       │
│                                                              │
│  Key Exchange:                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  User A                              User B            │ │
│  │  ┌─────────────────────────────────────────────────┐  │ │
│  │  │ Identity Key (long-term)                         │  │ │
│  │  │ Signed Pre-key (medium-term)                     │  │ │
│  │  │ One-time Pre-key (single use)                    │  │ │
│  │  └─────────────────────────────────────────────────┘  │ │
│  │                                                         │ │
│  │  1. A fetches B's public keys from server             │ │
│  │  2. A generates session key using X3DH                │ │
│  │  3. A encrypts message with session key               │ │
│  │  4. Server stores encrypted blob (can't read)         │ │
│  │  5. B decrypts using own private keys                 │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Server's role:                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  ✓ Store encrypted messages                            │ │
│  │  ✓ Store public keys                                   │ │
│  │  ✓ Route messages                                      │ │
│  │  ✗ Cannot read message content                        │ │
│  │  ✗ Cannot decrypt messages                            │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Group encryption:                                          │
│  • Sender key: Each member has a sender key               │
│  • Message encrypted once with sender key                 │
│  • Sender key shared with each member (using pairwise)   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. Summary

| Component | Choice | Reason |
|-----------|--------|--------|
| **Connection** | WebSocket | Real-time bidirectional |
| **Message Queue** | Kafka | Durability, ordering |
| **Message Store** | Cassandra | Write-heavy, partitioned by conversation |
| **Presence** | Redis | Fast TTL-based expiry |
| **Media** | S3 + CDN | Scalable blob storage |
| **Encryption** | Signal Protocol | End-to-end encryption |

---

[Back to System Designs](../README.md)
