# Design an Ad Serving Platform

## 1. Requirements

### Functional Requirements
- Serve ads based on targeting criteria
- Real-time bidding (RTB)
- Campaign management
- Billing and budget tracking
- Analytics and reporting
- A/B testing for creatives

### Non-Functional Requirements
- Ultra-low latency (< 100ms for ad selection)
- Handle 1M+ ad requests per second
- High availability
- Real-time budget enforcement
- Fraud detection

### Scale Estimates
- 10B ad impressions per day
- 1M advertisers
- 100M targeting segments
- 100K+ QPS peak

---

## 2. High-Level Design

```
┌─────────────────────────────────────────────────────────────┐
│                  High-Level Architecture                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                   Publisher Site                      │  │
│  │             (Website/App with ad slots)              │  │
│  └────────────────────────┬─────────────────────────────┘  │
│                           │ Ad Request                      │
│                           ▼                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                    Ad Exchange                        │  │
│  │           (Orchestrates real-time bidding)           │  │
│  └────────────────────────┬─────────────────────────────┘  │
│                           │                                  │
│         ┌─────────────────┼─────────────────┐               │
│         ▼                 ▼                 ▼               │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐       │
│  │   DSP 1     │   │   DSP 2     │   │   DSP N     │       │
│  │ (Demand-Side│   │             │   │             │       │
│  │  Platform)  │   │             │   │             │       │
│  └─────────────┘   └─────────────┘   └─────────────┘       │
│         │                                                   │
│         ▼                                                   │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Advertiser Campaigns                     │  │
│  │         (Targeting, Budget, Creatives)               │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Real-Time Bidding Flow

```
┌─────────────────────────────────────────────────────────────┐
│                 Real-Time Bidding Flow                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Timeline: < 100ms total                                    │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. User visits publisher site (T=0ms)                 │ │
│  │     ┌────────────┐                                     │ │
│  │     │ Publisher  │ → Ad slot needs to be filled       │ │
│  │     └────────────┘                                     │ │
│  │                                                         │ │
│  │  2. Bid request to exchange (T=5ms)                    │ │
│  │     ┌────────────┐   ┌───────────────────────┐        │ │
│  │     │ Publisher  │──▶│ Ad Exchange           │        │ │
│  │     └────────────┘   │ {user_id, context,    │        │ │
│  │                      │  ad_slot_size, ...}   │        │ │
│  │                      └───────────────────────┘        │ │
│  │                                                         │ │
│  │  3. Exchange fans out to DSPs (T=10ms)                │ │
│  │     ┌─────────────────────────────────────────────┐   │ │
│  │     │           Bid Request (parallel)             │   │ │
│  │     │   ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐       │   │ │
│  │     └──▶│DSP 1│  │DSP 2│  │DSP 3│  │DSP N│       │   │ │
│  │         └─────┘  └─────┘  └─────┘  └─────┘       │   │ │
│  │                                                         │ │
│  │  4. DSPs evaluate and bid (T=10-50ms)                 │ │
│  │     - Match user to campaigns                          │ │
│  │     - Check budget                                     │ │
│  │     - Calculate bid price                              │ │
│  │     - Return bid response                              │ │
│  │                                                         │ │
│  │  5. Exchange runs auction (T=55ms)                    │ │
│  │     Second-price auction: Winner pays $0.01 more      │ │
│  │     than second-highest bid                           │ │
│  │                                                         │ │
│  │  6. Return winning ad to publisher (T=60ms)           │ │
│  │                                                         │ │
│  │  7. Render ad to user (T=100ms)                       │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Ad Selection (DSP Internals)

