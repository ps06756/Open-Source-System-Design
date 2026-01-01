# Deep Dive

Going deeper into specific components. This is where you demonstrate technical depth.

## Choosing What to Deep Dive

### Pick Components That Are:

1. **Core to the system** - the main functionality
2. **Technically challenging** - scale, consistency, latency
3. **Your strength** - areas you know well

### Common Deep Dive Topics

| System Type | Typical Deep Dives |
|-------------|-------------------|
| Social Media | Feed generation, notification fanout |
| E-Commerce | Inventory management, order processing |
| Messaging | Message delivery, presence |
| Storage | Sync algorithm, conflict resolution |
| Search | Indexing, ranking |

## How to Structure a Deep Dive

### 1. State the Problem
"Let me dive into how we generate the home feed. The challenge is that users follow many accounts, and we need to show a ranked feed in <200ms."

### 2. Discuss Options
"There are two main approaches: fan-out on write and fan-out on read. Let me explain both..."

### 3. Make a Decision
"I'll go with a hybrid approach because..."

### 4. Explain Implementation
"Here's how it works: When a user posts, we..."

### 5. Address Edge Cases
"For celebrities with millions of followers, we handle them differently..."

## Example Deep Dives

### Timeline Generation (Twitter)

**Problem:** Show personalized feed from followed users

**Options:**
```
1. Fan-out on Write (Push)
   - When user posts, push to all followers' timelines
   - Pro: Fast reads
   - Con: Slow writes for popular users

2. Fan-out on Read (Pull)
   - When user views feed, fetch from all followed users
   - Pro: Fast writes
   - Con: Slow reads

3. Hybrid
   - Push for regular users
   - Pull for celebrities (>10K followers)
   - Merge at read time
```

**Implementation:**
```
When user posts:
1. Store tweet in database
2. If user has < 10K followers:
   - Get all follower IDs
   - For each follower, add tweet_id to their feed cache
3. If user has >= 10K followers:
   - Don't fan-out (will be pulled at read time)

When user reads feed:
1. Get pre-computed feed from cache
2. Get celebrity tweets they follow
3. Merge, rank, and return top N
```

**Edge Cases:**
- New user with no feed: Show popular/trending content
- Celebrity unfollows: Remove from their pull list
- Timeline too old: Rebuild from scratch

### Message Delivery (WhatsApp)

**Problem:** Deliver messages in real-time with receipts

**Options:**
```
1. Polling
   - Client periodically checks for new messages
   - Pro: Simple
   - Con: High latency, wasteful

2. Long Polling
   - Client opens connection, server holds until message arrives
   - Pro: Lower latency
   - Con: Still has reconnection overhead

3. WebSocket
   - Persistent bidirectional connection
   - Pro: True real-time
   - Con: Connection management complexity
```

**Implementation:**
```
Message flow (A sends to B):
1. A sends message via WebSocket to Chat Server A
2. Server A stores message, generates message_id
3. Server A looks up B's connection in Redis
4. If B online: Forward to Chat Server B → B's device
5. If B offline: Store in message queue
6. B comes online: Deliver queued messages
7. B's device acks: Update status to "delivered"
8. B reads message: Send read receipt to A
```

**Edge Cases:**
- Both users on same server: Direct delivery
- User switches servers: Session handoff
- Message during reconnect: Idempotent delivery with message_id

### Inventory Management (E-Commerce)

**Problem:** Prevent overselling while handling high concurrency

**Options:**
```
1. Pessimistic Locking
   - Lock inventory row during checkout
   - Pro: Simple, correct
   - Con: Blocks concurrent purchases

2. Optimistic Locking
   - Check-and-set with version number
   - Pro: Higher throughput
   - Con: Retries on conflict

3. Reserve Pattern
   - Create temporary reservation
   - Pro: Better UX (items don't disappear)
   - Con: Need to handle expiration
```

**Implementation:**
```sql
-- Optimistic locking approach
UPDATE inventory
SET quantity = quantity - 1,
    version = version + 1
WHERE product_id = ?
  AND quantity >= 1
  AND version = ?

-- If affected_rows = 0: retry or fail
```

**Edge Cases:**
- Flash sale: Pre-warm cache, queue requests
- Cart abandonment: Release reserved inventory after timeout
- Split shipment: Reserve from multiple warehouses

## Deep Dive Checklist

For each component, cover:

- [ ] **What** is the problem?
- [ ] **Options** considered
- [ ] **Why** this approach?
- [ ] **How** it works (step by step)
- [ ] **Edge cases** and how to handle them
- [ ] **Trade-offs** made

## Signaling to the Interviewer

### Checking In
- "Should I go deeper on this, or move to another component?"
- "I can talk about [X] or [Y] next - which would you prefer?"
- "Does this make sense so far?"

### Pivoting
If the interviewer seems uninterested:
- "Let me switch to something more interesting..."
- "The more challenging part is actually..."

### Time Management
- If running low on time, summarize: "At a high level, we'd use [X] approach for [Y] reason"
- Don't get stuck on one component for too long

## Common Mistakes

### 1. Going Too Deep Too Soon
Don't dive into implementation details before showing the big picture.

### 2. Not Addressing Scale
Always consider: "What happens at 10×, 100×, 1000× scale?"

### 3. Ignoring Failure Cases
What if the database is down? What if the message is lost?

### 4. Single Solution Mindset
Always show you considered alternatives.

---

Next: [Trade-offs Discussion](../06-trade-offs/README.md)
