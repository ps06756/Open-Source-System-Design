# Design Google Maps

Design a mapping and navigation system like Google Maps.

## 1. Requirements

### Functional Requirements
- Display maps at various zoom levels
- Search for places/addresses
- Get directions (driving, walking, transit)
- Real-time traffic information
- Turn-by-turn navigation
- Estimated time of arrival (ETA)

### Non-Functional Requirements
- Low latency map tile loading (<200ms)
- High availability
- Accurate directions and ETAs
- Scale: 1B+ users, 1B km of roads

### Capacity Estimation

```
Map tiles: ~20 zoom levels, millions of tiles per level
Total tiles: ~100 billion tiles
Average tile size: 20 KB
Storage: 100B × 20 KB = 2 PB (compressed)

Requests: 1B users × 10 tiles/session × 5 sessions/day
        = 50B tile requests/day ≈ 600K/sec

Route calculations: 100M/day ≈ 1,200/sec
```

## 2. High-Level Design

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌────────┐     ┌──────────────┐     ┌────────────────────┐   │
│  │ Client │────▶│ Load Balancer│────▶│    API Servers     │   │
│  └────────┘     └──────────────┘     └─────────┬──────────┘   │
│       │                                         │              │
│       │                    ┌────────────────────┼───────────┐ │
│       │                    │                    │           │ │
│       │                    ▼                    ▼           │ │
│       │             ┌─────────────┐     ┌─────────────────┐ │ │
│       │             │   Routing   │     │    Places       │ │ │
│       │             │   Service   │     │    Service      │ │ │
│       │             └──────┬──────┘     └─────────────────┘ │ │
│       │                    │                                 │ │
│       │                    ▼                                 │ │
│       │             ┌─────────────┐     ┌─────────────────┐ │ │
│       │             │ Road Graph  │     │  Traffic        │ │ │
│       │             │ (Neo4j)     │     │  Service        │ │ │
│       │             └─────────────┘     └─────────────────┘ │ │
│       │                                                      │ │
│       ▼                                                      │ │
│  ┌─────────────┐                                            │ │
│  │     CDN     │◀── Map Tiles                               │ │
│  └─────────────┘                                            │ │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 3. Map Tiles

```
┌────────────────────────────────────────────────────────────────┐
│                    Map Tile System                             │
│                                                                │
│  Zoom Level 0: 1 tile (whole world)                           │
│  Zoom Level 1: 4 tiles (2×2)                                  │
│  Zoom Level 2: 16 tiles (4×4)                                 │
│  ...                                                           │
│  Zoom Level 20: 2^40 tiles (~1 trillion)                      │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Tile Addressing (z/x/y)                                │  │
│  │                                                          │  │
│  │  Zoom 0:        Zoom 1:          Zoom 2:                │  │
│  │  ┌─────┐        ┌───┬───┐        ┌─┬─┬─┬─┐             │  │
│  │  │ 0/0 │        │0/0│0/1│        │ │ │ │ │             │  │
│  │  │     │        ├───┼───┤        ├─┼─┼─┼─┤             │  │
│  │  └─────┘        │1/0│1/1│        │ │ │ │ │             │  │
│  │                 └───┴───┘        ├─┼─┼─┼─┤             │  │
│  │                                  │ │ │ │ │             │  │
│  │                                  └─┴─┴─┴─┘             │  │
│  │                                                          │  │
│  │  URL: /tiles/{z}/{x}/{y}.png                            │  │
│  │  Example: /tiles/15/16384/10896.png                     │  │
│  │                                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  Vector Tiles (modern):                                       │
│  - Send geometry data, render client-side                    │
│  - Smaller size, dynamic styling                             │
│  - Format: MVT (Mapbox Vector Tiles)                         │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Tile Generation Pipeline

```python
class TileGenerator:
    def __init__(self):
        self.zoom_levels = range(0, 21)

    def generate_tiles(self, region):
        """Generate tiles for a region"""
        for zoom in self.zoom_levels:
            tiles = self.get_tiles_for_region(region, zoom)

            for tile in tiles:
                # Render tile
                image = self.render_tile(tile, zoom)

                # Save to storage
                key = f"tiles/{zoom}/{tile.x}/{tile.y}.png"
                storage.put(key, image)

    def render_tile(self, tile, zoom):
        """Render a single tile"""
        # Get data for this tile's bounding box
        bbox = self.tile_to_bbox(tile, zoom)

        # Query relevant features
        roads = self.get_roads(bbox, zoom)
        buildings = self.get_buildings(bbox, zoom)
        labels = self.get_labels(bbox, zoom)

        # Render layers
        canvas = Image.new('RGBA', (256, 256))
        self.draw_roads(canvas, roads, zoom)
        self.draw_buildings(canvas, buildings, zoom)
        self.draw_labels(canvas, labels, zoom)

        return canvas

    def get_roads(self, bbox, zoom):
        """Get roads for bounding box, filtered by zoom level"""
        if zoom < 5:
            # Only highways
            return self.query_roads(bbox, ['motorway', 'trunk'])
        elif zoom < 10:
            # Major roads
            return self.query_roads(bbox, ['motorway', 'trunk', 'primary'])
        else:
            # All roads
            return self.query_roads(bbox, all_types=True)
