# Design a Stock Exchange

Design a stock exchange matching engine and trading system.

## 1. Requirements

### Functional Requirements
- Place orders (buy/sell, limit/market)
- Cancel/modify orders
- Match orders and execute trades
- Real-time price updates
- Order book management
- Position and balance tracking

### Non-Functional Requirements
- Ultra-low latency (<1ms for matching)
- High throughput (100K+ orders/sec)
- Strong consistency (no partial fills)
- Fault tolerance (no order loss)
- Fairness (FIFO order matching)

### Capacity Estimation

```
Orders: 100K/sec peak
Trades: 50K/sec peak (assuming 50% match rate)
Symbols: 10,000 tradable instruments
Order book depth: 100 price levels per side

Message rate: 100K orders + 50K trades + price updates
           = 200K+ messages/sec
```

## 2. High-Level Design

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌────────────────────────────────────────────────────────┐   │
│  │                     Gateway Layer                       │   │
│  │  (Connection management, authentication, rate limiting) │   │
│  └───────────────────────┬────────────────────────────────┘   │
│                          │                                     │
│                          ▼                                     │
│  ┌────────────────────────────────────────────────────────┐   │
│  │                    Order Router                         │   │
│  │  (Routes to correct matching engine by symbol)         │   │
│  └───────────────────────┬────────────────────────────────┘   │
│                          │                                     │
│           ┌──────────────┼──────────────┐                     │
│           │              │              │                     │
│           ▼              ▼              ▼                     │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐             │
│  │  Matching   │ │  Matching   │ │  Matching   │             │
│  │  Engine     │ │  Engine     │ │  Engine     │             │
│  │  (AAPL)     │ │  (GOOGL)    │ │  (MSFT)     │             │
│  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘             │
│         │               │               │                     │
│         └───────────────┼───────────────┘                     │
│                         │                                     │
│                         ▼                                     │
│  ┌────────────────────────────────────────────────────────┐   │
│  │              Trade Processing & Settlement              │   │
│  │  (Position updates, clearing, reporting)               │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 3. Order Book

```
┌────────────────────────────────────────────────────────────────┐
│                      Order Book                                │
│                                                                │
│  Symbol: AAPL                                                  │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  BID (Buy Orders)           ASK (Sell Orders)           │  │
│  │                                                          │  │
│  │  Price    Qty    Orders     Price    Qty    Orders      │  │
│  │  ─────────────────────     ─────────────────────        │  │
│  │  150.10   500    [O1,O2]   150.15   300    [O5]        │  │
│  │  150.05   1000   [O3]      150.20   800    [O6,O7]     │  │
│  │  150.00   2000   [O4]      150.25   1500   [O8,O9,O10] │  │
│  │                                                          │  │
│  │  Best Bid: 150.10          Best Ask: 150.15            │  │
│  │  Spread: 0.05                                           │  │
│  │                                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  Data structures:                                              │
│  - Price levels: Sorted map (Red-Black Tree)                 │
│  - Orders at price: FIFO queue (linked list)                 │
│  - Order lookup: Hash map (order_id → order)                 │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Order Book Implementation

```python
from sortedcontainers import SortedDict
from collections import deque

class OrderBook:
    def __init__(self, symbol):
        self.symbol = symbol
        self.bids = SortedDict()  # price → deque of orders (descending)
        self.asks = SortedDict()  # price → deque of orders (ascending)
        self.orders = {}  # order_id → order

    def add_order(self, order):
        """Add order to book"""
        self.orders[order.id] = order

        book = self.bids if order.side == 'BUY' else self.asks
        price = order.price

        if price not in book:
            book[price] = deque()

        book[price].append(order)

    def remove_order(self, order_id):
        """Remove order from book"""
        if order_id not in self.orders:
            return None

        order = self.orders.pop(order_id)
        book = self.bids if order.side == 'BUY' else self.asks

        if order.price in book:
            book[order.price].remove(order)
            if not book[order.price]:
                del book[order.price]

        return order

    def get_best_bid(self):
        """Get highest bid price"""
        if self.bids:
            return self.bids.keys()[-1]  # Last key (highest)
        return None

    def get_best_ask(self):
        """Get lowest ask price"""
        if self.asks:
            return self.asks.keys()[0]  # First key (lowest)
        return None

    def get_orders_at_price(self, side, price):
        """Get all orders at a price level"""
        book = self.bids if side == 'BUY' else self.asks
        return list(book.get(price, []))
