# Design a Stock Exchange

## 1. Requirements

### Functional Requirements
- Place orders (buy/sell, market/limit)
- Match orders
- Cancel/modify orders
- Real-time price feeds
- Position and balance tracking
- Trade history

### Non-Functional Requirements
- Ultra-low latency (< 1ms for matching)
- High throughput (1M+ orders/second)
- Fairness (FIFO ordering)
- Zero data loss
- Deterministic matching

### Scale Estimates
- 10,000 stocks
- 1M orders per second peak
- 100K active traders
- Nanosecond-level timing matters

---

## 2. High-Level Design

```
┌─────────────────────────────────────────────────────────────┐
│                  High-Level Architecture                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                   Trading Clients                     │  │
│  │            (Brokers, Market Makers)                  │  │
│  └────────────────────────┬─────────────────────────────┘  │
│                           │                                  │
│                           ▼                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                    Gateway                            │  │
│  │         (Authentication, Rate Limiting)              │  │
│  └────────────────────────┬─────────────────────────────┘  │
│                           │                                  │
│                           ▼                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                 Order Router                          │  │
│  │      (Route to correct matching engine)              │  │
│  └────────────────────────┬─────────────────────────────┘  │
│                           │                                  │
│         ┌─────────────────┼─────────────────┐               │
│         ▼                 ▼                 ▼               │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐       │
│  │  Matching   │   │  Matching   │   │  Matching   │       │
│  │  Engine     │   │  Engine     │   │  Engine     │       │
│  │  (AAPL)     │   │  (GOOG)     │   │  (MSFT)     │       │
│  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘       │
│         │                 │                 │               │
│         └─────────────────┼─────────────────┘               │
│                           ▼                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Market Data Publisher                    │  │
│  │        (Prices, Trades, Order Book)                  │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Order Book

```
┌─────────────────────────────────────────────────────────────┐
│                       Order Book                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  For each stock, maintain buy and sell orders:             │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  AAPL Order Book                                       │ │
│  │                                                         │ │
│  │  BIDS (Buy Orders)         ASKS (Sell Orders)          │ │
│  │  ┌───────┬───────────┐    ┌───────┬───────────┐       │ │
│  │  │ Price │ Quantity  │    │ Price │ Quantity  │       │ │
│  │  ├───────┼───────────┤    ├───────┼───────────┤       │ │
│  │  │$150.05│   500     │    │$150.10│   200     │       │ │
│  │  │$150.00│  1000     │    │$150.15│   800     │       │ │
│  │  │$149.95│   300     │    │$150.20│   400     │       │ │
│  │  │$149.90│   750     │    │$150.25│   600     │       │ │
│  │  └───────┴───────────┘    └───────┴───────────┘       │ │
│  │                                                         │ │
│  │  Best Bid: $150.05        Best Ask: $150.10            │ │
│  │  Spread: $0.05                                         │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Data Structures:                                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Price Level = TreeMap<Price, Queue<Order>>            │ │
│  │  • Sorted by price (best price first)                 │ │
│  │  • FIFO queue at each price level                     │ │
│  │                                                         │ │
│  │  Order = {                                              │ │
│  │    order_id, user_id, side (buy/sell),                │ │
│  │    price, quantity, timestamp                         │ │
│  │  }                                                      │ │
│  │                                                         │ │
│  │  Operations:                                            │ │
│  │  • Add order: O(log P) where P = price levels         │ │
│  │  • Match: O(1) for top of book                        │ │
│  │  • Cancel: O(1) with order_id → order map            │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Matching Algorithm

