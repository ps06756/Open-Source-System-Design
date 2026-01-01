# The System Design Interview Framework

A structured approach to tackle any system design question.

## The 4-Step Framework

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  Step 1: REQUIREMENTS (5-7 min)                               │
│  ├── Functional requirements                                  │
│  ├── Non-functional requirements                              │
│  └── Constraints and assumptions                              │
│                                                                │
│  Step 2: ESTIMATION (3-5 min)                                 │
│  ├── Traffic estimates                                        │
│  ├── Storage estimates                                        │
│  └── Bandwidth estimates                                      │
│                                                                │
│  Step 3: HIGH-LEVEL DESIGN (10-15 min)                       │
│  ├── API design                                               │
│  ├── Database schema                                          │
│  └── System architecture diagram                              │
│                                                                │
│  Step 4: DETAILED DESIGN (15-20 min)                         │
│  ├── Deep dive into key components                           │
│  ├── Handle scale and edge cases                             │
│  └── Discuss trade-offs and alternatives                     │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## Step 1: Requirements

### Questions to Ask

**Functional (What should the system do?)**
- What are the core features?
- Who are the users?
- What actions can users perform?
- Are there different user types (admin, regular)?

**Non-Functional (How should it perform?)**
- What's the expected scale? (users, requests)
- What's the latency requirement?
- Is consistency or availability more important?
- What's the data retention policy?

**Constraints**
- Any technology constraints?
- Budget or infrastructure limitations?
- Regulatory requirements (GDPR, etc.)?

### Example: Design Twitter

```
Functional:
- Post tweets (280 chars)
- Follow/unfollow users
- View home timeline
- Search tweets

Non-Functional:
- 500M DAU
- Timeline load < 200ms
- High availability
- Eventual consistency OK

Constraints:
- Support global users
- Handle celebrity accounts (millions of followers)
```

## Step 2: Estimation

### Quick Math Formula

```
Daily Active Users (DAU): Given or estimated
Requests/day = DAU × actions/user/day
Requests/sec = Requests/day ÷ 86,400

Storage = Items × Size × Retention period
Bandwidth = Requests/sec × Request size
```

### Example: Twitter Estimation

```
DAU: 500M
Tweets/day: 500M × 0.5 = 250M tweets
Tweets/sec: 250M ÷ 86,400 ≈ 3,000/sec

Reads (timeline): 500M × 20 views = 10B/day
Reads/sec: 10B ÷ 86,400 ≈ 115,000/sec

Storage (5 years):
Tweets: 250M × 365 × 5 = 456B tweets
Size: 456B × 280 bytes = 127 TB (text only)
```

## Step 3: High-Level Design

### Start with Components

1. **Client** - web, mobile apps
2. **Load Balancer** - distribute traffic
3. **API Servers** - handle requests
4. **Database** - store data
5. **Cache** - speed up reads
6. **CDN** - serve static content

### Draw the Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  [Clients] → [Load Balancer] → [API Servers] → [Cache]       │
│                                      ↓                        │
│                              [Database (Primary)]             │
│                                      ↓                        │
│                              [Database (Replica)]             │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Define APIs

```
POST /api/tweets
  Request: { content: string }
  Response: { tweet_id: string }

GET /api/timeline?cursor={cursor}
  Response: { tweets: [...], next_cursor: string }

POST /api/follow/{user_id}
  Response: { success: boolean }
```

### Design Database Schema

```sql
CREATE TABLE tweets (
    tweet_id BIGINT PRIMARY KEY,
    user_id BIGINT,
    content VARCHAR(280),
    created_at TIMESTAMP
);

CREATE TABLE follows (
    follower_id BIGINT,
    followee_id BIGINT,
    PRIMARY KEY (follower_id, followee_id)
);
```

## Step 4: Detailed Design

### Pick Components to Deep Dive

Choose 2-3 components that are:
- Core to the system
- Technically interesting
- Potentially challenging at scale

### Example Deep Dives for Twitter

1. **Timeline Generation**
   - Fan-out on write vs read
   - Handling celebrities
   - Caching strategy

2. **Tweet Storage**
   - Sharding strategy
   - ID generation (Snowflake)
   - Replication

3. **Search**
   - Elasticsearch indexing
   - Real-time vs batch
   - Ranking

### Discuss Trade-offs

For each decision, explain:
- Why you chose it
- What alternatives exist
- What you're giving up

## Sample Timeline

```
0:00 - 0:05  Requirements clarification
0:05 - 0:08  Capacity estimation
0:08 - 0:20  High-level design + API + schema
0:20 - 0:40  Detailed design (2-3 components)
0:40 - 0:45  Wrap up, questions, trade-offs
```

## Tips

1. **Write as you talk** - use the whiteboard actively
2. **Number your diagrams** - easy to reference later
3. **Check in with interviewer** - "Should I go deeper here?"
4. **Be flexible** - follow the interviewer's interests

---

Next: [Requirements Gathering](../02-requirements/README.md)
