# API Gateway

An API Gateway is a server that acts as a single entry point for all client requests to your backend services.

## Table of Contents
- [What is an API Gateway?](#what-is-an-api-gateway)
- [Core Features](#core-features)
- [Implementation](#implementation)
- [Popular API Gateways](#popular-api-gateways)
- [Key Takeaways](#key-takeaways)

## What is an API Gateway?

An API Gateway sits between clients and backend services, handling cross-cutting concerns like authentication, rate limiting, and routing.

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Mobile ──┐                              ┌─ User Service│
│           │     ┌─────────────────┐      │              │
│  Web ─────┼────▶│   API Gateway   │──────┼─ Order Service
│           │     │                 │      │              │
│  Partner ─┘     │ - Auth          │      ├─ Product Svc │
│                 │ - Rate Limit    │      │              │
│                 │ - Routing       │      └─ Payment Svc │
│                 │ - Transform     │                     │
│                 └─────────────────┘                     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Benefits

| Benefit | Description |
|---------|-------------|
| Single entry point | Simplified client experience |
| Cross-cutting concerns | Auth, rate limit, logging in one place |
| Protocol translation | REST to gRPC, HTTP to WebSocket |
| Request aggregation | Combine multiple service calls |
| Service abstraction | Hide backend complexity |

## Core Features

### 1. Authentication & Authorization

```
┌────────────────────────────────────────────────────────┐
│                    Authentication                      │
│                                                        │
│  Client ──▶ Gateway ──▶ Validate Token ──▶ Backend    │
│                │                                       │
│                ├── JWT validation                      │
│                ├── OAuth 2.0                          │
│                ├── API Key                            │
│                └── mTLS                               │
│                                                        │
│  Request with token:                                   │
│  Authorization: Bearer eyJhbGciOiJIUzI1NiIs...        │
│                                                        │
│  Gateway adds user context:                           │
│  X-User-Id: 12345                                     │
│  X-User-Role: admin                                   │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### 2. Rate Limiting

```
┌────────────────────────────────────────────────────────┐
│                    Rate Limiting                       │
│                                                        │
│  Per-user: 100 requests/minute                        │
│  Per-IP: 1000 requests/minute                         │
│  Per-endpoint: 50 requests/second                     │
│                                                        │
│  Response headers:                                     │
│  X-RateLimit-Limit: 100                               │
│  X-RateLimit-Remaining: 45                            │
│  X-RateLimit-Reset: 1640000000                        │
│                                                        │
│  When exceeded:                                        │
│  HTTP 429 Too Many Requests                           │
│  Retry-After: 30                                       │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### 3. Request Routing

```nginx
# Route by path
/api/users/*    → user-service
/api/orders/*   → order-service
/api/products/* → product-service

# Route by header
Host: api.example.com    → main API
Host: admin.example.com  → admin API

# Route by method
GET  /api/items   → read-service
POST /api/items   → write-service

# Canary/A-B testing
10% traffic → service-v2
90% traffic → service-v1
```

### 4. Request/Response Transformation

```
┌────────────────────────────────────────────────────────┐
│                   Transformation                       │
│                                                        │
│  Request:                                              │
│  - Add headers (correlation ID, auth context)        │
│  - Modify body (add defaults, validate)              │
│  - Change format (JSON → XML)                        │
│                                                        │
│  Response:                                             │
│  - Filter fields (remove internal data)              │
│  - Aggregate responses (from multiple services)      │
│  - Format errors (consistent error structure)        │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### 5. Load Balancing

```
┌────────────────────────────────────────────────────────┐
│                   Load Balancing                       │
│                                                        │
│  Gateway ──┬──▶ service-instance-1                    │
│            ├──▶ service-instance-2                    │
│            └──▶ service-instance-3                    │
│                                                        │
│  Algorithms:                                           │
│  - Round robin                                         │
│  - Least connections                                   │
│  - Random                                              │
│  - Weighted                                            │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### 6. Circuit Breaker

```
┌────────────────────────────────────────────────────────┐
│                   Circuit Breaker                      │
│                                                        │
│  Closed (normal):                                      │
│  Requests flow through                                │
│                                                        │
│  Open (failures exceeded threshold):                  │
│  Requests fail fast, don't hit backend               │
│                                                        │
│  Half-Open (after timeout):                           │
│  Allow some requests to test if backend recovered    │
│                                                        │
│  Config:                                               │
│  - Failure threshold: 50%                             │
│  - Window: 10 seconds                                 │
│  - Timeout: 30 seconds                                │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### 7. Caching

```
┌────────────────────────────────────────────────────────┐
│                      Caching                           │
│                                                        │
│  GET /api/products/123                                │
│                                                        │
│  Gateway checks cache:                                 │
│  - Hit: Return cached response                        │
│  - Miss: Forward to backend, cache response          │
│                                                        │
│  Cache key: method + path + query + headers           │
│  TTL: Based on Cache-Control header                   │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## Implementation

### Kong Configuration

```yaml
# Service
services:
  - name: user-service
    url: http://user-service:8080

# Route
routes:
  - name: user-routes
    service: user-service
    paths:
      - /api/users

# Plugin: Rate Limiting
plugins:
  - name: rate-limiting
    config:
      minute: 100
      policy: local

# Plugin: JWT Auth
  - name: jwt
    config:
      key_claim_name: kid
```

### AWS API Gateway (OpenAPI)

```yaml
openapi: 3.0.0
paths:
  /users/{id}:
    get:
      x-amazon-apigateway-integration:
        type: http_proxy
        uri: http://user-service.internal/users/{id}
        httpMethod: GET
      security:
        - api_key: []
      x-amazon-apigateway-request-validator: all
```

### Nginx as API Gateway

```nginx
upstream user_service {
    server user-service-1:8080;
    server user-service-2:8080;
}

server {
    listen 443 ssl;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

    location /api/users {
        limit_req zone=api burst=20;

        # Auth check
        auth_request /auth;

        proxy_pass http://user_service;
        proxy_set_header X-User-Id $http_x_user_id;
    }

    location = /auth {
        internal;
        proxy_pass http://auth-service/validate;
    }
}
```

## Popular API Gateways

| Gateway | Type | Best For |
|---------|------|----------|
| Kong | OSS/Enterprise | General purpose, plugins |
| AWS API Gateway | Managed | AWS integration |
| Apigee | Managed | Enterprise, analytics |
| Nginx | OSS | High performance |
| Traefik | OSS | Kubernetes, auto-discovery |
| Envoy | OSS | Service mesh, gRPC |
| Express Gateway | OSS | Node.js ecosystem |

### Comparison

```
┌────────────────────────────────────────────────────────┐
│              Gateway Comparison                        │
├────────────────────────────────────────────────────────┤
│                                                        │
│  Self-hosted (Kong, Nginx, Envoy):                    │
│  + Full control                                        │
│  + No vendor lock-in                                  │
│  - Operational overhead                               │
│                                                        │
│  Managed (AWS, Apigee, Azure):                        │
│  + No operations                                       │
│  + Built-in monitoring                                │
│  - Vendor lock-in                                     │
│  - Cost at scale                                       │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## Key Takeaways

1. **Single entry point** simplifies client integration
2. **Centralize cross-cutting concerns** - auth, rate limit, logging
3. **Don't put business logic** in the gateway
4. **Plan for high availability** - gateway is a critical path
5. **Monitor gateway performance** - it's on every request path
6. **Consider latency overhead** - each feature adds latency

## Practice Questions

1. Design an API gateway for a microservices architecture.
2. How would you handle authentication for internal vs external APIs?
3. What's the difference between an API gateway and a load balancer?
4. How do you avoid the gateway becoming a bottleneck?

## Further Reading

- [Kong Documentation](https://docs.konghq.com/)
- [AWS API Gateway Guide](https://docs.aws.amazon.com/apigateway/)
- [Building Microservices - API Gateway Pattern](https://microservices.io/patterns/apigateway.html)

---

Next: [Rate Limiter](../10-rate-limiter/README.md)
