# Requirements Gathering

The most important 5 minutes of your interview. Getting requirements right sets you up for success.

## Why This Matters

- Shows you think before coding
- Prevents wasted time on wrong features
- Demonstrates product sense
- Helps scope the problem appropriately

## Types of Requirements

### Functional Requirements
**What** the system should do.

```
Examples:
- Users can post messages
- Users can search for content
- System sends notifications
- Users can upload files
```

### Non-Functional Requirements
**How** the system should perform.

```
Categories:
- Scalability: How many users/requests?
- Availability: What uptime is required?
- Latency: How fast must responses be?
- Consistency: Strong or eventual?
- Durability: Can we lose data?
```

### Out of Scope
What you're NOT building. Important to clarify.

```
Examples:
- "Let's not worry about the payment system for now"
- "We can assume authentication is handled"
- "Mobile app design is out of scope"
```

## Questions to Ask

### About Users and Scale

```
"Who are the users of this system?"
"How many daily active users should we support?"
"What's the expected growth rate?"
"Is this a global system or regional?"
```

### About Features

```
"What are the must-have features vs nice-to-have?"
"Are there different user types with different capabilities?"
"What's the expected user behavior? (read-heavy, write-heavy)"
"Should we support real-time features?"
```

### About Data

```
"How long do we need to retain data?"
"Is data deletion required (GDPR compliance)?"
"What's the expected data size per item?"
"How important is data consistency?"
```

### About Performance

```
"What latency is acceptable?"
"Is there a peak traffic pattern? (time of day, events)"
"What's the availability requirement? (99.9%, 99.99%)"
"Is eventual consistency acceptable?"
```

## Example: Design a URL Shortener

### Bad Approach
Jumping straight to: "I'll use a hash function to generate short URLs and store them in a database..."

### Good Approach

**You:** "Let me make sure I understand the requirements. What are the core features?"

**Interviewer:** "Users should be able to shorten URLs and get redirected."

**You:** "Got it. A few clarifying questions:
1. Should shortened URLs expire?
2. Do we need analytics (click counts)?
3. Should users be able to customize the short URL?
4. What's the expected scale - how many URLs per day?"

**Interviewer:** "Let's say URLs don't expire, we want basic analytics, no custom URLs, and handle 100M new URLs per month."

**You:** "Great. So to summarize:
- Functional: Create short URL, redirect to long URL, track clicks
- Non-functional: 100M URLs/month, high availability, low redirect latency
- Out of scope: Custom URLs, user accounts, expiration

Does that sound right?"

## Summarizing Requirements

Always restate what you heard:

```
"Let me summarize what we're building:

Functional Requirements:
1. [Feature 1]
2. [Feature 2]
3. [Feature 3]

Non-Functional Requirements:
- Scale: X users, Y requests/sec
- Latency: < Z ms for main operations
- Availability: 99.9%
- Consistency: [Strong/Eventual]

Out of Scope:
- [Thing we're not building]

Does this align with what you had in mind?"
```

## Common Mistakes

### 1. Not Asking About Scale
Scale changes everything. A system for 1,000 users is very different from one for 1 billion.

### 2. Assuming Features
Don't assume the interviewer wants every feature. Ask what's important.

### 3. Ignoring Non-Functional Requirements
These often contain the hard problems. Consistency and availability trade-offs matter.

### 4. Taking Too Long
5-7 minutes is enough. Don't turn it into a 20-minute discovery session.

### 5. Not Writing It Down
Write the requirements on the whiteboard. Reference them throughout.

## Quick Reference: Common Requirements

### Social Media
- Users, posts, follows, feed
- Scale: 100M-1B users
- Read-heavy (100:1 read/write)
- Feed latency critical

### E-Commerce
- Products, orders, payments, inventory
- Strong consistency for orders
- Handle flash sales (spike traffic)
- Transaction integrity

### Messaging
- Real-time delivery
- Message ordering
- Presence (online status)
- End-to-end encryption

### Storage
- Upload/download files
- Sync across devices
- Versioning
- Sharing

---

Next: [Capacity Estimation](../03-estimation/README.md)
