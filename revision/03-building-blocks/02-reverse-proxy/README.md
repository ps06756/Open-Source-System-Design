# Reverse Proxy

A reverse proxy sits in front of servers and forwards client requests to the appropriate backend server.

## Table of Contents
- [What is a Reverse Proxy?](#what-is-a-reverse-proxy)
- [Forward vs Reverse Proxy](#forward-vs-reverse-proxy)
- [Use Cases](#use-cases)
- [Key Features](#key-features)
- [Implementation](#implementation)
- [Key Takeaways](#key-takeaways)

## What is a Reverse Proxy?

A reverse proxy accepts requests from clients and forwards them to backend servers. The client doesn't know which backend server handles the request.

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Client ──────▶ Reverse Proxy ──────▶ Backend Servers  │
│                      │                                  │
│              - SSL termination                          │
│              - Caching                                  │
│              - Compression                              │
│              - Security                                 │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## Forward vs Reverse Proxy

```
Forward Proxy (client-side):
┌────────┐     ┌────────────┐     ┌────────┐
│ Client │────▶│   Proxy    │────▶│ Server │
│(knows) │     │            │     │(doesn't│
│        │     │            │     │ know   │
│        │     │            │     │ client)│
└────────┘     └────────────┘     └────────┘
Use: Anonymity, caching, filtering for clients

Reverse Proxy (server-side):
┌────────┐     ┌────────────┐     ┌────────┐
│ Client │────▶│   Proxy    │────▶│ Server │
│(doesn't│     │            │     │(knows) │
│ know   │     │            │     │        │
│ server)│     │            │     │        │
└────────┘     └────────────┘     └────────┘
Use: Load balancing, security, caching for servers
```

## Use Cases

### 1. Load Balancing

Distribute requests across multiple servers.

```nginx
upstream backend {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080;
}

server {
    location / {
        proxy_pass http://backend;
    }
}
```

### 2. SSL Termination

Handle SSL at the proxy, send unencrypted traffic to backend.

```
Client ──HTTPS──▶ Nginx ──HTTP──▶ Backend
                   │
             Certificate
             installed here
```

### 3. Caching

Cache responses to reduce backend load.

```
Request 1: Client → Proxy → Backend → Response (cached)
Request 2: Client → Proxy → Cached Response (no backend call)
```

### 4. Compression

Compress responses to reduce bandwidth.

```nginx
gzip on;
gzip_types text/plain application/json application/javascript;
gzip_min_length 1000;
```

### 5. Security

- Hide backend server details
- Block malicious requests
- Rate limiting
- IP whitelisting

### 6. URL Rewriting

Transform URLs before forwarding.

```nginx
location /api/v1/ {
    rewrite ^/api/v1/(.*)$ /v1/$1 break;
    proxy_pass http://api-backend;
}
```

## Key Features

### Request Routing

```nginx
# Route based on path
location /api/ {
    proxy_pass http://api-servers;
}

location /static/ {
    proxy_pass http://static-servers;
}

# Route based on header
location / {
    if ($http_x_api_version = "v2") {
        proxy_pass http://api-v2;
    }
    proxy_pass http://api-v1;
}
```

### Connection Pooling

Maintain persistent connections to backends.

```
Without pooling:
Client → Proxy → [connect] → Backend → [close]
Client → Proxy → [connect] → Backend → [close]

With pooling:
Client → Proxy → [reuse connection] → Backend
Client → Proxy → [reuse connection] → Backend
```

### Request Buffering

Buffer requests before sending to backend.

```
Slow client:
Client ──slow──▶ Proxy (buffers) ──fast──▶ Backend
                     │
              Frees backend quickly
```

### Health Checks

Monitor backend health and route around failures.

```nginx
upstream backend {
    server 10.0.0.1:8080 max_fails=3 fail_timeout=30s;
    server 10.0.0.2:8080 max_fails=3 fail_timeout=30s;
    server 10.0.0.3:8080 backup;
}
```

## Implementation

### Nginx Configuration

```nginx
http {
    # Upstream servers
    upstream api {
        least_conn;
        server api1.internal:8080 weight=3;
        server api2.internal:8080 weight=2;
        keepalive 32;
    }

    # Caching
    proxy_cache_path /var/cache/nginx levels=1:2
                     keys_zone=cache:10m max_size=1g;

    server {
        listen 443 ssl http2;
        server_name api.example.com;

        # SSL
        ssl_certificate /etc/ssl/cert.pem;
        ssl_certificate_key /etc/ssl/key.pem;

        # Compression
        gzip on;
        gzip_types application/json;

        location /api/ {
            proxy_pass http://api;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # Timeouts
            proxy_connect_timeout 5s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;

            # Buffering
            proxy_buffering on;
            proxy_buffer_size 4k;
            proxy_buffers 8 4k;
        }

        location /static/ {
            proxy_pass http://static;
            proxy_cache cache;
            proxy_cache_valid 200 1d;
            add_header X-Cache-Status $upstream_cache_status;
        }
    }
}
```

### HAProxy Configuration

```haproxy
global
    maxconn 50000

defaults
    mode http
    timeout connect 5s
    timeout client 30s
    timeout server 30s

frontend http_front
    bind *:80
    bind *:443 ssl crt /etc/ssl/cert.pem

    # Route based on path
    acl is_api path_beg /api
    use_backend api_back if is_api
    default_backend web_back

backend api_back
    balance leastconn
    option httpchk GET /health
    server api1 10.0.0.1:8080 check
    server api2 10.0.0.2:8080 check

backend web_back
    balance roundrobin
    server web1 10.0.0.10:8080 check
    server web2 10.0.0.11:8080 check
```

### Comparison: Nginx vs HAProxy

| Feature | Nginx | HAProxy |
|---------|-------|---------|
| HTTP/2 | Yes | Yes |
| Static content | Excellent | Via backends |
| TCP proxy | Yes | Yes |
| Config reload | Yes | Yes |
| Caching | Built-in | Via backends |
| Lua scripting | Yes | No (but has ACLs) |
| Memory usage | Higher | Lower |
| Performance | Excellent | Excellent |

## Architecture Patterns

### Single Reverse Proxy

```
┌────────┐     ┌────────────┐     ┌────────┐
│ Client │────▶│   Nginx    │────▶│ Backend│
└────────┘     └────────────┘     └────────┘

Simple, but single point of failure
```

### High Availability

```
                    ┌───────────────┐
                    │   Keepalived  │
                    │   (VIP)       │
                    └───────┬───────┘
                            │
              ┌─────────────┴─────────────┐
              │                           │
        ┌───────────┐             ┌───────────┐
        │  Nginx 1  │             │  Nginx 2  │
        │ (active)  │             │ (standby) │
        └─────┬─────┘             └─────┬─────┘
              │                         │
              └────────────┬────────────┘
                           │
                    ┌──────┴──────┐
                    │   Backends  │
                    └─────────────┘
```

### Multi-tier

```
┌────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│  CDN   │────▶│ Edge    │────▶│ Origin  │────▶│ Backend │
│        │     │ Proxy   │     │ Proxy   │     │         │
└────────┘     └─────────┘     └─────────┘     └─────────┘

Edge: Geographic distribution
Origin: Application routing
```

## Key Takeaways

1. **Reverse proxy is foundational** - almost every system uses one
2. **SSL termination** simplifies certificate management
3. **Caching at the proxy** reduces backend load significantly
4. **Connection pooling** improves performance to backends
5. **Health checks** enable automatic failover
6. **Nginx vs HAProxy** - both excellent, choose based on needs

## Practice Questions

1. Design a reverse proxy setup for a multi-region deployment.
2. How would you configure caching for an API with user-specific data?
3. What headers should a reverse proxy add/modify?
4. How do you handle WebSocket connections through a reverse proxy?

## Further Reading

- [Nginx Admin Guide](https://docs.nginx.com/nginx/admin-guide/)
- [HAProxy Configuration Manual](http://cbonte.github.io/haproxy-dconv/)
- [Envoy Proxy](https://www.envoyproxy.io/)

---

Next: [CDN](../03-cdn/README.md)
