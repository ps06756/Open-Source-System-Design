# Replication

Replication is copying data across multiple machines for fault tolerance, scalability, and lower latency. Understanding replication strategies is crucial for designing reliable systems.

## Table of Contents
- [Why Replicate?](#why-replicate)
- [Leader-Follower Replication](#leader-follower-replication)
- [Multi-Leader Replication](#multi-leader-replication)
- [Leaderless Replication](#leaderless-replication)
- [Synchronous vs Asynchronous](#synchronous-vs-asynchronous)
- [Replication Lag](#replication-lag)
- [Key Takeaways](#key-takeaways)

## Why Replicate?

### Goals of Replication

```
┌─────────────────────────────────────────────────────────┐
│                  Replication Goals                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. High Availability                                   │
│     └── System continues if nodes fail                  │
│                                                         │
│  2. Fault Tolerance                                     │
│     └── Data survives hardware failures                 │
│                                                         │
│  3. Read Scalability                                    │
│     └── Distribute read load across replicas            │
│                                                         │
│  4. Geographic Distribution                             │
│     └── Data close to users = lower latency             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## Leader-Follower Replication

Also called primary-replica, master-slave, or active-passive.

### How It Works

```
                    ┌─────────────────┐
       Writes ────▶│     Leader      │
                    │   (Primary)     │
                    └────────┬────────┘
                             │
              Replication    │
                             │
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
   ┌──────────┐        ┌──────────┐        ┌──────────┐
   │ Follower │        │ Follower │        │ Follower │
   │    1     │        │    2     │        │    3     │
   └──────────┘        └──────────┘        └──────────┘
         ▲                   ▲                   ▲
         │                   │                   │
         └───────────────────┴───────────────────┘
                             │
                          Reads
```

### Advantages

| Advantage | Description |
|-----------|-------------|
| Simple | One writer, clear data flow |
| No write conflicts | Only leader accepts writes |
| Read scalability | Add followers for more read capacity |

### Disadvantages

| Disadvantage | Description |
|--------------|-------------|
| Write bottleneck | Single leader limits writes |
| Failover complexity | Need leader election |
| Replication lag | Followers may be behind |

### Failover

```
Normal operation:
┌────────┐                    ┌──────────┐
│ Leader │ ◀── replication ── │ Follower │
│  (OK)  │                    │          │
└────────┘                    └──────────┘

Leader fails:
┌────────┐                    ┌──────────┐
│ Leader │                    │ Follower │
│  (✗)   │                    │          │
└────────┘                    └──────────┘

Failover:
┌────────┐                    ┌──────────┐
│(dead)  │                    │  Leader  │ ◀── promoted
│        │                    │  (new)   │
└────────┘                    └──────────┘
```

**Failover challenges:**
- Detecting leader failure (timeout-based)
- Choosing new leader (consensus)
- Redirecting clients
- Handling data loss if async replication

### Used By

- PostgreSQL, MySQL, Oracle
- MongoDB
- Redis (with Sentinel)
- Kafka

## Multi-Leader Replication

Multiple nodes can accept writes.

### How It Works

```
┌─────────────────────────────────────────────────────────┐
│                    Multi-Leader                         │
│                                                         │
│  ┌──────────┐         ┌──────────┐         ┌──────────┐│
│  │ Leader 1 │◀───────▶│ Leader 2 │◀───────▶│ Leader 3 ││
│  │  (US)    │         │  (EU)    │         │  (Asia)  ││
│  └────┬─────┘         └────┬─────┘         └────┬─────┘│
│       │                    │                    │      │
│       ▼                    ▼                    ▼      │
│  ┌──────────┐         ┌──────────┐         ┌──────────┐│
│  │ Followers│         │ Followers│         │ Followers││
│  └──────────┘         └──────────┘         └──────────┘│
└─────────────────────────────────────────────────────────┘
```

### Use Cases

- **Multi-datacenter operation**: Each DC has a leader
- **Offline clients**: Laptop/mobile acts as local leader
- **Collaborative editing**: Google Docs

### Conflict Resolution

When same data is modified on different leaders:

```
Leader 1: UPDATE user SET name = 'Alice' WHERE id = 1
Leader 2: UPDATE user SET name = 'Bob' WHERE id = 1
                    ▼
                Conflict!
```

**Resolution strategies:**

| Strategy | Description | Example |
|----------|-------------|---------|
| Last Write Wins (LWW) | Highest timestamp wins | Cassandra |
| Merge values | Combine conflicting values | CRDTs |
| Keep all versions | Let application resolve | CouchDB |
| Custom logic | Application-specific rules | Shopping cart |

### Topologies

```
Circular:                    Star:                     All-to-all:

   ┌──▶ L1 ──┐                  L1                     L1 ◀──▶ L2
   │         ▼              ▲   │   ▲                   ▲       ▲
  L3        L2             L2◀──┴──▶L3                  │       │
   ▲         │                                          ▼       ▼
   └─────────┘                                         L3 ◀──▶ L4
```

## Leaderless Replication

No single leader. Any node can accept writes.

### How It Works

```
┌─────────────────────────────────────────────────────────┐
│                    Leaderless                           │
│                                                         │
│  Client writes to multiple nodes simultaneously:        │
│                                                         │
│          ┌─────────────────────────────┐               │
│          │         Client              │               │
│          └──────────┬──────────────────┘               │
│                     │                                   │
│         ┌───────────┼───────────┐                      │
│         ▼           ▼           ▼                      │
│    ┌────────┐  ┌────────┐  ┌────────┐                 │
│    │ Node 1 │  │ Node 2 │  │ Node 3 │                 │
│    │  (✓)   │  │  (✓)   │  │  (✗)   │                 │
│    └────────┘  └────────┘  └────────┘                 │
│                                                         │
│    Write succeeds if W nodes respond (W=2 here)        │
└─────────────────────────────────────────────────────────┘
```

### Quorum Reads and Writes

```
N = Total replicas
W = Write quorum (nodes that must confirm write)
R = Read quorum (nodes to read from)

If W + R > N: Guaranteed to read latest write

Example with N=3:
┌───────────────────────────────────────────────────────┐
│  W=2, R=2                                             │
│                                                       │
│  Write to 2 nodes:   [v2] [v2] [v1]                  │
│  Read from 2 nodes:  [v2] [v1]                       │
│  Return: v2 (latest)                                  │
│                                                       │
│  At least 1 node has the latest value                │
└───────────────────────────────────────────────────────┘
```

### Read Repair

Fix stale replicas during reads:

```
Client reads from 3 nodes:
┌────────┐  ┌────────┐  ┌────────┐
│ Node 1 │  │ Node 2 │  │ Node 3 │
│  v=5   │  │  v=5   │  │  v=3   │ ◀── stale!
└────────┘  └────────┘  └────────┘

Client detects v=3 is stale
Client writes v=5 to Node 3 (repair)
```

### Anti-Entropy

Background process to sync replicas:

```
┌────────────────────────────────────────────────────────┐
│                 Anti-Entropy Process                   │
│                                                        │
│  Periodically:                                         │
│  1. Compare Merkle trees between nodes                 │
│  2. Identify different branches                        │
│  3. Exchange only differing data                       │
│                                                        │
│  Node 1 Tree    Node 2 Tree                           │
│      A              A                                  │
│     / \            / \                                 │
│    B   C          B   C*  ◀── Different!              │
│                                                        │
│  Only sync subtree C                                   │
└────────────────────────────────────────────────────────┘
```

### Used By

- Cassandra
- DynamoDB
- Riak
- Voldemort

## Synchronous vs Asynchronous

### Synchronous Replication

Leader waits for follower confirmation before acknowledging write.

```
Client        Leader        Follower
   │            │              │
   │── Write ──▶│              │
   │            │── Replicate ▶│
   │            │              │
   │            │◀── ACK ──────│
   │◀── ACK ────│              │
   │            │              │
```

**Pros:** Guaranteed durability
**Cons:** Higher latency, availability depends on follower

### Asynchronous Replication

Leader acknowledges immediately, replicates in background.

```
Client        Leader        Follower
   │            │              │
   │── Write ──▶│              │
   │◀── ACK ────│              │
   │            │              │
   │            │── Replicate ▶│
   │            │  (background)│
```

**Pros:** Lower latency, higher availability
**Cons:** Potential data loss if leader fails

### Semi-Synchronous

Wait for at least one follower (not all).

```
┌────────────────────────────────────────────────────────┐
│               Semi-Synchronous                         │
│                                                        │
│  Leader waits for 1 sync follower + N async           │
│                                                        │
│  ┌────────┐                                           │
│  │ Leader │──┬── Sync ──▶ Follower 1 (wait)          │
│  │        │  │                                        │
│  │        │  └── Async ──▶ Follower 2 (don't wait)   │
│  │        │  └── Async ──▶ Follower 3 (don't wait)   │
│  └────────┘                                           │
│                                                        │
│  ACK when Leader + Follower 1 have data              │
└────────────────────────────────────────────────────────┘
```

## Replication Lag

The delay between a write on leader and its visibility on followers.

### Problems from Lag

**1. Read Your Own Writes**
```
Timeline:
User writes ──▶ Leader ──(lag)──▶ Follower
                 │                    │
                 │    User reads ─────┘
                 │    (stale data!)
```

**Solution:** Read from leader for recently-written data

**2. Monotonic Reads**
```
User reads from Follower 1: sees v5
User reads from Follower 2: sees v3 (further behind!)
                             ▲
                             └── Going backwards in time!
```

**Solution:** Stick user to same replica

**3. Consistent Prefix Reads**
```
Original order: A → B → C

Follower 1 receives: A, C (B delayed)
Follower 2 receives: B, C

Reader might see: C, B, A (wrong order!)
```

**Solution:** Use same partition for causally related writes

### Measuring Lag

```sql
-- MySQL
SHOW SLAVE STATUS\G
-- Look for: Seconds_Behind_Master

-- PostgreSQL
SELECT
    pg_last_wal_receive_lsn() - pg_last_wal_replay_lsn()
    AS replication_lag;
```

## Summary Table

| Aspect | Leader-Follower | Multi-Leader | Leaderless |
|--------|-----------------|--------------|------------|
| Write location | Leader only | Any leader | Any node |
| Write conflicts | None | Yes | Yes |
| Read scalability | Good | Good | Good |
| Write scalability | Limited | Good | Good |
| Latency (writes) | Higher | Lower | Lowest |
| Consistency | Easier | Harder | Harder |
| Failover | Required | Simpler | Not needed |
| Use case | General | Multi-DC | High write |

## Key Takeaways

1. **Leader-follower** is simplest - use for most cases
2. **Multi-leader** for multi-datacenter and offline support
3. **Leaderless** for high write availability
4. **Sync replication** for durability, **async** for speed
5. **Replication lag** causes consistency issues - design for it
6. **Quorums** provide tunable consistency in leaderless systems

## Practice Questions

1. How would you design failover for a leader-follower system?
2. When would you choose multi-leader over leader-follower?
3. Calculate W and R for N=5 to guarantee consistency.
4. How does read repair help maintain consistency?

## Further Reading

- [Designing Data-Intensive Applications](https://dataintensive.net/) - Chapter 5
- [Amazon Dynamo Paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
- [Raft Consensus Algorithm](https://raft.github.io/)

---

Next: [Consensus](../08-consensus/README.md)
