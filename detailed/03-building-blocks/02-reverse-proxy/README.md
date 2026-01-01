# Reverse Proxy

A reverse proxy sits in front of web servers and forwards client requests to those servers. It provides security, performance, and reliability benefits.

## Table of Contents
1. [Forward vs Reverse Proxy](#forward-vs-reverse-proxy)
2. [Use Cases](#use-cases)
3. [Popular Solutions](#popular-solutions)
4. [Interview Questions](#interview-questions)

---

## Forward vs Reverse Proxy

```
┌─────────────────────────────────────────────────────────────┐
│              Forward Proxy vs Reverse Proxy                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Forward Proxy (Client-side):                                │
│  Sits in front of CLIENTS                                   │
│                                                              │
│  ┌────────┐    ┌─────────┐    ┌──────────┐                 │
│  │ Client │───▶│ Forward │───▶│ Internet │                 │
│  │        │    │  Proxy  │    │ Servers  │                 │
│  └────────┘    └─────────┘    └──────────┘                 │
│                                                              │
│  Use: Hide client identity, bypass restrictions             │
│  Example: Corporate proxy, VPN                              │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Reverse Proxy (Server-side):                                │
│  Sits in front of SERVERS                                   │
│                                                              │
│  ┌────────┐    ┌─────────┐    ┌──────────┐                 │
│  │ Client │───▶│ Reverse │───▶│ Backend  │                 │
│  │        │    │  Proxy  │    │ Servers  │                 │
│  └────────┘    └─────────┘    └──────────┘                 │
│                                                              │
│  Use: Hide server identity, load balance, cache            │
│  Example: Nginx, HAProxy, Cloudflare                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Use Cases

```
┌─────────────────────────────────────────────────────────────┐
│                 Reverse Proxy Use Cases                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. SSL/TLS Termination                                      │
│     ┌────────┐  HTTPS  ┌─────────┐  HTTP  ┌────────┐       │
│     │ Client │────────▶│  Proxy  │───────▶│ Server │       │
│     └────────┘         └─────────┘        └────────┘       │
│     Decrypt at proxy, forward plain HTTP internally        │
│     Reduces CPU load on application servers                 │
│                                                              │
│  2. Caching                                                  │
│     Cache static content (CSS, JS, images)                  │
│     Reduce load on backend servers                          │
│                                                              │
│  3. Compression                                              │
│     Gzip/Brotli compress responses                          │
│     Reduce bandwidth, faster page loads                     │
│                                                              │
│  4. Security                                                 │
│     • Hide backend server IPs                               │
│     • WAF (Web Application Firewall)                        │
│     • DDoS protection                                       │
│     • Rate limiting                                          │
│                                                              │
│  5. Load Balancing                                           │
│     Distribute traffic across servers                       │
│     (Often combined in same tool)                           │
│                                                              │
│  6. A/B Testing & Canary Deployments                        │
│     Route % of traffic to new version                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Popular Solutions

```
┌─────────────────────────────────────────────────────────────┐
│                   Popular Reverse Proxies                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Nginx:                                                      │
│  • Most popular web server/reverse proxy                    │
│  • High performance, low memory                             │
│  • Great for static content                                 │
│  • Configuration-based                                       │
│                                                              │
│  HAProxy:                                                    │
│  • Focused on load balancing                                │
│  • TCP and HTTP modes                                       │
│  • Advanced health checks                                   │
│  • Used by GitHub, Stack Overflow                           │
│                                                              │
│  Envoy:                                                      │
│  • Modern, cloud-native proxy                               │
│  • Built for microservices                                  │
│  • Used in service meshes (Istio)                          │
│  • Dynamic configuration via API                           │
│                                                              │
│  Traefik:                                                    │
│  • Auto-discovery in container environments                 │
│  • Native Kubernetes/Docker support                        │
│  • Dynamic configuration                                    │
│                                                              │
│  Cloud Solutions:                                            │
│  • AWS ALB/NLB                                              │
│  • Google Cloud Load Balancer                               │
│  • Cloudflare                                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

### Basic
1. What is the difference between forward and reverse proxy?
2. What is SSL termination and why is it useful?

### Intermediate
3. How would you configure a reverse proxy for zero-downtime deployments?
4. When would you use Nginx vs HAProxy?

### Advanced
5. Design a multi-tier reverse proxy architecture for a global application.

---

[← Previous: Load Balancers](../01-load-balancers/README.md) | [Next: CDN →](../03-cdn/README.md)
