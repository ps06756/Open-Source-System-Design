# Design Uber

Design a ride-sharing platform like Uber.

## 1. Requirements

### Functional Requirements
- Riders can request rides
- Drivers can accept rides
- Real-time location tracking
- Match riders with nearby drivers
- Calculate fare and ETA
- Payment processing
- Rating system

### Non-Functional Requirements
- Low latency matching (<30s)
- High availability
- Real-time location updates (every 3-4 seconds)
- Scale: 20M DAU, 15M rides/day

### Capacity Estimation

```
Rides: 15M/day ≈ 175 rides/sec
Active drivers: 5M concurrent at peak
Location updates: 5M × 1 per 4s = 1.25M/sec

Storage:
Ride history: 15M × 365 = 5.5B rides/year
Location data: Ephemeral (real-time only)
```

## 2. High-Level Design

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌──────────┐                                ┌──────────┐     │
│  │  Rider   │                                │  Driver  │     │
│  │   App    │                                │   App    │     │
│  └────┬─────┘                                └─────┬────┘     │
│       │                                            │          │
│       └────────────────┬───────────────────────────┘          │
│                        ▼                                       │
│                 ┌─────────────┐                               │
│                 │   API GW    │                               │
│                 └──────┬──────┘                               │
│                        │                                       │
│    ┌───────────────────┼───────────────────┐                  │
│    │                   │                   │                  │
│    ▼                   ▼                   ▼                  │
│ ┌──────────┐    ┌──────────┐       ┌──────────────┐          │
│ │ Location │    │ Matching │       │ Ride Service │          │
│ │ Service  │    │ Service  │       │              │          │
│ └────┬─────┘    └────┬─────┘       └──────────────┘          │
│      │               │                                        │
│      ▼               ▼                                        │
│ ┌──────────┐    ┌──────────┐       ┌──────────────┐          │
│ │  Redis   │    │Geospatial│       │ PostgreSQL   │          │
│ │ (Real-   │    │  Index   │       │ (Rides,      │          │
│ │  time)   │    │          │       │  Users)      │          │
│ └──────────┘    └──────────┘       └──────────────┘          │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 3. Location Service

```
┌────────────────────────────────────────────────────────────────┐
│                    Location Tracking                           │
│                                                                │
│  Driver App:                                                   │
│  - Send location every 4 seconds via WebSocket                │
│  - Include: driver_id, lat, lng, heading, speed               │
│                                                                │
│  Location Service:                                             │
│  - Receive updates via WebSocket                              │
│  - Store in Redis (ephemeral, 30s TTL)                       │
│  - Update geospatial index                                    │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Redis Storage                                           │  │
│  │                                                          │  │
│  │  Key: driver:location:{driver_id}                       │  │
│  │  Value: {lat, lng, heading, speed, timestamp}           │  │
│  │  TTL: 30 seconds (driver goes offline if no update)    │  │
│  │                                                          │  │
│  │  Geospatial Index (Redis GEO):                          │  │
│  │  GEOADD drivers:active {lng} {lat} {driver_id}         │  │
│  │  GEORADIUS drivers:active {lng} {lat} 5 km             │  │
│  │                                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Location Update Handler

```python
class LocationService:
    def __init__(self, redis):
        self.redis = redis

    async def update_location(self, driver_id, lat, lng, heading, speed):
        """Handle driver location update"""
        pipe = self.redis.pipeline()

        # Store detailed location
        location_key = f"driver:location:{driver_id}"
        pipe.hset(location_key, {
            'lat': lat,
            'lng': lng,
            'heading': heading,
            'speed': speed,
            'timestamp': time.time()
        })
        pipe.expire(location_key, 30)

        # Update geospatial index
        pipe.geoadd('drivers:active', lng, lat, driver_id)

        await pipe.execute()

    async def get_nearby_drivers(self, lat, lng, radius_km=5, limit=20):
        """Find drivers within radius"""
        drivers = await self.redis.georadius(
            'drivers:active',
            lng, lat,
            radius_km, 'km',
            withdist=True,
            sort='ASC',
            count=limit
        )

        # Enrich with driver details
        result = []
        for driver_id, distance in drivers:
            location = await self.redis.hgetall(f"driver:location:{driver_id}")
            if location:
                result.append({
                    'driver_id': driver_id,
                    'distance_km': distance,
                    **location
                })

        return result
```

## 4. Matching Service

```
┌────────────────────────────────────────────────────────────────┐
│                    Driver Matching                             │
│                                                                │
│  1. Rider requests ride                                       │
│  2. Find nearby available drivers                             │
│  3. Score each driver                                         │
│  4. Send request to best driver                               │
│  5. If declined/timeout, try next driver                      │
│                                                                │
│  Scoring factors:                                              │
│  - Distance to pickup (primary)                               │
│  - Driver rating                                               │
│  - ETA (considering traffic)                                  │
│  - Driver acceptance rate                                      │
│  - Supply/demand in area                                       │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Matching Algorithm                                      │  │
│  │                                                          │  │
│  │  score = w1 * (1/distance)                             │  │
│  │        + w2 * rating                                    │  │
│  │        + w3 * acceptance_rate                          │  │
│  │        + w4 * (1/estimated_eta)                        │  │
│  │                                                          │  │
│  │  Select top N, send offers in parallel or sequential   │  │
│  │                                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Matching Implementation

