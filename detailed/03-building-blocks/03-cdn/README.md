# Content Delivery Network (CDN)

A CDN is a geographically distributed network of servers that delivers content to users from the nearest location, reducing latency and improving performance.

## Table of Contents
1. [How CDN Works](#how-cdn-works)
2. [Push vs Pull CDN](#push-vs-pull-cdn)
3. [Cache Invalidation](#cache-invalidation)
4. [CDN Selection](#cdn-selection)
5. [Interview Questions](#interview-questions)

---

## How CDN Works

```
┌─────────────────────────────────────────────────────────────┐
│                     How CDN Works                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Without CDN:                                                │
│  User in Tokyo → Origin in New York = 200ms latency         │
│                                                              │
│  With CDN:                                                   │
│  User in Tokyo → Edge in Tokyo = 20ms latency              │
│                                                              │
│                    ┌───────────────┐                        │
│                    │    Origin     │                        │
│                    │   (New York)  │                        │
│                    └───────┬───────┘                        │
│                            │                                 │
│         ┌──────────────────┼──────────────────┐             │
│         │                  │                  │             │
│         ▼                  ▼                  ▼             │
│    ┌─────────┐        ┌─────────┐        ┌─────────┐       │
│    │  Edge   │        │  Edge   │        │  Edge   │       │
│    │ Tokyo   │        │ London  │        │ Sydney  │       │
│    └────┬────┘        └────┬────┘        └────┬────┘       │
│         │                  │                  │             │
│         ▼                  ▼                  ▼             │
│      Users              Users              Users            │
│      (Asia)            (Europe)          (Australia)        │
│                                                              │
│  Flow:                                                       │
│  1. User requests image.jpg                                 │
│  2. DNS routes to nearest edge server                       │
│  3. If cached → return immediately (cache hit)             │
│  4. If not cached → fetch from origin, cache, return       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Push vs Pull CDN

```
┌─────────────────────────────────────────────────────────────┐
│                   Push vs Pull CDN                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Pull CDN (Lazy Loading):                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 1. User requests content                               │ │
│  │ 2. Edge checks cache - MISS                           │ │
│  │ 3. Edge fetches from origin                           │ │
│  │ 4. Edge caches and returns to user                    │ │
│  │ 5. Next request - cache HIT                           │ │
│  └────────────────────────────────────────────────────────┘ │
│  ✓ No upfront work                                          │
│  ✓ Only caches what's needed                               │
│  ✗ First request is slow (cache miss)                      │
│  Best for: Most websites, dynamic content                   │
│                                                              │
│  Push CDN (Proactive):                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 1. You upload content to CDN                          │ │
│  │ 2. CDN distributes to all edge servers                │ │
│  │ 3. User requests - always cache HIT                   │ │
│  └────────────────────────────────────────────────────────┘ │
│  ✓ No cache misses                                          │
│  ✓ Predictable performance                                  │
│  ✗ Must manage uploads                                      │
│  ✗ Storage costs for all edges                             │
│  Best for: Large files, known content (videos)             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Cache Invalidation

```
┌─────────────────────────────────────────────────────────────┐
│                  Cache Invalidation                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Time-based (TTL)                                        │
│     Cache-Control: max-age=3600                             │
│     Content expires after 1 hour                            │
│     Simple but may serve stale content                      │
│                                                              │
│  2. Versioned URLs                                           │
│     /static/app.v2.js                                       │
│     /static/app.js?v=abc123                                 │
│     Change URL = new cache entry                            │
│     Instant updates, old versions still cached              │
│                                                              │
│  3. Purge API                                                │
│     POST /purge {"urls": ["/image.jpg"]}                    │
│     Immediately remove from all edges                       │
│     Takes time to propagate globally                        │
│                                                              │
│  4. Soft Purge / Stale-While-Revalidate                    │
│     Mark as stale, serve while fetching new                │
│     No downtime during updates                              │
│                                                              │
│  Best Practice:                                              │
│  • Static assets: Versioned URLs + long TTL                │
│  • HTML pages: Short TTL or no-cache                       │
│  • API responses: Vary by auth, short TTL                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## CDN Selection

| CDN | Strengths | Best For |
|-----|-----------|----------|
| **Cloudflare** | Security, DDoS, free tier | Most websites |
| **AWS CloudFront** | AWS integration | AWS-based apps |
| **Akamai** | Enterprise, largest network | Large enterprises |
| **Fastly** | Real-time purging, edge compute | Dynamic content |
| **Google Cloud CDN** | GCP integration | GCP-based apps |

---

## Interview Questions

### Basic
1. What is a CDN and why is it used?
2. Explain the difference between push and pull CDN.

### Intermediate
3. How would you handle cache invalidation for a news website?
4. What content should and shouldn't be cached on a CDN?

### Advanced
5. Design a CDN strategy for a video streaming platform.

---

[← Previous: Reverse Proxy](../02-reverse-proxy/README.md) | [Next: Databases →](../04-databases/README.md)