```

## 4. Road Graph and Routing

```
┌────────────────────────────────────────────────────────────────┐
│                    Road Network Graph                          │
│                                                                │
│  Nodes: Intersections, endpoints                              │
│  Edges: Road segments                                          │
│                                                                │
│  Edge properties:                                              │
│  - length (meters)                                            │
│  - speed_limit                                                 │
│  - road_type (highway, local, etc.)                          │
│  - one_way                                                     │
│  - restrictions (no turns, time-based)                        │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Graph Storage                                           │  │
│  │                                                          │  │
│  │  CREATE TABLE nodes (                                    │  │
│  │      node_id BIGINT PRIMARY KEY,                        │  │
│  │      lat DOUBLE,                                         │  │
│  │      lng DOUBLE                                          │  │
│  │  );                                                       │  │
│  │                                                          │  │
│  │  CREATE TABLE edges (                                    │  │
│  │      edge_id BIGINT PRIMARY KEY,                        │  │
│  │      from_node BIGINT,                                  │  │
│  │      to_node BIGINT,                                    │  │
│  │      length_m INT,                                       │  │
│  │      speed_limit INT,                                    │  │
│  │      road_type VARCHAR(20)                              │  │
│  │  );                                                       │  │
│  │                                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  Scale: 1B+ edges globally                                    │
│  Partitioned by geographic region                             │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Routing Algorithms

```python
class RoutingService:
    def find_route(self, origin, destination, mode='driving'):
        """Find optimal route between two points"""
        # Find nearest nodes to origin and destination
        start_node = self.find_nearest_node(origin)
        end_node = self.find_nearest_node(destination)

        # Use appropriate algorithm based on distance
        distance = haversine(origin, destination)

        if distance < 50:  # km
            # Short distance: Dijkstra or A*
            route = self.dijkstra(start_node, end_node)
        else:
            # Long distance: Contraction Hierarchies
            route = self.contraction_hierarchies(start_node, end_node)

        # Convert to human-readable directions
        directions = self.generate_directions(route)

        return {
            'route': route,
            'directions': directions,
            'distance_km': route.total_distance / 1000,
            'duration_minutes': route.total_duration / 60
        }

    def dijkstra(self, start, end):
        """Standard Dijkstra's algorithm"""
        distances = {start: 0}
        previous = {}
        pq = [(0, start)]
        visited = set()

        while pq:
            current_dist, current = heapq.heappop(pq)

            if current in visited:
                continue
            visited.add(current)

            if current == end:
                return self.reconstruct_path(previous, end)

            for neighbor, edge in self.get_neighbors(current):
                if neighbor in visited:
                    continue

                # Calculate edge weight (considering traffic)
                weight = self.calculate_weight(edge)
                distance = current_dist + weight

                if distance < distances.get(neighbor, float('inf')):
                    distances[neighbor] = distance
                    previous[neighbor] = (current, edge)
                    heapq.heappush(pq, (distance, neighbor))

        return None  # No route found
```

### Contraction Hierarchies (for long-distance routing)

```
┌────────────────────────────────────────────────────────────────┐
│              Contraction Hierarchies                           │
│                                                                │
│  Pre-processing:                                               │
│  1. Order nodes by importance (highways > local roads)        │
│  2. "Contract" less important nodes by adding shortcuts       │
│  3. Build hierarchical graph                                  │
│                                                                │
│  Original:   A ─── B ─── C ─── D                              │
│                                                                │
│  Contracted: A ───────────────── D   (shortcut)               │
│                                                                │
│  Query time:                                                   │
│  - Bidirectional search (from origin AND destination)        │
│  - Only traverse "upward" in hierarchy                       │
│  - Meet in the middle at high-importance node                │
│  - O(log n) vs O(n) for Dijkstra                             │
│                                                                │
│  Pre-processing: Hours                                        │
│  Query time: Milliseconds                                     │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 5. Real-Time Traffic

```
┌────────────────────────────────────────────────────────────────┐
│                   Traffic System                               │
│                                                                │
│  Data Sources:                                                 │
│  - GPS data from users (anonymized)                          │
│  - Traffic sensors                                            │
│  - Incidents/accidents reports                                │
│  - Historical patterns                                        │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Traffic Data Pipeline                                   │  │
│  │                                                          │  │
│  │  GPS Updates ──▶ Kafka ──▶ Spark Streaming ──▶ Redis    │  │
│  │                                                          │  │
│  │  1. Map-match GPS points to road segments               │  │
│  │  2. Calculate speed for each segment                    │  │
│  │  3. Compare to free-flow speed                          │  │
│  │  4. Store traffic factor per segment                    │  │
│  │                                                          │  │
│  │  Redis:                                                  │  │
│  │  Key: traffic:{segment_id}                              │  │
│  │  Value: {speed: 35, free_flow: 60, factor: 0.58}       │  │
│  │  TTL: 5 minutes                                          │  │
│  │                                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  Traffic levels:                                               │
│  - Green: > 80% of free-flow speed                           │
│  - Yellow: 50-80%                                             │
│  - Red: 25-50%                                                │
│  - Dark Red: < 25%                                            │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 6. Places Search