```python
class MatchingService:
    def __init__(self, location_service, eta_service):
        self.location = location_service
        self.eta = eta_service
        self.timeout_seconds = 15

    async def find_driver(self, ride_request):
        """Match rider with best available driver"""
        pickup = ride_request['pickup_location']

        # Get nearby available drivers
        candidates = await self.location.get_nearby_drivers(
            pickup['lat'], pickup['lng'],
            radius_km=10,
            limit=50
        )

        # Filter to available drivers
        available = await self.filter_available(candidates)

        # Score and rank
        scored = []
        for driver in available:
            score = await self.calculate_score(driver, ride_request)
            scored.append((driver, score))

        scored.sort(key=lambda x: -x[1])

        # Send to drivers until one accepts
        for driver, score in scored[:10]:
            accepted = await self.send_offer(driver, ride_request)
            if accepted:
                return driver

        return None  # No driver found

    async def calculate_score(self, driver, request):
        pickup = request['pickup_location']

        # ETA from driver to pickup
        eta = await self.eta.calculate(
            driver['lat'], driver['lng'],
            pickup['lat'], pickup['lng']
        )

        # Get driver stats
        stats = await self.get_driver_stats(driver['driver_id'])

        score = (
            0.5 * (1 / (driver['distance_km'] + 0.1)) +
            0.2 * stats['rating'] / 5.0 +
            0.15 * stats['acceptance_rate'] +
            0.15 * (1 / (eta + 1))
        )

        return score

    async def send_offer(self, driver, ride_request):
        """Send ride offer to driver"""
        offer = {
            'ride_id': ride_request['ride_id'],
            'pickup': ride_request['pickup_location'],
            'dropoff': ride_request['dropoff_location'],
            'estimated_fare': ride_request['estimated_fare']
        }

        # Push to driver via WebSocket
        await self.push_to_driver(driver['driver_id'], offer)

        # Wait for response
        response = await self.wait_for_response(
            driver['driver_id'],
            ride_request['ride_id'],
            timeout=self.timeout_seconds
        )

        return response == 'accepted'
```

## 5. Geospatial Indexing

```
┌────────────────────────────────────────────────────────────────┐
│                 Geospatial Indexing Options                    │
│                                                                │
│  Option 1: Redis GEO (Simple)                                 │
│  - Built-in GEORADIUS command                                 │
│  - Good for moderate scale                                    │
│                                                                │
│  Option 2: Geohash (Grid-based)                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  World divided into grid cells                          │  │
│  │                                                          │  │
│  │  ┌───┬───┬───┬───┐                                      │  │
│  │  │ a │ b │ c │ d │  Level 1 (large cells)              │  │
│  │  ├───┼───┼───┼───┤                                      │  │
│  │  │ e │ f │ g │ h │                                      │  │
│  │  ├───┼───┼───┼───┤                                      │  │
│  │  │ i │ j │ k │ l │                                      │  │
│  │  └───┴───┴───┴───┘                                      │  │
│  │                                                          │  │
│  │  Each cell subdivides recursively                       │  │
│  │  Geohash "9q8yy" = specific ~150m x 150m cell          │  │
│  │                                                          │  │
│  │  Nearby search: Query current cell + neighbors         │  │
│  │                                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  Option 3: QuadTree (Dynamic)                                 │
│  - Adaptive subdivision based on density                     │
│  - More complex but efficient for uneven distribution        │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Geohash Implementation

```python
import geohash

class GeospatialIndex:
    def __init__(self, redis):
        self.redis = redis
        self.precision = 6  # ~1km cells

    def update_driver(self, driver_id, lat, lng):
        """Update driver location in index"""
        # Calculate geohash
        cell = geohash.encode(lat, lng, self.precision)

        # Get old cell and remove if changed
        old_cell = self.redis.get(f"driver:cell:{driver_id}")
        if old_cell and old_cell != cell:
            self.redis.srem(f"cell:{old_cell}", driver_id)

        # Add to new cell
        self.redis.sadd(f"cell:{cell}", driver_id)
        self.redis.set(f"driver:cell:{driver_id}", cell)
        self.redis.expire(f"driver:cell:{driver_id}", 30)

    def get_nearby_drivers(self, lat, lng, radius_km):
        """Get drivers in nearby cells"""
        # Get cell for search location
        center_cell = geohash.encode(lat, lng, self.precision)

        # Get neighboring cells
        neighbors = geohash.neighbors(center_cell)
        cells = [center_cell] + neighbors

        # Get all drivers in these cells
        drivers = set()
        for cell in cells:
            cell_drivers = self.redis.smembers(f"cell:{cell}")
            drivers.update(cell_drivers)

        return list(drivers)
