# Design a Hotel Booking System

Design a hotel booking system like Booking.com or Airbnb.

## 1. Requirements

### Functional Requirements
- Search hotels by location, dates, guests
- View hotel details and room availability
- Book rooms with payment
- Manage reservations (view, cancel, modify)
- Reviews and ratings
- Host management (for Airbnb-style)

### Non-Functional Requirements
- High availability
- Handle concurrent bookings (no double booking)
- Low latency search (<500ms)
- Scale: 10M hotels, 1M bookings/day

### Capacity Estimation

```
Hotels: 10M
Rooms: 10M × 20 rooms = 200M rooms
Bookings: 1M/day ≈ 12 TPS average
Search queries: 100M/day ≈ 1,200 QPS

Storage:
Hotels: 10M × 10 KB = 100 GB
Rooms: 200M × 1 KB = 200 GB
Bookings: 1M × 1 KB × 365 = 365 GB/year
```

## 2. High-Level Design

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌────────┐     ┌──────────────┐     ┌────────────────────┐   │
│  │ Client │────▶│ Load Balancer│────▶│    API Gateway     │   │
│  └────────┘     └──────────────┘     └─────────┬──────────┘   │
│                                                 │              │
│         ┌───────────────────────────────────────┼───────────┐ │
│         │                                       │           │ │
│         ▼                                       ▼           │ │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │ │
│  │   Search    │  │   Booking   │  │    Inventory        │ │ │
│  │   Service   │  │   Service   │  │    Service          │ │ │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘ │ │
│         │                │                     │            │ │
│         ▼                ▼                     ▼            │ │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │ │
│  │Elasticsearch│  │ PostgreSQL  │  │    Redis Cache      │ │ │
│  │  (Search)   │  │ (Bookings)  │  │  (Availability)     │ │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │ │
│                                                              │ │
└────────────────────────────────────────────────────────────────┘
```

## 3. Search Service

```
┌────────────────────────────────────────────────────────────────┐
│                    Hotel Search                                │
│                                                                │
│  Search criteria:                                              │
│  - Location (city, coordinates, radius)                       │
│  - Check-in/check-out dates                                   │
│  - Number of guests/rooms                                     │
│  - Filters (price, amenities, rating)                        │
│  - Sort (price, rating, distance)                            │
│                                                                │
│  Elasticsearch index:                                          │
│  {                                                             │
│    "hotel_id": "H123",                                        │
│    "name": "Grand Hotel",                                     │
│    "location": {"lat": 40.7128, "lon": -74.0060},            │
│    "city": "New York",                                        │
│    "star_rating": 4,                                          │
│    "user_rating": 8.5,                                        │
│    "price_range": {"min": 150, "max": 500},                  │
│    "amenities": ["wifi", "pool", "gym"],                     │
│    "room_types": ["standard", "deluxe", "suite"]             │
│  }                                                             │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Search Implementation

```python
class SearchService:
    def __init__(self, es_client, inventory_service):
        self.es = es_client
        self.inventory = inventory_service

    async def search(self, criteria):
        """Search hotels with availability check"""
        # Build Elasticsearch query
        query = self.build_query(criteria)

        # Search hotels
        hotel_results = await self.es.search(
            index='hotels',
            body=query,
            size=50
        )

        hotel_ids = [h['_id'] for h in hotel_results['hits']['hits']]

        # Check availability for date range
        availability = await self.inventory.check_availability(
            hotel_ids,
            criteria.check_in,
            criteria.check_out,
            criteria.rooms_needed
        )

        # Filter and enrich results
        available_hotels = []
        for hotel in hotel_results['hits']['hits']:
            hotel_id = hotel['_id']
            if hotel_id in availability and availability[hotel_id]['available']:
                hotel['_source']['lowest_price'] = availability[hotel_id]['lowest_price']
                available_hotels.append(hotel['_source'])

        return self.sort_results(available_hotels, criteria.sort_by)

    def build_query(self, criteria):
        query = {
            'bool': {
                'must': [],
                'filter': []
            }
        }

        # Location filter
        if criteria.location:
            query['bool']['filter'].append({
                'geo_distance': {
                    'distance': '10km',
                    'location': criteria.location
                }
            })

        # Amenities filter
        if criteria.amenities:
            for amenity in criteria.amenities:
                query['bool']['filter'].append({
                    'term': {'amenities': amenity}
                })

        # Price range filter
        if criteria.max_price:
            query['bool']['filter'].append({
                'range': {
                    'price_range.min': {'lte': criteria.max_price}
                }
            })

        return {'query': query}
```

## 4. Inventory Management

