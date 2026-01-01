# Load Balancers

Load balancers distribute incoming traffic across multiple servers to ensure no single server is overwhelmed. They're essential for scalability and high availability.

## Table of Contents
1. [What is a Load Balancer?](#what-is-a-load-balancer)
2. [L4 vs L7 Load Balancing](#l4-vs-l7-load-balancing)
3. [Load Balancing Algorithms](#load-balancing-algorithms)
4. [Health Checks](#health-checks)
5. [High Availability](#high-availability)
6. [Interview Questions](#interview-questions)

---

## What is a Load Balancer?

```
┌─────────────────────────────────────────────────────────────┐
│                    Load Balancer                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Without Load Balancer:                                      │
│  ┌─────────┐                    ┌─────────┐                 │
│  │ Clients │──── All traffic ──▶│ Server  │                 │
│  └─────────┘                    └─────────┘                 │
│                                  (overloaded!)               │
│                                                              │
│  With Load Balancer:                                         │
│  ┌─────────┐     ┌──────────────┐     ┌─────────┐          │
│  │         │     │              │────▶│Server 1 │          │
│  │ Clients │────▶│Load Balancer │────▶│Server 2 │          │
│  │         │     │              │────▶│Server 3 │          │
│  └─────────┘     └──────────────┘     └─────────┘          │
│                                                              │
│  Benefits:                                                   │
│  • Distributes load evenly                                  │
│  • Removes failed servers automatically                     │
│  • Enables horizontal scaling                               │
│  • Provides single entry point                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## L4 vs L7 Load Balancing

```
┌─────────────────────────────────────────────────────────────┐
│              Layer 4 vs Layer 7 Load Balancing               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Layer 4 (Transport Layer):                                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ • Works at TCP/UDP level                               │ │
│  │ • Sees: IP addresses, ports                            │ │
│  │ • Fast (no content inspection)                         │ │
│  │ • Simple (just forwards packets)                       │ │
│  │                                                         │ │
│  │ Decision based on:                                     │ │
│  │ Source IP: 192.168.1.100:54321                        │ │
│  │ Dest IP:   10.0.0.1:80                                │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Layer 7 (Application Layer):                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ • Works at HTTP/HTTPS level                            │ │
│  │ • Sees: URLs, headers, cookies, content                │ │
│  │ • Slower (must parse HTTP)                             │ │
│  │ • Smart routing capabilities                           │ │
│  │                                                         │ │
│  │ Decision based on:                                     │ │
│  │ URL: /api/users                                        │ │
│  │ Header: Authorization: Bearer xxx                      │ │
│  │ Cookie: session=abc123                                 │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────┬────────────────────────────────┐   │
│  │     Feature        │   L4          │      L7        │   │
│  ├────────────────────┼───────────────┼────────────────┤   │
│  │ Speed              │ Faster        │ Slower         │   │
│  │ SSL Termination    │ No            │ Yes            │   │
│  │ Content Routing    │ No            │ Yes            │   │
│  │ WebSocket          │ Pass-through  │ Full support   │   │
│  │ Caching            │ No            │ Yes            │   │
│  │ Compression        │ No            │ Yes            │   │
│  └────────────────────┴───────────────┴────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### L7 Content-Based Routing

```
┌─────────────────────────────────────────────────────────────┐
│                Content-Based Routing                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Route based on URL path:                                    │
│                                                              │
│       ┌───────────────────────────────┐                     │
│       │        L7 Load Balancer       │                     │
│       └───────────────┬───────────────┘                     │
│                       │                                      │
│     ┌─────────────────┼─────────────────┐                   │
│     │                 │                 │                   │
│     ▼                 ▼                 ▼                   │
│  /api/*           /static/*        /websocket/*             │
│     │                 │                 │                   │
│     ▼                 ▼                 ▼                   │
│ ┌───────┐        ┌───────┐        ┌───────┐                │
│ │  API  │        │  CDN  │        │  WS   │                │
│ │Servers│        │ Cache │        │Servers│                │
│ └───────┘        └───────┘        └───────┘                │
│                                                              │
│  Route based on headers:                                     │
│  • Mobile app (User-Agent) → Mobile backend                 │
│  • API version (X-API-Version) → v1/v2 servers             │
│  • A/B testing (Cookie) → Experiment variants              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Load Balancing Algorithms

```
┌─────────────────────────────────────────────────────────────┐
│              Load Balancing Algorithms                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Round Robin                                              │
│     Requests distributed sequentially                       │
│     Request 1 → Server A                                    │
│     Request 2 → Server B                                    │
│     Request 3 → Server C                                    │
│     Request 4 → Server A (repeat)                           │
│     ✓ Simple, even distribution                             │
│     ✗ Ignores server capacity/load                          │
│                                                              │
│  2. Weighted Round Robin                                     │
│     Higher weight = more requests                           │
│     Server A (weight 3): ●●●                               │
│     Server B (weight 2): ●●                                 │
│     Server C (weight 1): ●                                  │
│     ✓ Accounts for server capacity                          │
│     ✗ Static weights                                         │
│                                                              │
│  3. Least Connections                                        │
│     Route to server with fewest active connections          │
│     Server A: 10 connections                                │
│     Server B: 5 connections  ← next request                │
│     Server C: 8 connections                                 │
│     ✓ Adapts to actual load                                 │
│     ✓ Good for varying request duration                     │
│                                                              │
│  4. IP Hash                                                  │
│     hash(client_ip) % num_servers                           │
│     Same client → Same server (sticky sessions)            │
│     ✓ Session persistence                                   │
│     ✗ Uneven if IP distribution skewed                      │
│                                                              │
│  5. Least Response Time                                      │
│     Route to fastest responding server                      │
│     Combines: active connections + response time            │
│     ✓ Best user experience                                  │
│     ✗ More complex to implement                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Health Checks

```
┌─────────────────────────────────────────────────────────────┐
│                     Health Checks                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Passive Health Checks:                                      │
│  Monitor real traffic for failures                          │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ LB sends request → Server responds 5xx                 │ │
│  │ LB marks server as unhealthy after N failures         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Active Health Checks:                                       │
│  Periodically probe servers                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ LB ─── GET /health ───▶ Server                         │ │
│  │ LB ◀── 200 OK ─────────                                │ │
│  │                                                         │ │
│  │ Every 5 seconds, check each server                     │ │
│  │ 3 failures → mark unhealthy                           │ │
│  │ 2 successes → mark healthy again                      │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Health Check Response:                                      │
│  {                                                           │
│    "status": "healthy",                                     │
│    "database": "connected",                                 │
│    "cache": "connected",                                    │
│    "disk_space": "ok"                                       │
│  }                                                           │
│                                                              │
│  Graceful Shutdown:                                          │
│  1. Server receives shutdown signal                         │
│  2. Health endpoint returns 503                             │
│  3. LB stops sending new requests                           │
│  4. Server finishes existing requests                       │
│  5. Server shuts down                                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## High Availability

```
┌─────────────────────────────────────────────────────────────┐
│              Load Balancer High Availability                 │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Problem: LB itself is single point of failure              │
│                                                              │
│  Solution: Active-Passive LB pair                           │
│                                                              │
│           ┌──────────────────────────────────┐              │
│           │         Virtual IP (VIP)          │              │
│           │         203.0.113.10              │              │
│           └───────────────┬──────────────────┘              │
│                           │                                  │
│              ┌────────────┴────────────┐                    │
│              │                         │                    │
│              ▼                         ▼                    │
│       ┌──────────┐              ┌──────────┐               │
│       │  Active  │   heartbeat  │ Passive  │               │
│       │   LB 1   │◄────────────▶│   LB 2   │               │
│       └──────────┘              └──────────┘               │
│              │                         │                    │
│              │     If Active fails:   │                    │
│              │     Passive takes VIP  │                    │
│              │                         │                    │
│              └───────────┬─────────────┘                    │
│                          │                                   │
│            ┌─────────────┼─────────────┐                    │
│            ▼             ▼             ▼                    │
│       ┌────────┐   ┌────────┐   ┌────────┐                 │
│       │Server 1│   │Server 2│   │Server 3│                 │
│       └────────┘   └────────┘   └────────┘                 │
│                                                              │
│  Technologies:                                               │
│  • Keepalived (VRRP)                                        │
│  • AWS ELB/ALB (managed)                                    │
│  • Cloud load balancers (auto-HA)                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

### Basic
1. What is a load balancer and why is it needed?
2. Explain the difference between L4 and L7 load balancing.
3. What is round-robin load balancing?

### Intermediate
4. How do health checks work? What's the difference between active and passive?
5. When would you use sticky sessions? What are the drawbacks?
6. How do you make load balancers highly available?

### Advanced
7. Design a global load balancing strategy for a worldwide application.
8. How would you handle WebSocket connections with load balancing?
9. Explain how to do zero-downtime deployments with load balancers.

---

[← Back to Module](../README.md) | [Next: Reverse Proxy →](../02-reverse-proxy/README.md)
