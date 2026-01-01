# Load Balancers

Load balancers distribute incoming traffic across multiple servers to improve availability, reliability, and performance.

## Table of Contents
- [What is a Load Balancer?](#what-is-a-load-balancer)
- [Types of Load Balancers](#types-of-load-balancers)
- [Load Balancing Algorithms](#load-balancing-algorithms)
- [Health Checks](#health-checks)
- [Advanced Features](#advanced-features)
- [Implementation Options](#implementation-options)
- [Key Takeaways](#key-takeaways)

## What is a Load Balancer?

A load balancer sits between clients and servers, distributing requests to prevent any single server from being overwhelmed.

```
                    ┌─────────────────────┐
                    │    Load Balancer    │
                    │                     │
    Clients ───────▶│   ┌───────────┐    │───────▶ Server 1
                    │   │ Algorithm │    │
                    │   └───────────┘    │───────▶ Server 2
                    │                     │
                    │                     │───────▶ Server 3
                    └─────────────────────┘
```

### Benefits

| Benefit | Description |
|---------|-------------|
| High availability | Route around failed servers |
| Scalability | Add/remove servers transparently |
| Performance | Distribute load evenly |
| Flexibility | Upgrade servers without downtime |
| Security | Single entry point, hide backend |

## Types of Load Balancers

### Layer 4 (Transport Layer)

Operates at TCP/UDP level. Makes routing decisions based on IP and port.

```
┌────────────────────────────────────────────────────────┐
│                    Layer 4 Load Balancer               │
│                                                        │
│  Client IP: 192.168.1.5:54321                         │
│  Server IP: 10.0.0.1:80                               │
│                                                        │
│  Decision based on:                                    │
│  - Source IP/Port                                      │
│  - Destination IP/Port                                 │
│  - Protocol (TCP/UDP)                                  │
│                                                        │
│  Cannot see:                                           │
│  - HTTP headers                                        │
│  - URL path                                            │
│  - Cookies                                             │
└────────────────────────────────────────────────────────┘
```

**Advantages:**
- Fast (no packet inspection)
- Protocol agnostic
- Simple configuration

**Disadvantages:**
- No content-based routing
- Limited health checks

### Layer 7 (Application Layer)

Operates at HTTP/HTTPS level. Can inspect content and make intelligent routing decisions.

```
┌────────────────────────────────────────────────────────┐
│                    Layer 7 Load Balancer               │
│                                                        │
│  Can inspect and route based on:                       │
│  - URL path (/api/*, /static/*)                       │
│  - HTTP method (GET, POST)                            │
│  - Headers (Host, Authorization)                      │
│  - Cookies (session affinity)                         │
│  - Query parameters                                    │
│  - Request body (limited)                              │
│                                                        │
│  Examples:                                             │
│  - /api/* → API servers                               │
│  - /images/* → Image servers                          │
│  - Host: api.example.com → API cluster                │
└────────────────────────────────────────────────────────┘
```

**Advantages:**
- Content-based routing
- SSL termination
- Request modification
- Advanced health checks

**Disadvantages:**
- Higher latency
- More CPU intensive
- More complex

### Comparison

| Feature | Layer 4 | Layer 7 |
|---------|---------|---------|
| Speed | Faster | Slower |
| CPU usage | Lower | Higher |
| Content inspection | No | Yes |
| SSL termination | Pass-through | Terminate |
| Routing flexibility | Limited | High |
| Use case | High throughput | Web apps |

## Load Balancing Algorithms

### Round Robin

Distribute requests sequentially.

```
Request 1 → Server 1
Request 2 → Server 2
Request 3 → Server 3
Request 4 → Server 1
Request 5 → Server 2
...
```

**Pros:** Simple, even distribution
**Cons:** Ignores server capacity and current load

### Weighted Round Robin

Assign weights based on server capacity.

```
Server 1 (weight 5): ■■■■■
Server 2 (weight 3): ■■■
Server 3 (weight 2): ■■

Request distribution:
1,2,3,4,5 → Server 1
6,7,8 → Server 2
9,10 → Server 3
11-15 → Server 1
...
```

**Pros:** Accounts for different server capacities
**Cons:** Weights must be manually configured

### Least Connections

Route to server with fewest active connections.

```
Server 1: 10 connections ←
Server 2: 25 connections
Server 3: 15 connections

New request → Server 1 (fewest connections)
```

**Pros:** Adapts to varying request durations
**Cons:** Doesn't account for server capacity

### Weighted Least Connections

Combine least connections with weights.

```
Effective load = connections / weight

Server 1: 10 connections, weight 5 → 10/5 = 2.0
Server 2: 8 connections, weight 2 → 8/2 = 4.0 ←

New request → Server 1 (lower effective load)
```

### IP Hash

Hash client IP to determine server (sticky sessions).

```
hash(client_ip) % num_servers = server_index

192.168.1.5 → hash → 12847 % 3 = 1 → Server 1
192.168.1.5 → always goes to Server 1
```

**Pros:** Session persistence without cookies
**Cons:** Uneven distribution if many clients share IP (NAT)

### Least Response Time

Route to server with fastest response time.

```
Server 1: avg 50ms ←
Server 2: avg 80ms
Server 3: avg 120ms

New request → Server 1 (fastest)
```

**Pros:** Optimizes for user experience
**Cons:** Requires active monitoring

### Random

Select a random server.

```
random() % num_servers = server_index
```

**Pros:** Simple, stateless
**Cons:** Can be uneven with small request counts

## Health Checks

### Types of Health Checks

```
┌────────────────────────────────────────────────────────┐
│                    Health Checks                       │
├────────────────────────────────────────────────────────┤
│                                                        │
│  TCP Check:                                            │
│  - Can connect to port?                               │
│  - Fast but limited                                    │
│                                                        │
│  HTTP Check:                                           │
│  - GET /health returns 200?                           │
│  - Can check response body                            │
│                                                        │
│  Custom Script:                                        │
│  - Run arbitrary health check                         │
│  - Check database, dependencies                       │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Health Check Parameters

```yaml
health_check:
  interval: 10s        # Check every 10 seconds
  timeout: 5s          # Wait 5 seconds for response
  unhealthy_threshold: 3   # 3 failures = unhealthy
  healthy_threshold: 2     # 2 successes = healthy
  path: /health
  expected_status: 200
```

### Graceful Degradation

```
Server states:
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Healthy   │────▶│   Draining  │────▶│  Unhealthy  │
│  (serving)  │     │ (finishing) │     │  (removed)  │
└─────────────┘     └─────────────┘     └─────────────┘
                     ▲
                     │
              Admin triggers
              maintenance
```

## Advanced Features

### SSL/TLS Termination

```
┌────────────────────────────────────────────────────────┐
│                                                        │
│  Client ──HTTPS──▶ Load Balancer ──HTTP──▶ Servers    │
│                         │                              │
│                    SSL terminated                      │
│                    here                                │
│                                                        │
│  Benefits:                                             │
│  - Offload CPU-intensive SSL from servers             │
│  - Centralized certificate management                 │
│  - Can inspect and modify traffic                     │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Session Persistence (Sticky Sessions)

```
Methods:
1. Cookie-based: LB sets cookie with server ID
2. IP-based: Hash client IP
3. Application-based: Use existing session cookie

┌────────────────────────────────────────────────────────┐
│  Cookie: SERVERID=server1                             │
│                                                        │
│  Client ──▶ LB reads cookie ──▶ Always to Server 1   │
└────────────────────────────────────────────────────────┘
```

### Connection Draining

```
When removing server:
1. Stop sending new connections
2. Wait for existing connections to complete (timeout)
3. Remove server from pool

Timeline:
──────────────────────────────────────────────────────▶
    │                      │                    │
 Drain start         Drain timeout         Server removed
    │                      │                    │
    │ Allow existing       │ Force close        │
    │ to complete          │ remaining          │
```

### Rate Limiting

```
At load balancer level:
- Per-IP limits
- Per-endpoint limits
- Global limits

X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1640000000
```

## Implementation Options

### Hardware Load Balancers

- F5 BIG-IP
- Citrix ADC (NetScaler)
- A10 Networks

**Pros:** High performance, enterprise features
**Cons:** Expensive, vendor lock-in

### Software Load Balancers

| Software | Type | Use Case |
|----------|------|----------|
| Nginx | L7 | Web traffic, reverse proxy |
| HAProxy | L4/L7 | High-performance TCP/HTTP |
| Traefik | L7 | Kubernetes, microservices |
| Envoy | L7 | Service mesh, modern apps |

### Cloud Load Balancers

| Provider | L4 | L7 |
|----------|-----|-----|
| AWS | NLB | ALB |
| GCP | Network LB | HTTP(S) LB |
| Azure | Load Balancer | Application Gateway |

### Example: HAProxy Configuration

```haproxy
frontend http_front
    bind *:80
    default_backend http_back

backend http_back
    balance roundrobin
    option httpchk GET /health
    server server1 10.0.0.1:8080 check
    server server2 10.0.0.2:8080 check
    server server3 10.0.0.3:8080 check
```

### Example: Nginx Configuration

```nginx
upstream backend {
    least_conn;
    server 10.0.0.1:8080 weight=3;
    server 10.0.0.2:8080 weight=2;
    server 10.0.0.3:8080 weight=1;
}

server {
    listen 80;
    location / {
        proxy_pass http://backend;
        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;
    }
}
```

## Key Takeaways

1. **L4 for speed**, L7 for intelligence
2. **Least connections** often best for varying request times
3. **Health checks** are critical - tune thresholds carefully
4. **SSL termination** at LB simplifies server management
5. **Sticky sessions** trade-off: simplicity vs. even distribution
6. **Cloud LBs** handle most use cases, use software for customization

## Practice Questions

1. When would you choose L4 over L7 load balancing?
2. Design a load balancing strategy for a service with long-running requests.
3. How would you handle a deployment without dropping requests?
4. What's the impact of sticky sessions on scaling?

## Further Reading

- [HAProxy Documentation](http://www.haproxy.org/documentation.html)
- [Nginx Load Balancing](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/)
- [AWS ELB Best Practices](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/load-balancer-getting-started.html)

---

Next: [Reverse Proxy](../02-reverse-proxy/README.md)