```
┌────────────────────────────────────────────────────────────────┐
│                Inventory Management                            │
│                                                                │
│  Track room availability for each date                        │
│                                                                │
│  Database schema:                                              │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  CREATE TABLE room_inventory (                           │  │
│  │      hotel_id UUID,                                      │  │
│  │      room_type VARCHAR(50),                             │  │
│  │      date DATE,                                          │  │
│  │      total_rooms INT,                                   │  │
│  │      booked_rooms INT,                                  │  │
│  │      price DECIMAL(10,2),                               │  │
│  │      PRIMARY KEY (hotel_id, room_type, date)           │  │
│  │  );                                                       │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  Alternative: Redis for hot data                              │
│  Key: inventory:{hotel_id}:{room_type}:{date}                │
│  Value: {total: 10, booked: 7, price: 199.99}                │
│                                                                │
│  Pre-populate inventory for next 365 days                    │
│  Update on booking/cancellation                               │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Inventory Service

```python
class InventoryService:
    def __init__(self, redis, db):
        self.redis = redis
        self.db = db

    async def check_availability(self, hotel_ids, check_in, check_out, rooms_needed):
        """Check room availability for date range"""
        results = {}
        dates = self.get_date_range(check_in, check_out)

        for hotel_id in hotel_ids:
            room_types = await self.get_room_types(hotel_id)
            hotel_available = False
            lowest_price = float('inf')

            for room_type in room_types:
                available = True
                total_price = 0

                for date in dates:
                    key = f"inventory:{hotel_id}:{room_type}:{date}"
                    inv = await self.redis.hgetall(key)

                    if not inv:
                        inv = await self.load_from_db(hotel_id, room_type, date)

                    available_rooms = int(inv['total']) - int(inv['booked'])
                    if available_rooms < rooms_needed:
                        available = False
                        break

                    total_price += float(inv['price'])

                if available:
                    hotel_available = True
                    lowest_price = min(lowest_price, total_price)

            results[hotel_id] = {
                'available': hotel_available,
                'lowest_price': lowest_price if hotel_available else None
            }

        return results

    async def reserve_inventory(self, hotel_id, room_type, check_in, check_out, rooms):
        """Reserve rooms (with optimistic locking)"""
        dates = self.get_date_range(check_in, check_out)

        async with self.db.transaction():
            for date in dates:
                # Lock row and check availability
                result = await self.db.execute('''
                    UPDATE room_inventory
                    SET booked_rooms = booked_rooms + $1
                    WHERE hotel_id = $2
                      AND room_type = $3
                      AND date = $4
                      AND (total_rooms - booked_rooms) >= $1
                    RETURNING *
                ''', rooms, hotel_id, room_type, date)

                if not result:
                    raise InsufficientInventoryError()

                # Update cache
                key = f"inventory:{hotel_id}:{room_type}:{date}"
                await self.redis.hincrby(key, 'booked', rooms)

        return True
```

## 5. Booking Service

```
┌────────────────────────────────────────────────────────────────┐
│                    Booking Flow                                │
│                                                                │
│  1. User selects room and dates                               │
│  2. System creates temporary hold (5 min)                     │
│  3. User enters payment details                               │
│  4. System processes payment                                  │
│  5. System confirms booking                                   │
│  6. Release hold or convert to booking                        │
│                                                                │
│  State machine:                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                                                          │  │
│  │  INITIATED → HELD → PAYMENT_PENDING → CONFIRMED         │  │
│  │      │         │           │              │              │  │
│  │      │         │           │              ▼              │  │
│  │      └────────▶│───────────│─────────▶CANCELLED         │  │
│  │      (timeout) │  (timeout)│    (user/system)           │  │
│  │                │           │                             │  │
│  │                ▼           ▼                             │  │
│  │             EXPIRED   PAYMENT_FAILED                     │  │
│  │                                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Booking Implementation

