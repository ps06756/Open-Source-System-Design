# High-Level Design

Drawing the system architecture and defining key interfaces.

## Goals of High-Level Design

- Show the main components
- Illustrate data flow
- Define APIs
- Sketch database schema
- Identify potential bottlenecks

## Standard Components

### Tier 1: Entry Points
```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  [Client]    → [DNS]                                       │
│              → [CDN] (static content)                      │
│              → [Load Balancer] (dynamic requests)          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Tier 2: Application Layer
```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  [API Gateway] → [Service A] (user service)                │
│               → [Service B] (order service)                │
│               → [Service C] (payment service)              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Tier 3: Data Layer
```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  [Cache] ←→ [Primary DB] → [Replica DB]                   │
│  [Queue] → [Workers]                                       │
│  [Blob Storage]                                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Drawing the Architecture

### Start Simple

```
┌─────────────────────────────────────────────────────────────┐
│  Version 1: Basic Architecture                              │
│                                                             │
│  [Client] → [Load Balancer] → [Web Servers] → [Database]  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Add Complexity as Needed

```
┌─────────────────────────────────────────────────────────────┐
│  Version 2: With Caching and CDN                           │
│                                                             │
│  [Client] → [CDN] (static)                                 │
│          → [Load Balancer]                                 │
│              → [Web Servers] → [Cache] → [Database]       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Final Version

```
┌─────────────────────────────────────────────────────────────┐
│  Version 3: Full Architecture                               │
│                                                             │
│  [Client] ─┬─ [CDN]                                        │
│            │                                                │
│            └─ [Load Balancer]                              │
│                    │                                        │
│            ┌───────┴───────┐                               │
│            ▼               ▼                               │
│      [API Server]    [API Server]                          │
│            │               │                               │
│            └───────┬───────┘                               │
│                    │                                        │
│         ┌─────────┴─────────┐                              │
│         ▼                   ▼                              │
│      [Cache]          [Message Queue]                      │
│         │                   │                              │
│         ▼                   ▼                              │
│      [Database]        [Workers]                           │
│         │                                                   │
│         ▼                                                   │
│      [Blob Storage]                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## API Design

### RESTful Conventions

```
Resource: /api/v1/{resource}

GET    /users           List users
GET    /users/{id}      Get specific user
POST   /users           Create user
PUT    /users/{id}      Update user
DELETE /users/{id}      Delete user
```

### Example APIs

**Social Media:**
```
POST /api/posts          Create post
GET  /api/feed           Get user's feed
POST /api/posts/{id}/like   Like a post
POST /api/users/{id}/follow  Follow user
```

**E-Commerce:**
```
GET  /api/products       List products
POST /api/cart/items     Add to cart
POST /api/orders         Create order
GET  /api/orders/{id}    Get order status
```

### Pagination

Always include pagination for list endpoints:
```
GET /api/posts?cursor={last_id}&limit=20

Response:
{
  "data": [...],
  "next_cursor": "abc123",
  "has_more": true
}
```

## Database Schema

### Keep It Simple

Focus on the core entities:

```sql
-- User
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    username VARCHAR(50) UNIQUE,
    email VARCHAR(100) UNIQUE,
    created_at TIMESTAMP
);

-- Core entity (varies by system)
CREATE TABLE posts (
    id BIGINT PRIMARY KEY,
    user_id BIGINT REFERENCES users(id),
    content TEXT,
    created_at TIMESTAMP,
    INDEX idx_user_created (user_id, created_at)
);

-- Relationship
CREATE TABLE follows (
    follower_id BIGINT,
    followee_id BIGINT,
    created_at TIMESTAMP,
    PRIMARY KEY (follower_id, followee_id)
);
```

### Mention Key Decisions

- **SQL vs NoSQL**: "I'm using PostgreSQL because we need transactions for..."
- **Indexing**: "We'll index on user_id and created_at for timeline queries"
- **Sharding**: "We'll shard by user_id to distribute load"

## Data Flow Diagrams

### Write Path
```
User creates post:
1. Client → Load Balancer → API Server
2. API Server validates request
3. API Server → Database (write post)
4. API Server → Cache (invalidate)
5. API Server → Queue (fan-out job)
6. Return success to client
```

### Read Path
```
User views feed:
1. Client → Load Balancer → API Server
2. API Server → Cache (check for cached feed)
3. If cache miss: API Server → Database
4. API Server → Cache (store result)
5. Return feed to client
```

## Tips

### 1. Draw Before You Talk
Put something on the whiteboard, then explain it.

### 2. Use Boxes and Arrows
Keep it clean and organized.

### 3. Label Everything
Don't assume the interviewer knows what each box is.

### 4. Show Data Flow
Arrows should indicate direction of data movement.

### 5. Start Broad, Then Zoom In
Don't get stuck on one component too early.

## Common Patterns

### Read-Heavy Systems
```
Client → CDN → Cache → Database (read replica)
```

### Write-Heavy Systems
```
Client → Queue → Workers → Database (sharded)
```

### Real-Time Systems
```
Client ←→ WebSocket Server → Pub/Sub → Other Servers
```

---

Next: [Deep Dive](../05-deep-dive/README.md)