```python
class PlacesService:
    def __init__(self):
        self.elasticsearch = Elasticsearch()

    def search(self, query, location, radius_km=10):
        """Search for places"""
        results = self.elasticsearch.search(
            index='places',
            body={
                'query': {
                    'bool': {
                        'must': [
                            {'match': {'name': query}}
                        ],
                        'filter': [
                            {
                                'geo_distance': {
                                    'distance': f'{radius_km}km',
                                    'location': {
                                        'lat': location['lat'],
                                        'lon': location['lng']
                                    }
                                }
                            }
                        ]
                    }
                },
                'sort': [
                    {
                        '_geo_distance': {
                            'location': location,
                            'order': 'asc'
                        }
                    },
                    {'popularity': 'desc'}
                ]
            }
        )

        return results['hits']['hits']

    def geocode(self, address):
        """Convert address to coordinates"""
        # Parse address components
        parsed = self.parse_address(address)

        # Search in address database
        result = self.address_db.search(parsed)

        if result:
            return {
                'lat': result.lat,
                'lng': result.lng,
                'formatted_address': result.formatted
            }

        return None
```

## 7. Turn-by-Turn Navigation

```
┌────────────────────────────────────────────────────────────────┐
│               Turn-by-Turn Navigation                          │
│                                                                │
│  1. Calculate initial route                                   │
│  2. Stream updates to client                                  │
│  3. Re-route if user deviates                                │
│  4. Update ETA based on real-time traffic                    │
│                                                                │
│  Direction generation:                                         │
│  - Analyze angle between road segments                        │
│  - Detect maneuver type (turn, merge, exit)                  │
│  - Include road names and distances                          │
│                                                                │
│  {                                                             │
│    "maneuver": "turn-right",                                  │
│    "instruction": "Turn right onto Main Street",             │
│    "distance_m": 500,                                         │
│    "duration_s": 60,                                          │
│    "street_name": "Main Street"                               │
│  }                                                             │
│                                                                │
│  Client-side:                                                  │
│  - Compare GPS to route                                       │
│  - If off-route > 50m for > 10s: request reroute            │
│  - Voice guidance based on distance to next maneuver        │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 8. ETA Prediction

```python
class ETAService:
    def predict_eta(self, route, departure_time=None):
        """Predict ETA considering traffic"""
        if departure_time is None:
            departure_time = datetime.now()

        total_seconds = 0
        current_time = departure_time

        for segment in route.segments:
            # Get traffic for this segment at current time
            traffic = self.get_traffic(segment.id, current_time)

            # Calculate segment duration
            if traffic:
                speed = traffic['speed']
            else:
                # Use historical average for future times
                speed = self.get_historical_speed(segment.id, current_time)

            duration = segment.length_m / (speed * 1000 / 3600)  # seconds

            total_seconds += duration
            current_time += timedelta(seconds=duration)

        eta = departure_time + timedelta(seconds=total_seconds)

        return {
            'eta': eta,
            'duration_seconds': total_seconds,
            'traffic_conditions': self.summarize_traffic(route)
        }

    def get_historical_speed(self, segment_id, time):
        """Get typical speed for segment at given time"""
        day_of_week = time.weekday()
        hour = time.hour

        # Query historical data
        return self.historical_db.get_avg_speed(
            segment_id, day_of_week, hour
        )
```

## 9. Key Takeaways

1. **Tile-based maps** - Pre-rendered tiles at multiple zoom levels
2. **Contraction Hierarchies** - Fast routing for long distances
3. **Real-time traffic** - GPS data + stream processing
4. **Vector tiles** - Modern approach for dynamic styling
5. **ETA prediction** - Traffic-aware with historical patterns
6. **CDN for tiles** - Essential for latency

---

Next: [Dropbox](../04-dropbox/README.md)