```

## 4. Matching Engine

```
┌────────────────────────────────────────────────────────────────┐
│                    Matching Algorithm                          │
│                                                                │
│  Price-Time Priority (FIFO):                                  │
│  1. Match at best price first                                 │
│  2. At same price, first order has priority                  │
│                                                                │
│  Example: Incoming BUY order @ 150.20 for 500 shares         │
│                                                                │
│  ASK book:                                                     │
│  150.15: [Order A (100), Order B (200)]                      │
│  150.20: [Order C (150), Order D (300)]                      │
│                                                                │
│  Matching sequence:                                            │
│  1. Match with A: 100 shares @ 150.15 (A fully filled)       │
│  2. Match with B: 200 shares @ 150.15 (B fully filled)       │
│  3. Match with C: 150 shares @ 150.20 (C fully filled)       │
│  4. Match with D: 50 shares @ 150.20 (D partial, 250 left)  │
│                                                                │
│  Incoming order fully filled (500 shares)                     │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Matching Engine Implementation

```python
class MatchingEngine:
    def __init__(self, symbol):
        self.symbol = symbol
        self.order_book = OrderBook(symbol)
        self.trade_id_counter = 0

    def process_order(self, order):
        """Process incoming order"""
        trades = []

        if order.type == 'MARKET':
            trades = self.match_market_order(order)
        elif order.type == 'LIMIT':
            trades = self.match_limit_order(order)

        # If order has remaining quantity, add to book
        if order.remaining_qty > 0 and order.type == 'LIMIT':
            self.order_book.add_order(order)

        return trades

    def match_limit_order(self, order):
        """Match limit order against order book"""
        trades = []

        if order.side == 'BUY':
            # Match against asks
            while order.remaining_qty > 0:
                best_ask = self.order_book.get_best_ask()

                if best_ask is None or best_ask > order.price:
                    break  # No matching price

                trades.extend(self.match_at_price(order, 'SELL', best_ask))
        else:
            # Match against bids
            while order.remaining_qty > 0:
                best_bid = self.order_book.get_best_bid()

                if best_bid is None or best_bid < order.price:
                    break  # No matching price

                trades.extend(self.match_at_price(order, 'BUY', best_bid))

        return trades

    def match_at_price(self, incoming, contra_side, price):
        """Match incoming order against orders at a price level"""
        trades = []
        book = self.order_book.bids if contra_side == 'BUY' else self.order_book.asks

        while incoming.remaining_qty > 0 and price in book:
            resting_orders = book[price]

            if not resting_orders:
                del book[price]
                break

            resting = resting_orders[0]
            match_qty = min(incoming.remaining_qty, resting.remaining_qty)

            # Create trade
            trade = Trade(
                id=self.generate_trade_id(),
                symbol=self.symbol,
                price=price,
                quantity=match_qty,
                buy_order_id=incoming.id if incoming.side == 'BUY' else resting.id,
                sell_order_id=resting.id if incoming.side == 'BUY' else incoming.id,
                timestamp=datetime.utcnow()
            )
            trades.append(trade)

            # Update quantities
            incoming.remaining_qty -= match_qty
            resting.remaining_qty -= match_qty

            # Remove fully filled resting order
            if resting.remaining_qty == 0:
                resting_orders.popleft()
                del self.order_book.orders[resting.id]

        return trades

    def cancel_order(self, order_id):
        """Cancel an order"""
        return self.order_book.remove_order(order_id)
```

## 5. Order Types

