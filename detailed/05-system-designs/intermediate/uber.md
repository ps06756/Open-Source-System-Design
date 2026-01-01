# Design Uber

## 1. Requirements

### Functional Requirements
- Riders can request rides
- Drivers can accept/reject rides
- Real-time location tracking
- Matching riders with nearby drivers
- ETA calculation
- Fare calculation
- Payment processing
- Rating system

### Non-Functional Requirements
- Low latency for matching (< 1 second)
- Real-time location updates
- High availability
- Handle surge traffic
- Global scale

### Scale Estimates
- 100M monthly active users
- 10M daily rides
- 5M drivers
- Location updates every 4 seconds

```
┌─────────────────────────────────────────────────────────────┐
│                   Scale Summary                             │
├─────────────────────────────────────────────────────────────┤
│  Rides: 10M/day = ~115/second peak                         │
│  Location updates: 5M drivers × (1/4s) = 1.25M/second     │
│  Matching queries: ~115/second (ride requests)             │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. High-Level Design

```
┌─────────────────────────────────────────────────────────────┐
│                  High-Level Architecture                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────┐              ┌────────────┐                 │
│  │   Rider    │              │   Driver   │                 │
│  │    App     │              │    App     │                 │
│  └─────┬──────┘              └──────┬─────┘                 │
│        │                            │                        │
│        ▼                            ▼                        │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                  API Gateway                          │  │
│  └──────────────────────────────────────────────────────┘  │
│                          │                                  │
│      ┌───────────────────┼───────────────────┐             │
│      ▼                   ▼                   ▼             │
│ ┌──────────┐       ┌──────────┐       ┌──────────┐        │
│ │  Ride    │       │ Location │       │ Matching │        │
│ │ Service  │       │ Service  │       │ Service  │        │
│ └────┬─────┘       └────┬─────┘       └────┬─────┘        │
│      │                  │                  │               │
│      │                  ▼                  │               │
│      │           ┌──────────────┐          │               │
│      │           │   Location   │◀─────────┘               │
│      │           │     Index    │                          │
│      │           │ (QuadTree/   │                          │
│      │           │  Geohash)    │                          │
│      │           └──────────────┘                          │
│      │                                                      │
│      ▼                                                      │
│ ┌──────────────────────────────────────────────────────┐  │
│ │                  Ride Database                        │  │
│ └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Location Tracking

```
┌─────────────────────────────────────────────────────────────┐
│                  Location Tracking                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Driver location updates:                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Driver App ──(every 4s)──▶ Location Service           │ │
│  │                                                         │ │
│  │  Payload:                                               │ │
│  │  {                                                      │ │
│  │    "driver_id": "123",                                 │ │
│  │    "lat": 37.7749,                                     │ │
│  │    "lng": -122.4194,                                   │ │
│  │    "timestamp": 1640000000,                            │ │
│  │    "heading": 45,  // degrees                         │ │
│  │    "speed": 30     // mph                             │ │
│  │  }                                                      │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Spatial Indexing Options:                                  │
│                                                              │
│  1. Geohash                                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  • Encode lat/lng to string: "9q8yyk"                 │ │
│  │  • Similar locations share prefix                      │ │
│  │  • Easy to query nearby (prefix search)               │ │
│  │  • Works well with Redis/Cassandra                    │ │
│  │                                                         │ │
│  │  Geohash precision:                                    │ │
│  │  Length 6: ~1.2 km × 0.6 km                           │ │
│  │  Length 7: ~150 m × 150 m                              │ │
│  │  Length 8: ~19 m × 19 m                                │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  2. QuadTree                                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Divide space into quadrants recursively              │ │
│  │                                                         │ │
│  │  ┌─────────┬─────────┐                                │ │
│  │  │   NW    │   NE    │                                │ │
│  │  │ ┌──┬──┐ │         │                                │ │
│  │  │ │  │••│ │   •     │                                │ │
│  │  ├─┴──┴──┴─┼─────────┤                                │ │
│  │  │   SW    │   SE    │                                │ │
│  │  │         │    •    │                                │ │
│  │  └─────────┴─────────┘                                │ │
│  │                                                         │ │
│  │  Subdivide when too many points in cell               │ │
│  │  Good for uneven distribution                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Redis for real-time location:                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  GEOADD drivers 122.4194 37.7749 "driver:123"         │ │
│  │  GEORADIUS drivers 122.4194 37.7749 5 km              │ │
│  │                                                         │ │
│  │  Returns drivers within 5km radius                    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Ride Matching

```
┌─────────────────────────────────────────────────────────────┐
│                    Ride Matching                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Matching Flow:                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Rider requests ride                                │ │
│  │     → Pickup location, destination                     │ │
│  │                                                         │ │
│  │  2. Find nearby available drivers                      │ │
│  │     GEORADIUS drivers {pickup_lat} {pickup_lng} 5 km  │ │
│  │     Filter: status = AVAILABLE                        │ │
│  │     Returns: [driver1, driver2, driver3, ...]         │ │
│  │                                                         │ │
│  │  3. Rank drivers                                       │ │
│  │     Score = f(distance, ETA, rating, acceptance_rate) │ │
│  │     Sort by score                                      │ │
│  │                                                         │ │
│  │  4. Dispatch to best driver                            │ │
│  │     Send ride request to top-ranked driver            │ │
│  │     Wait for accept/reject (timeout: 15 seconds)      │ │
│  │                                                         │ │
│  │  5. If rejected or timeout                             │ │
│  │     Try next driver in list                           │ │
│  │                                                         │ │
│  │  6. If all reject, expand radius                       │ │
│  │     Retry with larger search radius                   │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Driver States:                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  OFFLINE ──▶ AVAILABLE ──▶ DISPATCHED ──▶ ON_TRIP     │ │
│  │     ▲                           │              │        │ │
│  │     │                           │ (reject/     │        │ │
│  │     │                           │  timeout)    │        │ │
│  │     │◀──────────────────────────┴──────────────┘        │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. ETA Calculation

