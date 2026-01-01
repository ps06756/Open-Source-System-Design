# Consensus

Consensus algorithms enable distributed systems to agree on values despite failures. They're fundamental to building reliable distributed systems.

## Table of Contents
- [What is Consensus?](#what-is-consensus)
- [The Problem](#the-problem)
- [Paxos](#paxos)
- [Raft](#raft)
- [Leader Election](#leader-election)
- [Consensus in Practice](#consensus-in-practice)
- [Key Takeaways](#key-takeaways)

## What is Consensus?

Consensus is the process of getting distributed nodes to agree on a single value.

```
┌─────────────────────────────────────────────────────────┐
│                   Consensus                             │
│                                                         │
│  Node 1: "I propose value A"                           │
│  Node 2: "I propose value B"                           │
│  Node 3: "I propose value A"                           │
│                                                         │
│  After consensus:                                       │
│  Node 1: "Agreed: A"                                   │
│  Node 2: "Agreed: A"                                   │
│  Node 3: "Agreed: A"                                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Properties of Consensus

| Property | Description |
|----------|-------------|
| Agreement | All nodes decide on the same value |
| Validity | The decided value was proposed by some node |
| Termination | All non-faulty nodes eventually decide |
| Integrity | Each node decides at most once |

### Use Cases

- **Leader election**: Agree on which node is leader
- **Distributed transactions**: Commit or abort
- **Replicated state machines**: Same operations in same order
- **Distributed locks**: Who holds the lock
- **Configuration management**: Agree on cluster membership

## The Problem

### Why is Consensus Hard?

```
Scenario: 3 nodes, Node 2 crashes

Node 1                    Node 2                    Node 3
   │                         │                         │
   │── "I vote A" ─────────▶ ✗                        │
   │                                                   │
   │                         ✗ ◀─────── "I vote B" ───│
   │                                                   │
   │◀─────────────────────────────────────────────────│
   │          No communication about Node 2           │
   │                                                   │
   │  What happened to Node 2?                        │
   │  - Crashed?                                       │
   │  - Network partition?                             │
   │  - Just slow?                                     │
```

### FLP Impossibility

It's impossible to guarantee consensus in an asynchronous system with even one faulty node.

**Implications:**
- Can't distinguish slow nodes from dead nodes
- Must use timeouts (which can be wrong)
- Algorithms may not terminate in all cases

### Practical Approach

- Accept that termination isn't guaranteed in all cases
- Use timeouts to make progress
- Ensure safety (never decide wrong value)
- Optimize for liveness (usually terminates)

## Paxos

The first proven consensus algorithm (Lamport, 1989).

### Roles

```
┌─────────────────────────────────────────────────────────┐
│                    Paxos Roles                          │
│                                                         │
│  Proposer: Proposes values (can be multiple)           │
│  Acceptor: Votes on proposals (forms quorum)           │
│  Learner: Learns the decided value                     │
│                                                         │
│  (A node can play multiple roles)                      │
└─────────────────────────────────────────────────────────┘
```

### Two Phases

**Phase 1: Prepare**
```
Proposer                               Acceptors
    │                                      │
    │── Prepare(n) ──────────────────────▶│
    │   "I want to propose with           │
    │    proposal number n"               │
    │                                      │
    │◀── Promise(n, prev_accepted) ───────│
    │   "I promise not to accept           │
    │    proposals < n"                    │
    │   "Here's what I previously          │
    │    accepted (if any)"                │
```

**Phase 2: Accept**
```
Proposer                               Acceptors
    │                                      │
    │── Accept(n, value) ────────────────▶│
    │   "Please accept value              │
    │    with proposal number n"          │
    │                                      │
    │◀── Accepted(n) ─────────────────────│
    │   "I accepted it"                    │
    │                                      │
    │                                      │
    │ If majority accepted:               │
    │ Value is decided!                    │
```

### Example Flow

```
5 Acceptors (A1-A5), need majority of 3

Proposer P1: propose "X" with n=1
───────────────────────────────────
1. P1 sends Prepare(1) to all
2. A1, A2, A3 promise (majority)
3. P1 sends Accept(1, "X") to all
4. A1, A2, A3 accept (majority)
5. Value "X" is decided!

Proposer P2 (concurrent): propose "Y" with n=2
─────────────────────────────────────────────
1. P2 sends Prepare(2) to all
2. A4, A5 promise (no prior acceptance)
3. A3 promises but reports it accepted "X" at n=1
4. P2 must use "X" (highest previously accepted)
5. P2 sends Accept(2, "X") - not "Y"!
6. Safety preserved!
```

### Paxos Challenges

- Complex to understand and implement
- Multiple proposers cause dueling (livelock)
- Single decree Paxos decides one value
- Multi-Paxos needed for log replication

## Raft

Designed to be more understandable than Paxos (Ongaro & Ousterhout, 2014).

### Key Insight

Break consensus into subproblems:
1. **Leader election**
2. **Log replication**
3. **Safety**

### States

```
                    ┌─────────────────────┐
                    │      Follower       │
                    │                     │
                    └──────────┬──────────┘
                               │
                   Election timeout
                               │
                               ▼
                    ┌─────────────────────┐
                    │     Candidate       │
                    │                     │
                    └──────────┬──────────┘
                               │
                  Receives majority votes
                               │
                               ▼
                    ┌─────────────────────┐
                    │       Leader        │
                    │                     │
                    └─────────────────────┘
```

### Leader Election

```
┌─────────────────────────────────────────────────────────┐
│                   Leader Election                       │
│                                                         │
│  Term 1:                                               │
│  ┌────────┐  ┌────────┐  ┌────────┐                   │
│  │Follower│  │ Leader │  │Follower│                   │
│  └────────┘  └────────┘  └────────┘                   │
│                                                         │
│  Leader fails:                                          │
│  ┌────────┐  ┌────────┐  ┌────────┐                   │
│  │Follower│  │  (✗)   │  │Follower│                   │
│  └────────┘  └────────┘  └────────┘                   │
│                                                         │
│  Election timeout on Node 1:                           │
│  ┌──────────┐  ┌────────┐  ┌────────┐                 │
│  │Candidate │  │  (✗)   │  │Follower│                 │
│  │ Term 2   │  │        │  │        │                 │
│  └──────────┘  └────────┘  └────────┘                 │
│       │                          │                     │
│       └─── RequestVote ──────────┘                     │
│                                                         │
│  Majority votes:                                        │
│  ┌────────┐  ┌────────┐  ┌────────┐                   │
│  │ Leader │  │  (✗)   │  │Follower│                   │
│  │ Term 2 │  │        │  │        │                   │
│  └────────┘  └────────┘  └────────┘                   │
└─────────────────────────────────────────────────────────┘
```

### Log Replication

```
Leader                                Followers
   │                                      │
   │  Client request: "SET x=1"          │
   │                                      │
   │  1. Append to local log             │
   │     ┌───┬───┬───┐                   │
   │     │ 1 │ 2 │ 3 │ ◀── new entry     │
   │     └───┴───┴───┘                   │
   │                                      │
   │── AppendEntries ────────────────────▶│
   │                                      │
   │◀── Success ─────────────────────────│
   │                                      │
   │  2. Once majority confirms:          │
   │     - Commit entry                   │
   │     - Apply to state machine         │
   │     - Respond to client             │
```

### Safety Rules

```
┌─────────────────────────────────────────────────────────┐
│                    Raft Safety                          │
│                                                         │
│  1. Election Safety                                     │
│     At most one leader per term                        │
│                                                         │
│  2. Leader Append-Only                                  │
│     Leader never overwrites its log                    │
│                                                         │
│  3. Log Matching                                        │
│     If two logs have entry with same index & term,     │
│     all preceding entries are identical                │
│                                                         │
│  4. Leader Completeness                                 │
│     If entry committed in term T, it's in all          │
│     leaders' logs for terms > T                        │
│                                                         │
│  5. State Machine Safety                                │
│     If server applies entry at index i, no other       │
│     server applies different entry at i                │
└─────────────────────────────────────────────────────────┘
```

### Term and Voting

```
Terms act as logical clocks:

Term 1        Term 2        Term 3
├────────────┼────────────┼────────────▶
   Leader A     Leader B      Leader C
   elected      elected       elected

Rules:
- Each node votes at most once per term
- Candidate with higher term wins
- Stale leaders step down when they see higher term
```

## Leader Election

Common pattern across distributed systems.

### Methods

**1. Bully Algorithm**
```
Highest ID node becomes leader
Simple but not partition-tolerant
```

**2. Raft/Paxos**
```
Quorum-based voting
Partition-tolerant
Used by: etcd, Consul, ZooKeeper (ZAB)
```

**3. External Coordinator**
```
Dedicated service (ZooKeeper) manages election
Services register and watch for leader changes
```

### ZooKeeper Leader Election

```
┌─────────────────────────────────────────────────────────┐
│              ZooKeeper Leader Election                  │
│                                                         │
│  1. All nodes create ephemeral sequential nodes:       │
│     /election/node_000000001  (Node A)                 │
│     /election/node_000000002  (Node B)                 │
│     /election/node_000000003  (Node C)                 │
│                                                         │
│  2. Lowest sequence number is leader:                  │
│     Node A (000000001) is leader                       │
│                                                         │
│  3. Others watch the node just before them:            │
│     Node B watches node_000000001                      │
│     Node C watches node_000000002                      │
│                                                         │
│  4. If leader fails:                                    │
│     Node A's ephemeral node disappears                 │
│     Node B notified → becomes leader                   │
└─────────────────────────────────────────────────────────┘
```

### Lease-Based Leadership

```
Leader acquires lease (time-limited lock)
Must renew before expiry
If fails to renew → loses leadership

┌────────────────────────────────────────────────────────┐
│   Time:  0s      5s      10s     15s     20s         │
│          │       │       │       │       │            │
│   Leader:│ A     │ A     │ A     │   B   │ B         │
│          │       │       │ ✗     │       │            │
│          │ lease │renew  │fail   │B wins │            │
└────────────────────────────────────────────────────────┘
```

## Consensus in Practice

### Comparison

| Aspect | Paxos | Raft | ZAB |
|--------|-------|------|-----|
| Understandability | Hard | Easier | Medium |
| Leader | Optional | Required | Required |
| Log ordering | Flexible | Strict | Strict |
| Used by | Chubby, Spanner | etcd, Consul, TiDB | ZooKeeper |

### When to Use Consensus

| Use Case | Example |
|----------|---------|
| Distributed lock | One process holds lock |
| Configuration | Cluster membership |
| Leader election | Database primary |
| Distributed transactions | Commit/abort |
| Replicated log | State machine replication |

### When NOT to Use

- High-throughput writes (consensus is slow)
- Eventual consistency is acceptable
- Single-datacenter deployments (simpler options)

### Performance Considerations

```
Consensus latency = 2 RTT (minimum)
- Prepare/RequestVote
- Accept/AppendEntries

Geographic impact:
Same datacenter: 2 × 1ms = 2ms
Cross-region: 2 × 50ms = 100ms
Cross-continent: 2 × 150ms = 300ms
```

### Optimizations

- **Batching**: Combine multiple operations
- **Pipelining**: Send next request before previous completes
- **Witnesses**: Lightweight nodes for quorum
- **Flexible Paxos**: Different quorums for phases

## Key Takeaways

1. **Consensus enables agreement** despite failures
2. **Paxos is foundational** but complex
3. **Raft is more understandable** - prefer for new implementations
4. **Quorums ensure safety** - majority must agree
5. **Terms/epochs detect stale leaders**
6. **Consensus is expensive** - use only when needed

## Practice Questions

1. Why do we need a majority quorum (not just any quorum)?
2. What happens if two Raft nodes have the same term?
3. How does Raft prevent split-brain?
4. Design a leader election system using ZooKeeper.

## Further Reading

- [Raft Paper](https://raft.github.io/raft.pdf)
- [Raft Visualization](https://raft.github.io/)
- [Paxos Made Simple](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf)
- [ZooKeeper: Distributed Process Coordination](https://zookeeper.apache.org/doc/current/zookeeperOver.html)

---

Next: [Back-of-envelope Calculations](../09-calculations/README.md)