```

## 6. Ride State Machine

```
┌────────────────────────────────────────────────────────────────┐
│                    Ride State Machine                          │
│                                                                │
│  REQUESTED ──▶ MATCHING ──▶ DRIVER_ASSIGNED ──▶ EN_ROUTE      │
│      │            │              │                  │          │
│      │            │              │                  ▼          │
│      │            │              │            ARRIVED          │
│      │            │              │                  │          │
│      │            │              │                  ▼          │
│      │            │              │             IN_PROGRESS     │
│      │            │              │                  │          │
│      │            │              │                  ▼          │
│      │            │              │             COMPLETED       │
│      │            │              │                              │
│      ▼            ▼              ▼                              │
│  CANCELLED   NO_DRIVERS    CANCELLED                           │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Ride Service

```python
class RideService:
    async def request_ride(self, rider_id, pickup, dropoff):
        """Create ride request"""
        ride = {
            'ride_id': generate_id(),
            'rider_id': rider_id,
            'pickup_location': pickup,
            'dropoff_location': dropoff,
            'status': 'REQUESTED',
            'created_at': time.time()
        }

        # Calculate estimates
        ride['estimated_fare'] = await self.calculate_fare(pickup, dropoff)
        ride['estimated_eta'] = await self.calculate_eta(pickup, dropoff)

        # Save ride
        await self.db.insert('rides', ride)

        # Start matching
        ride['status'] = 'MATCHING'
        driver = await self.matching.find_driver(ride)

        if driver:
            ride['driver_id'] = driver['driver_id']
            ride['status'] = 'DRIVER_ASSIGNED'
            await self.notify_rider(ride)
        else:
            ride['status'] = 'NO_DRIVERS'
            await self.notify_rider_no_drivers(ride)

        await self.db.update('rides', ride)
        return ride

    async def calculate_fare(self, pickup, dropoff):
        """Calculate estimated fare"""
        route = await self.routing.get_route(pickup, dropoff)

        base_fare = 2.50
        per_km = 1.50
        per_minute = 0.25

        fare = base_fare + (route['distance_km'] * per_km) + \
               (route['duration_minutes'] * per_minute)

        # Apply surge pricing if applicable
        surge = await self.get_surge_multiplier(pickup)
        fare *= surge

        return round(fare, 2)
```

## 7. Surge Pricing

```
┌────────────────────────────────────────────────────────────────┐
│                    Surge Pricing                               │
│                                                                │
│  Based on supply/demand ratio in an area                      │
│                                                                │
│  1. Divide city into hexagonal zones                         │
│  2. Track: riders requesting, drivers available              │
│  3. Calculate ratio: demand / supply                         │
│  4. Apply multiplier based on ratio                          │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Demand/Supply Ratio  │  Surge Multiplier               │  │
│  ├───────────────────────┼────────────────────────────────│  │
│  │  < 1.0                │  1.0x (no surge)               │  │
│  │  1.0 - 1.5            │  1.25x                         │  │
│  │  1.5 - 2.0            │  1.5x                          │  │
│  │  2.0 - 3.0            │  1.75x                         │  │
│  │  > 3.0                │  2.0x+                         │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  Implementation:                                               │
│  - Use H3 hexagonal grid (Uber's system)                     │
│  - Real-time counters in Redis                               │
│  - Recalculate every 2 minutes                               │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 8. ETA Calculation

```python
class ETAService:
    def __init__(self, routing_service):
        self.routing = routing_service
        self.cache = {}

    async def calculate_eta(self, origin, destination):
        """Calculate ETA considering real-time traffic"""
        # Get base route
        route = await self.routing.get_route(origin, destination)

        # Get traffic conditions
        traffic = await self.get_traffic_conditions(route['segments'])

        # Adjust for traffic
        adjusted_duration = 0
        for segment, condition in zip(route['segments'], traffic):
            base_time = segment['duration']
            traffic_factor = self.traffic_factors[condition]
            adjusted_duration += base_time * traffic_factor

        return {
            'duration_minutes': adjusted_duration,
            'distance_km': route['distance_km'],
            'traffic_level': self.summarize_traffic(traffic)
        }

    traffic_factors = {
        'free': 1.0,
        'light': 1.1,
        'moderate': 1.3,
        'heavy': 1.6,
        'standstill': 2.5
    }
```

## 9. Key Takeaways

1. **Real-time location** - WebSocket + Redis for ephemeral data
2. **Geospatial indexing** - Geohash or quadtree for fast nearby search
3. **Matching algorithm** - Multi-factor scoring with fallback
4. **Surge pricing** - Dynamic pricing based on supply/demand
5. **State machine** - Clear ride lifecycle management
6. **ETA with traffic** - Real-time traffic adjustments

---

Next: [Google Maps](../03-google-maps/README.md)