```
┌─────────────────────────────────────────────────────────────┐
│                   ETA Calculation                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ETA Components:                                            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Road Network Graph                                 │ │
│  │     ┌───┐       ┌───┐       ┌───┐                     │ │
│  │     │ A │──5m──▶│ B │──3m──▶│ C │                     │ │
│  │     └───┘       └───┘       └───┘                     │ │
│  │       │                       ▲                        │ │
│  │       └─────────8m────────────┘                        │ │
│  │                                                         │ │
│  │  2. Real-time Traffic Data                             │ │
│  │     Edge weights adjusted by traffic conditions       │ │
│  │     Traffic factor: 1.0 (clear) to 3.0 (heavy)       │ │
│  │                                                         │ │
│  │  3. Historical Patterns                                │ │
│  │     • Time of day                                      │ │
│  │     • Day of week                                      │ │
│  │     • Special events                                   │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Routing Algorithm:                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Pre-compute: Road network partitioned into cells  │ │
│  │                                                         │ │
│  │  2. At query time:                                     │ │
│  │     a. Find cell containing origin/destination        │ │
│  │     b. Use pre-computed inter-cell distances          │ │
│  │     c. Adjust for real-time traffic                   │ │
│  │                                                         │ │
│  │  3. ML-based adjustment:                               │ │
│  │     • Historical accuracy                              │ │
│  │     • Current conditions                               │ │
│  │                                                         │ │
│  │  Similar to: Contraction Hierarchies, A* with heuristics│ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Caching:                                                   │
│  • Cache common routes (airport → downtown)                │
│  • TTL based on traffic variability                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Fare Calculation

```
┌─────────────────────────────────────────────────────────────┐
│                   Fare Calculation                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Fare Formula:                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Base Fare = $2.00                                     │ │
│  │  Per Mile  = $1.50                                     │ │
│  │  Per Minute = $0.25                                    │ │
│  │  Booking Fee = $1.00                                   │ │
│  │  Surge Multiplier = 1.0 - 3.0                          │ │
│  │                                                         │ │
│  │  Fare = (Base + (Distance × PerMile) + (Time × PerMin))│ │
│  │         × SurgeMultiplier + BookingFee                 │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Surge Pricing:                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Demand/Supply ratio per geographic zone              │ │
│  │                                                         │ │
│  │  Zone Demand: # of ride requests in last 5 minutes    │ │
│  │  Zone Supply: # of available drivers in zone          │ │
│  │                                                         │ │
│  │  If Demand/Supply > 1.5 → Surge 1.5x                  │ │
│  │  If Demand/Supply > 2.0 → Surge 2.0x                  │ │
│  │  If Demand/Supply > 3.0 → Surge 3.0x (cap)            │ │
│  │                                                         │ │
│  │  Update surge every 1-2 minutes                       │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Upfront Pricing:                                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Show fixed price before ride (not metered)           │ │
│  │  Based on estimated distance and time                 │ │
│  │  Protects rider from traffic delays                   │ │
│  │  Uber absorbs variance (within limits)                │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Data Model

```
┌─────────────────────────────────────────────────────────────┐
│                     Data Model                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Users                                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ user_id          │ BIGINT (PK)                          │ │
│  │ type             │ ENUM(rider, driver)                  │ │
│  │ name             │ VARCHAR(100)                         │ │
│  │ phone            │ VARCHAR(20)                          │ │
│  │ email            │ VARCHAR(255)                         │ │
│  │ rating           │ DECIMAL(2,1)                         │ │
│  │ created_at       │ TIMESTAMP                            │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Rides (Sharded by ride_id)                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ ride_id          │ BIGINT (PK)                          │ │
│  │ rider_id         │ BIGINT (FK)                          │ │
│  │ driver_id        │ BIGINT (FK)                          │ │
│  │ status           │ ENUM(requested, matched, started,   │ │
│  │                  │      completed, cancelled)           │ │
│  │ pickup_lat       │ DECIMAL(9,6)                         │ │
│  │ pickup_lng       │ DECIMAL(9,6)                         │ │
│  │ dropoff_lat      │ DECIMAL(9,6)                         │ │
│  │ dropoff_lng      │ DECIMAL(9,6)                         │ │
│  │ fare_amount      │ DECIMAL(10,2)                        │ │
│  │ distance_miles   │ DECIMAL(6,2)                         │ │
│  │ duration_minutes │ INT                                  │ │
│  │ requested_at     │ TIMESTAMP                            │ │
│  │ started_at       │ TIMESTAMP                            │ │
│  │ completed_at     │ TIMESTAMP                            │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Driver Locations (Redis GEOSPATIAL)                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Key: drivers:active                                     │ │
│  │ Type: GEOSPATIAL                                        │ │
│  │ Members: driver_id with lat/lng                        │ │
│  │ TTL on individual entries: 30 seconds                  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. Summary

| Component | Choice | Reason |
|-----------|--------|--------|
| **Location Index** | Redis GEO + Geohash | Fast proximity queries |
| **Ride Data** | PostgreSQL (sharded) | ACID for transactions |
| **Location Updates** | Kafka | High throughput streaming |
| **Routing** | Pre-computed + real-time | Balance accuracy/speed |
| **Matching** | Custom algorithm | Domain-specific ranking |

---

[Back to System Designs](../README.md)