```python
class BookingService:
    HOLD_TIMEOUT = 300  # 5 minutes

    async def create_booking(self, request):
        """Create a new booking with temporary hold"""
        booking_id = generate_id()

        # Calculate total price
        total_price = await self.inventory.calculate_price(
            request.hotel_id,
            request.room_type,
            request.check_in,
            request.check_out,
            request.rooms
        )

        # Create booking record
        booking = Booking(
            id=booking_id,
            hotel_id=request.hotel_id,
            user_id=request.user_id,
            room_type=request.room_type,
            check_in=request.check_in,
            check_out=request.check_out,
            rooms=request.rooms,
            total_price=total_price,
            status='INITIATED'
        )
        await self.db.insert('bookings', booking)

        # Try to hold inventory
        try:
            await self.inventory.reserve_inventory(
                request.hotel_id,
                request.room_type,
                request.check_in,
                request.check_out,
                request.rooms
            )
            booking.status = 'HELD'
            booking.hold_expires_at = datetime.now() + timedelta(seconds=self.HOLD_TIMEOUT)
            await self.db.update('bookings', booking)

            # Schedule hold expiration
            await self.scheduler.schedule(
                'expire_hold',
                booking_id,
                delay=self.HOLD_TIMEOUT
            )

        except InsufficientInventoryError:
            booking.status = 'FAILED'
            await self.db.update('bookings', booking)
            raise

        return booking

    async def confirm_booking(self, booking_id, payment_token):
        """Confirm booking with payment"""
        booking = await self.db.get('bookings', booking_id)

        if booking.status != 'HELD':
            raise InvalidBookingStateError()

        if datetime.now() > booking.hold_expires_at:
            booking.status = 'EXPIRED'
            await self.release_inventory(booking)
            raise BookingExpiredError()

        # Process payment
        booking.status = 'PAYMENT_PENDING'
        await self.db.update('bookings', booking)

        try:
            payment_result = await self.payment_service.charge(
                amount=booking.total_price,
                token=payment_token,
                booking_id=booking_id
            )

            booking.status = 'CONFIRMED'
            booking.payment_id = payment_result.id
            booking.confirmation_code = self.generate_confirmation_code()

        except PaymentError as e:
            booking.status = 'PAYMENT_FAILED'
            await self.release_inventory(booking)

        await self.db.update('bookings', booking)
        return booking
```

## 6. Handling Concurrent Bookings

```
┌────────────────────────────────────────────────────────────────┐
│           Preventing Double Booking                            │
│                                                                │
│  Problem: Two users try to book last room simultaneously     │
│                                                                │
│  Solution 1: Pessimistic Locking                             │
│  - Lock inventory row during booking                         │
│  - Simple but reduces throughput                             │
│                                                                │
│  Solution 2: Optimistic Locking (preferred)                  │
│  - Use version number or condition check                     │
│  - Retry on conflict                                          │
│                                                                │
│  UPDATE room_inventory                                        │
│  SET booked_rooms = booked_rooms + 1,                        │
│      version = version + 1                                    │
│  WHERE hotel_id = ? AND date = ?                             │
│    AND (total_rooms - booked_rooms) >= 1                     │
│    AND version = ?                                            │
│                                                                │
│  Solution 3: Redis atomic operations                         │
│  - WATCH/MULTI/EXEC transaction                              │
│  - Fast, but requires careful cache invalidation            │
│                                                                │
│  WATCH inventory:H123:standard:2024-01-15                    │
│  MULTI                                                        │
│  HINCRBY inventory:H123:standard:2024-01-15 booked 1        │
│  EXEC  # Fails if key changed since WATCH                    │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 7. Database Schema

```sql
-- Hotels
CREATE TABLE hotels (
    id UUID PRIMARY KEY,
    name VARCHAR(255),
    description TEXT,
    address TEXT,
    city VARCHAR(100),
    country VARCHAR(100),
    latitude DECIMAL(9,6),
    longitude DECIMAL(9,6),
    star_rating INT,
    amenities JSONB,
    policies JSONB,
    created_at TIMESTAMP
);

-- Room Types
CREATE TABLE room_types (
    id UUID PRIMARY KEY,
    hotel_id UUID REFERENCES hotels(id),
    name VARCHAR(100),
    description TEXT,
    max_guests INT,
    bed_configuration VARCHAR(100),
    amenities JSONB,
    base_price DECIMAL(10,2)
);

-- Room Inventory (per day)
CREATE TABLE room_inventory (
    hotel_id UUID,
    room_type_id UUID,
    date DATE,
    total_rooms INT,
    booked_rooms INT DEFAULT 0,
    price DECIMAL(10,2),
    PRIMARY KEY (hotel_id, room_type_id, date)
);

-- Bookings
CREATE TABLE bookings (
    id UUID PRIMARY KEY,
    hotel_id UUID,
    user_id UUID,
    room_type_id UUID,
    check_in DATE,
    check_out DATE,
    rooms INT,
    guests INT,
    total_price DECIMAL(10,2),
    status VARCHAR(20),
    confirmation_code VARCHAR(20),
    payment_id UUID,
    created_at TIMESTAMP,
    INDEX idx_user (user_id),
    INDEX idx_hotel_dates (hotel_id, check_in, check_out)
);
```

## 8. Key Takeaways

1. **Inventory per day** - track availability for each date
2. **Temporary holds** - prevent overselling during checkout
3. **Optimistic locking** - handle concurrent bookings
4. **Search + filter** - Elasticsearch with availability check
5. **State machine** - clear booking lifecycle
6. **Cache hot data** - Redis for availability checks

---

Next: [Search Engine](../09-search-engine/README.md)