```
┌────────────────────────────────────────────────────────────────┐
│                    Order Types                                 │
│                                                                │
│  1. Market Order                                               │
│     - Execute immediately at best available price             │
│     - No price guarantee                                       │
│                                                                │
│  2. Limit Order                                                │
│     - Execute at specified price or better                    │
│     - May not fill if price not reached                       │
│                                                                │
│  3. Stop Order                                                 │
│     - Becomes market order when trigger price reached         │
│     - Used for stop-loss                                       │
│                                                                │
│  4. Stop-Limit Order                                           │
│     - Becomes limit order when trigger price reached          │
│                                                                │
│  Time-in-Force:                                                │
│  - DAY: Valid until market close                              │
│  - GTC: Good till cancelled                                   │
│  - IOC: Immediate or cancel (fill what you can, cancel rest) │
│  - FOK: Fill or kill (all or nothing)                        │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 6. Sequencer (Event Ordering)

```
┌────────────────────────────────────────────────────────────────┐
│                      Sequencer                                 │
│                                                                │
│  Problem: Multiple gateways receive orders simultaneously     │
│  Solution: Single sequencer assigns global sequence numbers   │
│                                                                │
│  ┌────────────────────────────────────────────────────────┐   │
│  │                                                         │   │
│  │  Gateway 1 ─┐                                          │   │
│  │             │      ┌───────────┐      ┌──────────┐    │   │
│  │  Gateway 2 ─┼─────▶│ Sequencer │─────▶│ Matching │    │   │
│  │             │      │ (single)  │      │  Engine  │    │   │
│  │  Gateway 3 ─┘      └───────────┘      └──────────┘    │   │
│  │                                                         │   │
│  │  Sequence: 1, 2, 3, 4, 5, ...                         │   │
│  │                                                         │   │
│  │  All orders processed in sequence order                │   │
│  │  Deterministic replay for recovery                     │   │
│  │                                                         │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                                │
│  High availability:                                            │
│  - Primary/backup sequencer                                   │
│  - Synchronous replication of sequence                       │
│  - Fast failover (<100ms)                                     │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 7. Risk Checks

```python
class RiskManager:
    def __init__(self):
        self.position_limits = {}
        self.order_limits = {}

    def pre_trade_check(self, order, account):
        """Run risk checks before accepting order"""
        checks = [
            self.check_buying_power(order, account),
            self.check_position_limit(order, account),
            self.check_order_size(order),
            self.check_price_collar(order),
            self.check_rate_limit(order, account)
        ]

        for check in checks:
            if not check.passed:
                return RiskResult(passed=False, reason=check.reason)

        return RiskResult(passed=True)

    def check_buying_power(self, order, account):
        """Ensure account has sufficient funds/margin"""
        if order.side == 'BUY':
            required = order.price * order.quantity
            available = account.buying_power

            if required > available:
                return CheckResult(False, 'INSUFFICIENT_BUYING_POWER')

        return CheckResult(True)

    def check_price_collar(self, order):
        """Ensure price is within acceptable range"""
        last_price = self.get_last_price(order.symbol)

        if last_price:
            deviation = abs(order.price - last_price) / last_price

            # Reject if more than 10% from last price
            if deviation > 0.10:
                return CheckResult(False, 'PRICE_OUTSIDE_COLLAR')

        return CheckResult(True)

    def check_rate_limit(self, order, account):
        """Limit orders per second per account"""
        key = f"rate:{account.id}"
        count = self.redis.incr(key)

        if count == 1:
            self.redis.expire(key, 1)

        if count > 100:  # 100 orders/sec limit
            return CheckResult(False, 'RATE_LIMIT_EXCEEDED')

        return CheckResult(True)
```

## 8. Market Data Distribution

```
┌────────────────────────────────────────────────────────────────┐
│              Market Data Distribution                          │
│                                                                │
│  Data types:                                                   │
│  - Trade ticks (each executed trade)                         │
│  - Quote updates (best bid/ask changes)                      │
│  - Order book depth (multiple price levels)                  │
│                                                                │
│  Distribution methods:                                         │
│  1. Multicast (UDP) for low latency                          │
│  2. WebSocket for web clients                                │
│  3. Message queue (Kafka) for downstream systems             │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Matching Engine                                         │  │
│  │       │                                                  │  │
│  │       ▼                                                  │  │
│  │  Market Data Publisher                                   │  │
│  │       │                                                  │  │
│  │       ├──▶ Multicast (UDP) ──▶ Co-located traders      │  │
│  │       │                                                  │  │
│  │       ├──▶ Kafka ──▶ Analytics, Settlement              │  │
│  │       │                                                  │  │
│  │       └──▶ WebSocket Gateway ──▶ Retail clients        │  │
│  │                                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 9. Key Takeaways

1. **Order book** - sorted map + FIFO queues for price-time priority
2. **Single-threaded matching** - simplifies correctness, use per-symbol
3. **Sequencer** - deterministic ordering for fairness and recovery
4. **Pre-trade risk** - validate before accepting orders
5. **Colocation** - low latency for high-frequency traders
6. **Event sourcing** - replay for recovery and audit

---

[Back to Advanced](../README.md) | [Back to Course](../../README.md)