```
┌─────────────────────────────────────────────────────────────┐
│                   Matching Algorithm                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Price-Time Priority (FIFO):                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Price priority: Best price matches first          │ │
│  │     Buy at $150 matches before Buy at $149            │ │
│  │                                                         │ │
│  │  2. Time priority: At same price, first in wins       │ │
│  │     Earlier order at $150 matches before later order  │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Matching Flow:                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  New Order: BUY 1000 shares @ $150.10 (limit)         │ │
│  │                                                         │ │
│  │  1. Check if crosses with asks                        │ │
│  │     Best Ask: $150.10 @ 200 shares                    │ │
│  │     $150.10 >= $150.10 ✓ Match!                       │ │
│  │                                                         │ │
│  │  2. Execute match                                      │ │
│  │     Trade: 200 shares @ $150.10                       │ │
│  │     Remaining: 800 shares to fill                     │ │
│  │                                                         │ │
│  │  3. Continue to next ask level                        │ │
│  │     Next Ask: $150.15 @ 800 shares                    │ │
│  │     $150.10 < $150.15 ✗ No match                      │ │
│  │                                                         │ │
│  │  4. Add remainder to order book                       │ │
│  │     800 shares @ $150.10 added to BIDS               │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Order Types:                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Market Order: Execute at best available price        │ │
│  │  Limit Order: Execute only at specified price or better│ │
│  │  Stop Order: Trigger when price hits level            │ │
│  │  IOC (Immediate or Cancel): Fill what you can, cancel rest│ │
│  │  FOK (Fill or Kill): All or nothing                   │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Low Latency Design

```
┌─────────────────────────────────────────────────────────────┐
│                  Low Latency Design                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Target: < 1 microsecond matching latency                  │
│                                                              │
│  Techniques:                                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Single-threaded matching engine                    │ │
│  │     • No locks, no contention                         │ │
│  │     • Deterministic execution                         │ │
│  │     • One thread per symbol                           │ │
│  │                                                         │ │
│  │  2. Memory optimization                                │ │
│  │     • All data in memory (no disk I/O on hot path)   │ │
│  │     • Object pooling (no GC pressure)                 │ │
│  │     • Cache-friendly data structures                  │ │
│  │                                                         │ │
│  │  3. Network optimization                               │ │
│  │     • Kernel bypass (DPDK, RDMA)                      │ │
│  │     • Binary protocols (not JSON)                     │ │
│  │     • Co-located servers                              │ │
│  │                                                         │ │
│  │  4. Language choice                                    │ │
│  │     • C++ or Rust for matching engine                 │ │
│  │     • Java with GC tuning also common                 │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Architecture for scale:                                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Partition by symbol:                                  │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │ │
│  │  │ Engine 1     │  │ Engine 2     │  │ Engine 3     │ │ │
│  │  │ AAPL, MSFT   │  │ GOOG, AMZN   │  │ FB, NFLX     │ │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘ │ │
│  │                                                         │ │
│  │  Each engine: Single-threaded, handles subset         │ │
│  │  No coordination needed between engines               │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Fault Tolerance

```
┌─────────────────────────────────────────────────────────────┐
│                   Fault Tolerance                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Requirements: Zero data loss, fast recovery               │
│                                                              │
│  Write-Ahead Log:                                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Every order → Log → Then process                     │ │
│  │                                                         │ │
│  │  Sequence:                                              │ │
│  │  1. Receive order                                      │ │
│  │  2. Append to WAL (durable, replicated)               │ │
│  │  3. Process in matching engine                        │ │
│  │  4. Publish results                                    │ │
│  │                                                         │ │
│  │  Recovery:                                              │ │
│  │  1. Load last snapshot                                 │ │
│  │  2. Replay WAL from snapshot                          │ │
│  │  3. Matching engine = same state                      │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Active-Passive Replication:                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ┌──────────────┐      ┌──────────────┐              │ │
│  │  │   Primary    │─────▶│   Replica    │              │ │
│  │  │ (processes)  │ WAL  │ (standby)    │              │ │
│  │  └──────────────┘      └──────────────┘              │ │
│  │                                                         │ │
│  │  On primary failure:                                   │ │
│  │  1. Detect failure (heartbeat)                        │ │
│  │  2. Promote replica to primary                        │ │
│  │  3. Failover time: < 1 second                        │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Sequencer (optional, for strict ordering):               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  All orders go through sequencer first:               │ │
│  │  → Assigns global sequence number                     │ │
│  │  → Guarantees same order on primary and replica       │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Market Data

```
┌─────────────────────────────────────────────────────────────┐
│                      Market Data                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Types of Market Data:                                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Trades (tape)                                      │ │
│  │     { symbol, price, quantity, timestamp }            │ │
│  │                                                         │ │
│  │  2. Quotes (top of book)                              │ │
│  │     { symbol, bid_price, bid_qty, ask_price, ask_qty }│ │
│  │                                                         │ │
│  │  3. Order book depth                                   │ │
│  │     Full order book or N levels                       │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Distribution:                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Matching Engine                                       │ │
│  │        │                                               │ │
│  │        ▼                                               │ │
│  │  ┌─────────────────┐                                   │ │
│  │  │ Market Data     │                                   │ │
│  │  │ Publisher       │                                   │ │
│  │  └────────┬────────┘                                   │ │
│  │           │ Multicast                                  │ │
│  │    ┌──────┼──────┬──────┬──────┐                      │ │
│  │    ▼      ▼      ▼      ▼      ▼                      │ │
│  │  Sub 1  Sub 2  Sub 3  Sub 4  Sub N                    │ │
│  │                                                         │ │
│  │  UDP Multicast: Most efficient for many subscribers   │ │
│  │  Reliable: Gap detection + recovery protocol          │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. Summary

| Component | Choice | Reason |
|-----------|--------|--------|
| **Matching Engine** | Single-threaded per symbol | No contention |
| **Order Book** | TreeMap + FIFO queues | Price-time priority |
| **Storage** | In-memory + WAL | Speed + durability |
| **Network** | Kernel bypass, binary | Ultra-low latency |
| **Market Data** | UDP multicast | Scalable distribution |

---

[Back to System Designs](../README.md)
