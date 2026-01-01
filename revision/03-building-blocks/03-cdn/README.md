# Content Delivery Network (CDN)

A CDN is a geographically distributed network of servers that delivers content to users from the nearest location.

## Table of Contents
- [What is a CDN?](#what-is-a-cdn)
- [How CDNs Work](#how-cdns-work)
- [Push vs Pull CDN](#push-vs-pull-cdn)
- [Caching Strategies](#caching-strategies)
- [CDN Features](#cdn-features)
- [CDN Providers](#cdn-providers)
- [Key Takeaways](#key-takeaways)

## What is a CDN?

A CDN caches content at edge locations worldwide, serving content from servers geographically close to users.

```
Without CDN:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  User (Tokyo) ────────────────────▶ Origin (US)        │
│                    150ms+ latency                       │
│                                                         │
└─────────────────────────────────────────────────────────┘

With CDN:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  User (Tokyo) ──────▶ CDN Edge (Tokyo) ◀──── Origin    │
│                  10ms         cached                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Benefits

| Benefit | Description |
|---------|-------------|
| Lower latency | Content served from nearby edge |
| Reduced bandwidth | Origin only serves cache misses |
| High availability | Multiple edge locations |
| DDoS protection | Distributed absorbs attacks |
| Scalability | Handle traffic spikes |

## How CDNs Work

### Request Flow

```
User Request: https://cdn.example.com/image.jpg

1. DNS Resolution
   cdn.example.com → CDN DNS → Nearest edge IP

2. Edge Server Check
   ┌─────────────────────────────────────────────────┐
   │  Cache Hit?                                     │
   │                                                 │
   │  Yes → Return cached content                   │
   │  No  → Fetch from origin                       │
   │        Cache response                           │
   │        Return to user                           │
   └─────────────────────────────────────────────────┘

3. Response Headers
   Cache-Control: max-age=86400
   X-Cache: HIT (or MISS)
```

### CDN Architecture

```
                    ┌─────────────────────────────────┐
                    │        Origin Server            │
                    │      (your infrastructure)      │
                    └───────────────┬─────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │         Shield / Mid-tier      │
                    │     (optional caching layer)   │
                    └───────────────┬───────────────┘
                                    │
         ┌──────────────────────────┼──────────────────────────┐
         │                          │                          │
    ┌─────────┐               ┌─────────┐               ┌─────────┐
    │  Edge   │               │  Edge   │               │  Edge   │
    │  (US)   │               │  (EU)   │               │ (Asia)  │
    └─────────┘               └─────────┘               └─────────┘
         ▲                         ▲                         ▲
         │                         │                         │
    US Users                  EU Users                 Asia Users
```

## Push vs Pull CDN

### Pull CDN

Content is fetched from origin on demand.

```
┌────────────────────────────────────────────────────────┐
│                      Pull CDN                          │
│                                                        │
│  1. User requests image.jpg                           │
│  2. Edge doesn't have it → fetch from origin          │
│  3. Cache at edge                                      │
│  4. Serve to user                                      │
│  5. Future requests served from cache                 │
│                                                        │
│  Good for: Dynamic content, large catalogs            │
│  Bad for: Cold cache (first request slow)             │
└────────────────────────────────────────────────────────┘
```

### Push CDN

Content is pre-uploaded to CDN.

```
┌────────────────────────────────────────────────────────┐
│                      Push CDN                          │
│                                                        │
│  1. Upload content to CDN storage                     │
│  2. CDN distributes to edges                          │
│  3. User requests → served immediately                │
│                                                        │
│  Good for: Static content, predictable access         │
│  Bad for: Large/changing catalogs                     │
└────────────────────────────────────────────────────────┘
```

### Comparison

| Aspect | Pull CDN | Push CDN |
|--------|----------|----------|
| Setup | Easier | More work |
| First request | Slow (cache miss) | Fast (pre-cached) |
| Storage cost | Pay for cache | Pay for all content |
| Best for | Large/dynamic catalogs | Small/static content |
| Cache invalidation | TTL-based | Manual upload |

## Caching Strategies

### Cache-Control Headers

```http
# Cache for 1 day, allow CDN caching
Cache-Control: public, max-age=86400

# Cache for 1 hour, must revalidate after
Cache-Control: public, max-age=3600, must-revalidate

# Don't cache
Cache-Control: no-store

# Private (user-specific), don't cache on CDN
Cache-Control: private, max-age=3600

# Stale-while-revalidate
Cache-Control: public, max-age=3600, stale-while-revalidate=86400
```

### Cache Key

What makes a cached response unique?

```
Default cache key: URL + Host

Custom cache key:
- URL + Query params
- URL + Cookies
- URL + Headers
- URL + Device type

Example:
https://example.com/api/users?page=1
https://example.com/api/users?page=2
→ Different cache entries
```

### Cache Invalidation

```
Methods:

1. TTL Expiration
   Content expires after max-age
   Simple but slow for updates

2. Purge
   DELETE https://cdn.example.com/purge/image.jpg
   Immediate but manual

3. Versioned URLs
   /image.jpg?v=123 → /image.jpg?v=124
   Never need to invalidate

4. Surrogate Keys / Tags
   Tag content, purge by tag
   "Purge all product-123 content"
```

## CDN Features

### Dynamic Content Acceleration

Optimize non-cacheable content:

```
┌────────────────────────────────────────────────────────┐
│            Dynamic Content Acceleration                │
│                                                        │
│  - Persistent connections to origin                   │
│  - Route optimization (best path)                     │
│  - Protocol optimization (HTTP/2, QUIC)              │
│  - Compression                                         │
│  - TCP optimizations                                   │
│                                                        │
│  User → Edge (10ms) → Optimized route → Origin        │
│          vs                                            │
│  User → Direct to Origin (150ms)                      │
└────────────────────────────────────────────────────────┘
```

### Edge Computing

Run code at CDN edge:

```javascript
// Cloudflare Worker example
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  // A/B testing at edge
  const bucket = Math.random() < 0.5 ? 'A' : 'B'

  // Modify request
  const url = new URL(request.url)
  url.pathname = `/${bucket}${url.pathname}`

  return fetch(url)
}
```

### Security Features

```
┌────────────────────────────────────────────────────────┐
│                 CDN Security                           │
├────────────────────────────────────────────────────────┤
│                                                        │
│  DDoS Protection                                       │
│  - Absorb volumetric attacks                          │
│  - Rate limiting                                       │
│  - Bot detection                                       │
│                                                        │
│  WAF (Web Application Firewall)                       │
│  - Block SQL injection                                │
│  - Block XSS                                          │
│  - Custom rules                                        │
│                                                        │
│  SSL/TLS                                               │
│  - Free certificates                                   │
│  - Automatic renewal                                   │
│  - Modern protocols                                    │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Image Optimization

```
Original: /image.jpg (2 MB)

CDN transforms:
/image.jpg?width=300     → 50 KB
/image.jpg?format=webp   → 100 KB
/image.jpg?quality=80    → 500 KB

Automatic based on device:
Mobile → smaller, WebP
Desktop → larger, original format
```

## CDN Providers

### Comparison

| Provider | Strengths | Use Case |
|----------|-----------|----------|
| CloudFlare | Free tier, security | General purpose |
| AWS CloudFront | AWS integration | AWS users |
| Akamai | Enterprise, global | Large enterprises |
| Fastly | Real-time purge, VCL | High customization |
| Google Cloud CDN | GCP integration | GCP users |
| Azure CDN | Azure integration | Azure users |

### Pricing Models

```
┌────────────────────────────────────────────────────────┐
│                 CDN Pricing                            │
├────────────────────────────────────────────────────────┤
│                                                        │
│  Bandwidth:      $0.01-0.10 per GB                    │
│  Requests:       $0.01 per 10,000 requests            │
│  Invalidations:  Often free or $0.005 each            │
│  Edge compute:   Per request or CPU time              │
│                                                        │
│  Example monthly cost (1 PB egress):                  │
│  - CloudFlare Pro: $200/month (flat)                  │
│  - CloudFront: ~$50,000/month                         │
│  - Fastly: ~$40,000/month                             │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## Configuration Example

### CloudFront (AWS)

```yaml
# CloudFormation
Resources:
  CDN:
    Type: AWS::CloudFront::Distribution
    Properties:
      DistributionConfig:
        Origins:
          - DomainName: origin.example.com
            Id: origin
            CustomOriginConfig:
              OriginProtocolPolicy: https-only
        DefaultCacheBehavior:
          TargetOriginId: origin
          ViewerProtocolPolicy: redirect-to-https
          CachePolicyId: !Ref CachePolicy
          Compress: true
        Enabled: true

  CachePolicy:
    Type: AWS::CloudFront::CachePolicy
    Properties:
      CachePolicyConfig:
        Name: OptimizedCaching
        DefaultTTL: 86400
        MaxTTL: 31536000
        MinTTL: 1
```

### Nginx (as CDN origin)

```nginx
server {
    location /static/ {
        # Cache headers for CDN
        add_header Cache-Control "public, max-age=31536000";
        add_header Vary "Accept-Encoding";

        # Enable gzip for CDN to cache compressed
        gzip on;
        gzip_types text/css application/javascript;
    }

    location /api/ {
        # Don't cache API responses at CDN
        add_header Cache-Control "private, no-store";
    }
}
```

## Key Takeaways

1. **CDN dramatically reduces latency** for global users
2. **Pull CDN** for most use cases, push for static assets
3. **Proper Cache-Control headers** are essential
4. **Version URLs** to avoid cache invalidation complexity
5. **Edge computing** for request-level logic
6. **Consider costs** - bandwidth can get expensive at scale

## Practice Questions

1. How would you handle user-specific content with a CDN?
2. Design a CDN caching strategy for an e-commerce site.
3. What's the impact of cache hit ratio on origin load?
4. How do you debug CDN caching issues?

## Further Reading

- [HTTP Caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)
- [CloudFlare Learning Center](https://www.cloudflare.com/learning/)
- [AWS CloudFront Developer Guide](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/)

---

Next: [Databases](../04-databases/README.md)