```
┌─────────────────────────────────────────────────────────────┐
│                     Ad Selection                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Given a bid request, which ad to show?                    │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Candidate Retrieval (fast filtering)              │ │
│  │     ┌───────────────────────────────────────────────┐ │ │
│  │     │ Filter by:                                     │ │ │
│  │     │ • Geography (user in US?)                     │ │ │
│  │     │ • Ad format (banner? video?)                  │ │ │
│  │     │ • Device (mobile? desktop?)                   │ │ │
│  │     │ • Budget remaining > 0                        │ │ │
│  │     │                                                │ │ │
│  │     │ 1M campaigns → 1000 candidates               │ │ │
│  │     └───────────────────────────────────────────────┘ │ │
│  │                                                         │ │
│  │  2. Targeting Match                                    │ │
│  │     ┌───────────────────────────────────────────────┐ │ │
│  │     │ User segments: [tech_enthusiast, age_25_34]  │ │ │
│  │     │ Campaign targets: [tech_enthusiast, income_high]│ │
│  │     │ Match score: 0.7                              │ │ │
│  │     └───────────────────────────────────────────────┘ │ │
│  │                                                         │ │
│  │  3. Predicted Performance (ML model)                  │ │
│  │     ┌───────────────────────────────────────────────┐ │ │
│  │     │ pCTR = 0.02 (2% predicted click-through rate) │ │ │
│  │     │ pCVR = 0.05 (5% predicted conversion rate)    │ │ │
│  │     └───────────────────────────────────────────────┘ │ │
│  │                                                         │ │
│  │  4. Calculate Bid (eCPM optimization)                 │ │
│  │     ┌───────────────────────────────────────────────┐ │ │
│  │     │ eCPM = pCTR × pCVR × Conversion_Value × 1000 │ │ │
│  │     │                                                │ │ │
│  │     │ Example:                                       │ │ │
│  │     │ eCPM = 0.02 × 0.05 × $50 × 1000 = $50        │ │ │
│  │     │ Bid = $50 × margin = $40                     │ │ │
│  │     └───────────────────────────────────────────────┘ │ │
│  │                                                         │ │
│  │  5. Budget Check (real-time)                          │ │
│  │     ┌───────────────────────────────────────────────┐ │ │
│  │     │ Campaign budget: $1000/day                    │ │ │
│  │     │ Spent so far: $800                            │ │ │
│  │     │ Remaining: $200 ✓ Can bid                    │ │ │
│  │     └───────────────────────────────────────────────┘ │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Pacing and Budget Control

```
┌─────────────────────────────────────────────────────────────┐
│                Pacing and Budget Control                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Problem: Don't spend entire daily budget in first hour    │
│                                                              │
│  Pacing Algorithms:                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Probabilistic Throttling                           │ │
│  │     ┌───────────────────────────────────────────────┐ │ │
│  │     │ bid_probability = remaining_budget / hours_left│ │ │
│  │     │                                                │ │ │
│  │     │ If ahead of pace: lower probability           │ │ │
│  │     │ If behind pace: higher probability            │ │ │
│  │     │                                                │ │ │
│  │     │ Random number < probability → bid             │ │ │
│  │     └───────────────────────────────────────────────┘ │ │
│  │                                                         │ │
│  │  2. Bid Shading                                        │ │
│  │     ┌───────────────────────────────────────────────┐ │ │
│  │     │ Reduce bid price to control spend rate        │ │ │
│  │     │                                                │ │ │
│  │     │ shaded_bid = base_bid × pacing_factor         │ │ │
│  │     │ pacing_factor = 0.5 to 1.0 based on pace      │ │ │
│  │     └───────────────────────────────────────────────┘ │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Real-Time Budget Tracking:                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Challenge: Distributed servers, avoid overspend      │ │
│  │                                                         │ │
│  │  Solution: Two-phase approach                         │ │
│  │                                                         │ │
│  │  1. Pre-decrement (estimate on bid)                   │ │
│  │     Redis DECRBY budget:{campaign_id} estimated_cost │ │
│  │                                                         │ │
│  │  2. Reconcile (actual on impression)                  │ │
│  │     Adjust for actual win price                       │ │
│  │                                                         │ │
│  │  3. Periodic sync to database                         │ │
│  │     Every 5 minutes, persist to durable storage       │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Fraud Detection

```
┌─────────────────────────────────────────────────────────────┐
│                    Fraud Detection                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Types of Ad Fraud:                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Bot traffic: Non-human impressions/clicks         │ │
│  │  2. Click fraud: Invalid or incentivized clicks       │ │
│  │  3. Ad stacking: Multiple ads stacked invisibly       │ │
│  │  4. Domain spoofing: Fake premium inventory           │ │
│  │  5. Pixel stuffing: 1x1 pixel ads                     │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Detection Methods:                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Real-time (pre-bid):                                  │ │
│  │  • IP reputation scoring                               │ │
│  │  • User agent analysis                                 │ │
│  │  • Device fingerprinting                               │ │
│  │  • ads.txt/app-ads.txt verification                   │ │
│  │                                                         │ │
│  │  Post-impression (analytics):                         │ │
│  │  • Click-to-conversion rate analysis                  │ │
│  │  • Session behavior patterns                          │ │
│  │  • Geographic anomalies                               │ │
│  │  • Time-based patterns                                │ │
│  │                                                         │ │
│  │  ML Models:                                            │ │
│  │  • Train on labeled fraud data                        │ │
│  │  • Features: click timing, mouse movement, etc.       │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Summary

| Component | Choice | Reason |
|-----------|--------|--------|
| **Ad Selection** | Multi-stage filtering + ML | Speed + accuracy |
| **Budget Tracking** | Redis + async reconciliation | Real-time |
| **Auction** | Second-price | Incentive compatible |
| **Targeting Index** | Inverted index | Fast segment matching |
| **Logging** | Kafka → Data warehouse | Analytics |

---

[Back to System Designs](../README.md)
