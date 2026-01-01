# How the Internet Works

Understanding how data travels across the internet is fundamental to designing distributed systems. This chapter covers the complete journey of a web request.

## Table of Contents
1. [The Big Picture](#the-big-picture)
2. [DNS: The Internet's Phone Book](#dns-the-internets-phone-book)
3. [TCP/IP: The Foundation](#tcpip-the-foundation)
4. [HTTP/HTTPS: Application Layer](#httphttps-application-layer)
5. [TLS: Securing Communications](#tls-securing-communications)
6. [Putting It All Together](#putting-it-all-together)
7. [Interview Questions](#interview-questions)

---

## The Big Picture

When you type `https://www.google.com` in your browser, here's what happens:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Journey of a Web Request                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. DNS Lookup          2. TCP Handshake       3. TLS Handshake             │
│  ┌──────────┐           ┌──────────────┐       ┌──────────────┐             │
│  │ Browser  │──────────▶│ DNS Server   │       │              │             │
│  │          │◀──────────│ "142.250.x.x"│       │  Establish   │             │
│  └──────────┘           └──────────────┘       │  Encryption  │             │
│       │                                        └──────────────┘             │
│       │                                               │                      │
│       ▼                                               ▼                      │
│  4. HTTP Request        5. Server Processing   6. HTTP Response             │
│  ┌──────────┐           ┌──────────────┐       ┌──────────────┐             │
│  │ GET /    │──────────▶│   Google     │──────▶│ 200 OK       │             │
│  │          │           │   Servers    │       │ <html>...    │             │
│  └──────────┘           └──────────────┘       └──────────────┘             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Total time**: 100-500ms for a typical request

Let's dive deep into each step.

---

## DNS: The Internet's Phone Book

### What is DNS?

The Domain Name System (DNS) translates human-readable domain names (like `google.com`) into IP addresses (like `142.250.190.14`) that computers use to identify each other.

### DNS Hierarchy

DNS is a hierarchical, distributed database:

```
                    ┌─────────────────┐
                    │   Root DNS (.)   │
                    │  13 root servers │
                    └────────┬────────┘
                             │
           ┌─────────────────┼─────────────────┐
           │                 │                 │
           ▼                 ▼                 ▼
    ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
    │  .com TLD   │   │  .org TLD   │   │  .io TLD    │
    └──────┬──────┘   └─────────────┘   └─────────────┘
           │
     ┌─────┴─────┐
     │           │
     ▼           ▼
┌─────────┐ ┌─────────┐
│ google  │ │ amazon  │
│  .com   │ │  .com   │
└─────────┘ └─────────┘
```

### DNS Resolution Process

```
┌──────────┐     ┌───────────────┐     ┌─────────────┐     ┌─────────────┐
│ Browser  │────▶│ Local DNS     │────▶│ Root DNS    │────▶│ .com TLD    │
│          │     │ Resolver      │     │ Server      │     │ Server      │
└──────────┘     │ (ISP/Company) │     └─────────────┘     └─────────────┘
                 └───────────────┘                                │
                        ▲                                         │
                        │                                         ▼
                        │                               ┌─────────────────┐
                        └───────────────────────────────│ google.com      │
                              IP: 142.250.190.14        │ Authoritative   │
                                                        │ DNS Server      │
                                                        └─────────────────┘
```

### DNS Record Types

| Type | Purpose | Example |
|------|---------|---------|
| **A** | Maps domain to IPv4 | `google.com → 142.250.190.14` |
| **AAAA** | Maps domain to IPv6 | `google.com → 2607:f8b0:4004:...` |
| **CNAME** | Alias to another domain | `www.google.com → google.com` |
| **MX** | Mail server | `google.com → mail.google.com` |
| **NS** | Name server | `google.com → ns1.google.com` |
| **TXT** | Text records (SPF, verification) | Various |

### DNS Caching

DNS responses are cached at multiple levels to reduce latency:

```
┌─────────────────────────────────────────────────────────────┐
│                    DNS Caching Layers                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Layer 1: Browser Cache                                      │
│  ├── Chrome: chrome://net-internals/#dns                     │
│  └── TTL: Usually 1 minute                                   │
│                                                              │
│  Layer 2: Operating System Cache                             │
│  ├── Linux: systemd-resolved                                 │
│  ├── macOS: mDNSResponder                                    │
│  └── Windows: DNS Client service                             │
│                                                              │
│  Layer 3: Router Cache                                       │
│  └── Your home/office router                                 │
│                                                              │
│  Layer 4: ISP DNS Resolver                                   │
│  └── Or public DNS (8.8.8.8, 1.1.1.1)                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### TTL (Time To Live)

Each DNS record has a TTL that determines how long it can be cached:

```python
# Example DNS response
{
    "name": "google.com",
    "type": "A",
    "ttl": 300,  # Cache for 5 minutes
    "value": "142.250.190.14"
}
```

**Trade-offs:**
- **Short TTL (60-300s)**: Faster failover, more DNS queries, higher load
- **Long TTL (3600s+)**: Better caching, slower updates, stale data risk

### System Design Implications

1. **DNS-based Load Balancing**: Return different IPs for the same domain
2. **GeoDNS**: Return IPs based on user location
3. **Failover**: Change DNS when servers fail (limited by TTL)
4. **CDN Integration**: Route to nearest edge server

---

## TCP/IP: The Foundation

### The OSI and TCP/IP Models

```
┌─────────────────────────────────────────────────────────────┐
│              OSI Model          │     TCP/IP Model          │
├─────────────────────────────────┼───────────────────────────┤
│  7. Application  (HTTP, FTP)    │                           │
│  6. Presentation (SSL, TLS)     │  Application Layer        │
│  5. Session      (Sockets)      │  (HTTP, FTP, SMTP, DNS)   │
├─────────────────────────────────┼───────────────────────────┤
│  4. Transport    (TCP, UDP)     │  Transport Layer          │
│                                 │  (TCP, UDP)               │
├─────────────────────────────────┼───────────────────────────┤
│  3. Network      (IP, ICMP)     │  Internet Layer           │
│                                 │  (IP, ICMP, ARP)          │
├─────────────────────────────────┼───────────────────────────┤
│  2. Data Link    (Ethernet)     │  Network Access Layer     │
│  1. Physical     (Cables)       │  (Ethernet, Wi-Fi)        │
└─────────────────────────────────┴───────────────────────────┘
```

### IP (Internet Protocol)

IP handles addressing and routing packets across networks.

**IPv4 Address**: 32 bits, e.g., `192.168.1.1`
**IPv6 Address**: 128 bits, e.g., `2001:0db8:85a3:0000:0000:8a2e:0370:7334`

```
┌─────────────────────────────────────────────────────────────┐
│                     IP Packet Structure                      │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────┐   │
│  │ IP Header (20-60 bytes)                               │   │
│  │ ┌─────────────┬─────────────┬────────────────────┐   │   │
│  │ │ Version (4) │ Header Len  │ Total Length       │   │   │
│  │ ├─────────────┼─────────────┼────────────────────┤   │   │
│  │ │ TTL         │ Protocol    │ Header Checksum    │   │   │
│  │ ├─────────────┴─────────────┴────────────────────┤   │   │
│  │ │ Source IP Address                               │   │   │
│  │ ├────────────────────────────────────────────────┤   │   │
│  │ │ Destination IP Address                          │   │   │
│  │ └────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Data Payload (TCP/UDP segment)                        │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### TCP (Transmission Control Protocol)

TCP provides reliable, ordered, error-checked delivery of data.

#### TCP Three-Way Handshake

```
┌──────────┐                                    ┌──────────┐
│  Client  │                                    │  Server  │
└────┬─────┘                                    └────┬─────┘
     │                                               │
     │  1. SYN (seq=100)                            │
     │──────────────────────────────────────────────▶│
     │                                               │
     │  2. SYN-ACK (seq=300, ack=101)               │
     │◀──────────────────────────────────────────────│
     │                                               │
     │  3. ACK (seq=101, ack=301)                   │
     │──────────────────────────────────────────────▶│
     │                                               │
     │  ═══════ Connection Established ═══════      │
     │                                               │
```

**Why three-way?**
- Establishes both directions of communication
- Synchronizes sequence numbers
- Prevents old duplicate connections

#### TCP Connection Termination (Four-Way Handshake)

```
┌──────────┐                                    ┌──────────┐
│  Client  │                                    │  Server  │
└────┬─────┘                                    └────┬─────┘
     │                                               │
     │  1. FIN                                       │
     │──────────────────────────────────────────────▶│
     │                                               │
     │  2. ACK                                       │
     │◀──────────────────────────────────────────────│
     │                                               │
     │  3. FIN                                       │
     │◀──────────────────────────────────────────────│
     │                                               │
     │  4. ACK                                       │
     │──────────────────────────────────────────────▶│
     │                                               │
     │  ═══════ Connection Closed ═══════           │
     │                                               │
```

#### TCP Features

| Feature | Description | System Design Impact |
|---------|-------------|---------------------|
| **Reliability** | Guaranteed delivery via ACKs | No data loss |
| **Ordering** | Sequence numbers maintain order | No reordering |
| **Flow Control** | Receiver controls sender's rate | Prevents overflow |
| **Congestion Control** | Adapts to network conditions | Fair bandwidth sharing |

### UDP (User Datagram Protocol)

UDP is a simple, connectionless protocol with no reliability guarantees.

```
┌─────────────────────────────────────────────────────────────┐
│                      TCP vs UDP                              │
├────────────────────────────┬────────────────────────────────┤
│          TCP               │            UDP                  │
├────────────────────────────┼────────────────────────────────┤
│ Connection-oriented        │ Connectionless                  │
│ Reliable delivery          │ Best-effort delivery           │
│ Ordered packets            │ No ordering guarantees         │
│ Flow & congestion control  │ No flow control                │
│ Higher latency             │ Lower latency                  │
│ Higher overhead            │ Lower overhead                 │
├────────────────────────────┼────────────────────────────────┤
│ Use: HTTP, FTP, Email      │ Use: DNS, Video, Gaming        │
└────────────────────────────┴────────────────────────────────┘
```

### Ports

Ports allow multiple applications to share a single IP address:

```
┌───────────────────────────────────────────────────────────┐
│                     Common Ports                           │
├──────────────┬────────────────────────────────────────────┤
│ Port         │ Service                                     │
├──────────────┼────────────────────────────────────────────┤
│ 20, 21       │ FTP (File Transfer)                        │
│ 22           │ SSH (Secure Shell)                         │
│ 25           │ SMTP (Email)                               │
│ 53           │ DNS                                        │
│ 80           │ HTTP                                       │
│ 443          │ HTTPS                                      │
│ 3306         │ MySQL                                      │
│ 5432         │ PostgreSQL                                 │
│ 6379         │ Redis                                      │
│ 27017        │ MongoDB                                    │
└──────────────┴────────────────────────────────────────────┘
```

---

## HTTP/HTTPS: Application Layer

### HTTP Request Structure

```
┌─────────────────────────────────────────────────────────────┐
│                    HTTP Request                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Request Line:                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ GET /api/users?id=123 HTTP/1.1                         │ │
│  │ ▲    ▲                 ▲                               │ │
│  │ │    │                 └── Protocol Version            │ │
│  │ │    └── Path + Query String                          │ │
│  │ └── Method (GET, POST, PUT, DELETE, etc.)             │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Headers:                                                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Host: api.example.com                                  │ │
│  │ User-Agent: Mozilla/5.0...                             │ │
│  │ Accept: application/json                               │ │
│  │ Authorization: Bearer eyJ...                           │ │
│  │ Content-Type: application/json                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Body (for POST, PUT, PATCH):                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ {"name": "John", "email": "john@example.com"}          │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### HTTP Response Structure

```
┌─────────────────────────────────────────────────────────────┐
│                    HTTP Response                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Status Line:                                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ HTTP/1.1 200 OK                                        │ │
│  │ ▲        ▲   ▲                                         │ │
│  │ │        │   └── Reason Phrase                         │ │
│  │ │        └── Status Code                               │ │
│  │ └── Protocol Version                                   │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Headers:                                                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Content-Type: application/json                         │ │
│  │ Content-Length: 1234                                   │ │
│  │ Cache-Control: max-age=3600                            │ │
│  │ Set-Cookie: session=abc123                             │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Body:                                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ {"id": 123, "name": "John", "status": "active"}        │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### HTTP Status Codes

| Range | Category | Examples |
|-------|----------|----------|
| **1xx** | Informational | 100 Continue, 101 Switching Protocols |
| **2xx** | Success | 200 OK, 201 Created, 204 No Content |
| **3xx** | Redirection | 301 Moved Permanently, 302 Found, 304 Not Modified |
| **4xx** | Client Error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found |
| **5xx** | Server Error | 500 Internal Error, 502 Bad Gateway, 503 Service Unavailable |

### HTTP Methods

| Method | Idempotent | Safe | Cacheable | Use Case |
|--------|------------|------|-----------|----------|
| GET | Yes | Yes | Yes | Retrieve resource |
| POST | No | No | No* | Create resource |
| PUT | Yes | No | No | Replace resource |
| PATCH | No | No | No | Partial update |
| DELETE | Yes | No | No | Delete resource |
| HEAD | Yes | Yes | Yes | Get headers only |
| OPTIONS | Yes | Yes | No | Get allowed methods |

### HTTP/1.1 vs HTTP/2 vs HTTP/3

```
┌─────────────────────────────────────────────────────────────┐
│                    HTTP Evolution                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  HTTP/1.1 (1997)                                            │
│  ├── Text-based protocol                                    │
│  ├── One request per connection (or keep-alive)             │
│  ├── Head-of-line blocking                                  │
│  └── Headers sent with every request                        │
│                                                              │
│  HTTP/2 (2015)                                              │
│  ├── Binary protocol                                        │
│  ├── Multiplexing (multiple requests per connection)        │
│  ├── Header compression (HPACK)                             │
│  ├── Server push                                            │
│  └── Still has TCP head-of-line blocking                    │
│                                                              │
│  HTTP/3 (2022)                                              │
│  ├── Built on QUIC (UDP-based)                              │
│  ├── No TCP head-of-line blocking                           │
│  ├── Faster connection establishment (0-RTT)                │
│  └── Built-in encryption                                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## TLS: Securing Communications

### What TLS Provides

1. **Encryption**: Data is unreadable to eavesdroppers
2. **Authentication**: Server (and optionally client) identity verified
3. **Integrity**: Data hasn't been tampered with

### TLS Handshake (TLS 1.2)

```
┌──────────┐                                    ┌──────────┐
│  Client  │                                    │  Server  │
└────┬─────┘                                    └────┬─────┘
     │                                               │
     │  1. ClientHello                               │
     │  (supported ciphers, random number)           │
     │──────────────────────────────────────────────▶│
     │                                               │
     │  2. ServerHello                               │
     │  (chosen cipher, random number)               │
     │◀──────────────────────────────────────────────│
     │                                               │
     │  3. Certificate                               │
     │  (server's public key certificate)            │
     │◀──────────────────────────────────────────────│
     │                                               │
     │  4. ServerHelloDone                           │
     │◀──────────────────────────────────────────────│
     │                                               │
     │  5. ClientKeyExchange                         │
     │  (pre-master secret, encrypted)               │
     │──────────────────────────────────────────────▶│
     │                                               │
     │  6. ChangeCipherSpec                          │
     │──────────────────────────────────────────────▶│
     │                                               │
     │  7. Finished (encrypted)                      │
     │──────────────────────────────────────────────▶│
     │                                               │
     │  8. ChangeCipherSpec                          │
     │◀──────────────────────────────────────────────│
     │                                               │
     │  9. Finished (encrypted)                      │
     │◀──────────────────────────────────────────────│
     │                                               │
     │  ═══════ Secure Connection Established ═══════│
     │                                               │
```

### TLS 1.3 Improvements

```
┌─────────────────────────────────────────────────────────────┐
│                    TLS 1.2 vs TLS 1.3                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  TLS 1.2: 2 round trips (2-RTT)                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Client ──▶ Server ──▶ Client ──▶ Server ──▶ Client    │ │
│  │         ↑         ↑          ↑          ↑              │ │
│  │        RTT 1              RTT 2                        │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  TLS 1.3: 1 round trip (1-RTT), or 0-RTT for resumption     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Client ──▶ Server ──▶ Client                           │ │
│  │         ↑         ↑                                    │ │
│  │        RTT 1                                           │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Key Improvements:                                           │
│  • Removed insecure algorithms (RSA, SHA-1, etc.)           │
│  • Encrypted more of the handshake                          │
│  • 0-RTT resumption for repeat connections                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Certificates and PKI

```
┌─────────────────────────────────────────────────────────────┐
│                Certificate Chain                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │               Root CA Certificate                     │    │
│  │        (Built into browsers/OS - ~150 trusted)       │    │
│  │              DigiCert, Let's Encrypt                 │    │
│  └─────────────────────┬───────────────────────────────┘    │
│                        │ Signs                               │
│                        ▼                                     │
│  ┌─────────────────────────────────────────────────────┐    │
│  │            Intermediate CA Certificate                │    │
│  │              (Reduces root CA exposure)               │    │
│  └─────────────────────┬───────────────────────────────┘    │
│                        │ Signs                               │
│                        ▼                                     │
│  ┌─────────────────────────────────────────────────────┐    │
│  │             End-Entity Certificate                    │    │
│  │           (Your server's certificate)                 │    │
│  │              www.yoursite.com                        │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Putting It All Together

### Complete Request Timeline

```
┌─────────────────────────────────────────────────────────────┐
│              Full HTTPS Request Timeline                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Time (ms)   Action                                          │
│  ─────────   ──────                                          │
│                                                              │
│  0           User types URL                                  │
│                                                              │
│  0-50        DNS Lookup                                      │
│              ├── Check browser cache (0-1ms)                 │
│              ├── Check OS cache (0-1ms)                      │
│              ├── Query recursive resolver                    │
│              └── Cache miss: full resolution                 │
│                                                              │
│  50-100      TCP Handshake (1 RTT)                          │
│              └── SYN → SYN-ACK → ACK                        │
│                                                              │
│  100-200     TLS Handshake (1-2 RTT)                        │
│              ├── TLS 1.2: 2 round trips                     │
│              └── TLS 1.3: 1 round trip                      │
│                                                              │
│  200-250     HTTP Request                                    │
│              └── Send GET/POST request                       │
│                                                              │
│  250-400     Server Processing                               │
│              ├── Parse request                               │
│              ├── Execute business logic                      │
│              ├── Query database                              │
│              └── Generate response                           │
│                                                              │
│  400-450     HTTP Response                                   │
│              └── Receive response data                       │
│                                                              │
│  450-500     Render                                          │
│              └── Browser renders content                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Optimization Techniques

| Technique | What It Helps | How |
|-----------|--------------|-----|
| **Connection Pooling** | Avoid TCP handshakes | Reuse connections |
| **HTTP/2** | Reduce latency | Multiplexing |
| **DNS Prefetching** | Reduce DNS time | `<link rel="dns-prefetch">` |
| **TLS Session Resumption** | Reduce TLS time | Session tickets |
| **CDN** | Reduce latency | Edge servers closer to users |
| **Compression** | Reduce transfer time | gzip, Brotli |

---

## Interview Questions

### Basic
1. What happens when you type a URL in your browser?
2. What is the difference between TCP and UDP?
3. What are the common HTTP status codes?

### Intermediate
4. Explain the TCP three-way handshake. Why three steps?
5. How does DNS caching work? What are the trade-offs of TTL?
6. What's the difference between HTTP/1.1 and HTTP/2?

### Advanced
7. How would you design a system to minimize latency for a global user base?
8. Explain how TLS provides security. What are the performance implications?
9. What is TCP head-of-line blocking and how does HTTP/3 solve it?

---

[← Back to Module](../README.md) | [Next: Client-Server Architecture →](../02-client-server/README.md)
