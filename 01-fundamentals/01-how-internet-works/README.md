# How the Internet Works

Understanding the internet's architecture is fundamental to designing distributed systems. This chapter covers DNS, TCP/IP, HTTP, and TLS.

## Table of Contents
- [Overview](#overview)
- [DNS (Domain Name System)](#dns-domain-name-system)
- [TCP/IP Model](#tcpip-model)
- [HTTP/HTTPS](#httphttps)
- [TLS/SSL](#tlsssl)
- [Putting It All Together](#putting-it-all-together)
- [Key Takeaways](#key-takeaways)

## Overview

When you type `google.com` in your browser, a complex series of events occurs:

```
┌──────────┐    DNS     ┌──────────┐    TCP     ┌──────────┐    HTTP    ┌──────────┐
│  Browser │──────────▶│   DNS    │           │  Google  │◀──────────│  Browser │
│          │◀──────────│  Server  │           │  Server  │──────────▶│          │
└──────────┘  IP Addr   └──────────┘           └──────────┘  Response  └──────────┘
                                    └─────────────────────────┘
                                         TCP Handshake
```

1. **DNS Resolution**: Browser looks up IP address for `google.com`
2. **TCP Connection**: Browser establishes connection with server
3. **TLS Handshake**: If HTTPS, encryption is negotiated
4. **HTTP Request**: Browser sends GET request
5. **HTTP Response**: Server sends back HTML

## DNS (Domain Name System)

DNS is the internet's phone book. It translates human-readable domain names to IP addresses.

### How DNS Resolution Works

```
┌────────┐     ┌───────────┐     ┌────────────┐     ┌─────────────┐     ┌─────────────┐
│Browser │────▶│  Local    │────▶│   Root     │────▶│    TLD      │────▶│Authoritative│
│        │     │  Resolver │     │   Server   │     │   Server    │     │   Server    │
└────────┘     └───────────┘     └────────────┘     └─────────────┘     └─────────────┘
                    │                  │                  │                    │
                    │   "Where is      │  "Ask .com       │   "Ask             │
                    │   google.com?"   │   server"        │   ns.google.com"   │
                    │                  │                  │                    │
                    ◀──────────────────┴──────────────────┴────────────────────┘
                              Returns: 142.250.80.46
```

### DNS Record Types

| Record | Purpose | Example |
|--------|---------|---------|
| **A** | Maps domain to IPv4 | `google.com → 142.250.80.46` |
| **AAAA** | Maps domain to IPv6 | `google.com → 2607:f8b0:4004:800::200e` |
| **CNAME** | Alias to another domain | `www.google.com → google.com` |
| **MX** | Mail server | `google.com → smtp.google.com` |
| **NS** | Name server | `google.com → ns1.google.com` |
| **TXT** | Text data | Used for verification, SPF |

### DNS Caching

DNS responses are cached at multiple levels:

1. **Browser cache**: Chrome caches for ~1 minute
2. **OS cache**: System-level DNS cache
3. **Router cache**: Home/office router
4. **ISP cache**: ISP's recursive resolver

**TTL (Time To Live)**: Each DNS record has a TTL specifying how long it can be cached.

### DNS in System Design

**Considerations:**
- DNS propagation can take 24-48 hours globally
- Use low TTL (e.g., 60s) for services that need quick failover
- DNS can be used for load balancing (Round Robin DNS)
- GeoDNS routes users to nearest data center

**Common patterns:**
- **Failover**: Point to backup server if primary fails
- **Load distribution**: Return different IPs for each query
- **Geographic routing**: Return IP based on user location

## TCP/IP Model

The TCP/IP model is a layered architecture for network communication.

### The Four Layers

```
┌─────────────────────────────────────┐
│  Application Layer (HTTP, FTP, SMTP)│  ← Data
├─────────────────────────────────────┤
│  Transport Layer (TCP, UDP)         │  ← Segments
├─────────────────────────────────────┤
│  Internet Layer (IP)                │  ← Packets
├─────────────────────────────────────┤
│  Network Access Layer (Ethernet)    │  ← Frames
└─────────────────────────────────────┘
```

### TCP vs UDP

| Feature | TCP | UDP |
|---------|-----|-----|
| Connection | Connection-oriented | Connectionless |
| Reliability | Guaranteed delivery | Best effort |
| Ordering | Ordered | No ordering |
| Speed | Slower (overhead) | Faster |
| Use cases | HTTP, Email, File transfer | Video streaming, Gaming, DNS |

### TCP Three-Way Handshake

```
    Client                    Server
       │                         │
       │──────── SYN ──────────▶│
       │                         │
       │◀─────── SYN-ACK ───────│
       │                         │
       │──────── ACK ──────────▶│
       │                         │
       │     Connection          │
       │     Established         │
```

**Latency impact**: This handshake adds 1 Round Trip Time (RTT) before data can flow.

### TCP Flow Control & Congestion Control

**Flow Control (Receiver-side)**:
- Receiver advertises window size
- Sender doesn't exceed receiver's capacity

**Congestion Control (Network-side)**:
- **Slow Start**: Begin with small window, double each RTT
- **Congestion Avoidance**: Linear increase after threshold
- **Fast Retransmit**: Retransmit on 3 duplicate ACKs

### IP Addressing

**IPv4**: 32-bit address (e.g., `192.168.1.1`)
- ~4.3 billion addresses (exhausted)
- Private ranges: `10.x.x.x`, `172.16-31.x.x`, `192.168.x.x`

**IPv6**: 128-bit address (e.g., `2001:0db8:85a3:0000:0000:8a2e:0370:7334`)
- Virtually unlimited addresses
- Slowly being adopted

## HTTP/HTTPS

HTTP (Hypertext Transfer Protocol) is the foundation of web communication.

### HTTP Request Structure

```
GET /search?q=system+design HTTP/1.1
Host: www.google.com
User-Agent: Mozilla/5.0
Accept: text/html
Accept-Language: en-US
Connection: keep-alive
Cookie: session=abc123
```

### HTTP Response Structure

```
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
Content-Length: 12345
Cache-Control: max-age=3600
Set-Cookie: session=xyz789

<!DOCTYPE html>
<html>...
```

### HTTP Methods

| Method | Purpose | Idempotent | Safe |
|--------|---------|------------|------|
| GET | Retrieve resource | Yes | Yes |
| POST | Create resource | No | No |
| PUT | Replace resource | Yes | No |
| PATCH | Partial update | No | No |
| DELETE | Remove resource | Yes | No |
| HEAD | Get headers only | Yes | Yes |
| OPTIONS | Get allowed methods | Yes | Yes |

### HTTP Status Codes

| Range | Category | Common Codes |
|-------|----------|--------------|
| 1xx | Informational | 101 Switching Protocols |
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirection | 301 Moved Permanently, 302 Found, 304 Not Modified |
| 4xx | Client Error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found |
| 5xx | Server Error | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable |

### HTTP/1.1 vs HTTP/2 vs HTTP/3

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|--------|--------|
| Connections | Multiple TCP | Single TCP, multiplexed | QUIC (UDP-based) |
| Header compression | None | HPACK | QPACK |
| Server push | No | Yes | Yes |
| Head-of-line blocking | Yes | Yes (TCP level) | No |
| Encryption | Optional | Practically required | Required |

### HTTP/2 Multiplexing

```
HTTP/1.1 (Sequential):
┌────────┐     ┌────────┐     ┌────────┐
│ Req 1  │────▶│ Req 2  │────▶│ Req 3  │
└────────┘     └────────┘     └────────┘

HTTP/2 (Multiplexed):
┌────────┬────────┬────────┐
│ Req 1  │ Req 2  │ Req 3  │  ← Single connection
└────────┴────────┴────────┘
```

## TLS/SSL

TLS (Transport Layer Security) provides encryption for HTTPS.

### TLS Handshake

```
    Client                              Server
       │                                   │
       │─────── Client Hello ─────────────▶│  (Supported cipher suites, random)
       │                                   │
       │◀────── Server Hello ──────────────│  (Chosen cipher, certificate)
       │                                   │
       │   (Client verifies certificate)   │
       │                                   │
       │─────── Key Exchange ─────────────▶│  (Pre-master secret)
       │                                   │
       │◀────── Finished ──────────────────│
       │─────── Finished ─────────────────▶│
       │                                   │
       │     Encrypted Communication       │
```

**Latency**: TLS 1.2 adds 2 RTT. TLS 1.3 reduces this to 1 RTT (0-RTT for resumed connections).

### Certificate Chain

```
┌─────────────────────────────┐
│     Root CA Certificate     │  ← Pre-installed in browsers/OS
├─────────────────────────────┤
│  Intermediate CA Certificate│  ← Signs server certificates
├─────────────────────────────┤
│    Server Certificate       │  ← Your website's certificate
└─────────────────────────────┘
```

### TLS in System Design

- **Certificate management**: Use Let's Encrypt or managed certificates
- **TLS termination**: Often done at load balancer level
- **Internal traffic**: May skip TLS within trusted networks (trade-off)
- **mTLS (Mutual TLS)**: Both client and server present certificates

## Putting It All Together

When you visit `https://google.com`:

```
1. DNS Lookup (~50ms)
   Browser → DNS Resolver → Root → .com TLD → google.com NS
   Returns: 142.250.80.46

2. TCP Handshake (~30ms)
   SYN → SYN-ACK → ACK

3. TLS Handshake (~50ms)
   Client Hello → Server Hello → Certificate → Key Exchange → Finished

4. HTTP Request (~20ms)
   GET / HTTP/2

5. HTTP Response (~100ms)
   200 OK + HTML content

Total: ~250ms for first byte
```

### Latency Breakdown by Region

| Hop | Same Region | Cross-Region | Cross-Continent |
|-----|-------------|--------------|-----------------|
| DNS | 1-5ms | 10-50ms | 50-100ms |
| TCP | 1-5ms | 30-70ms | 100-200ms |
| TLS | 2-10ms | 60-140ms | 200-400ms |
| **Total overhead** | **~20ms** | **~200ms** | **~500ms+** |

## Key Takeaways

1. **DNS resolution adds latency** - Cache aggressively, use low TTL for failover
2. **TCP handshake costs 1 RTT** - Use connection pooling, keep-alive
3. **TLS adds 1-2 RTT** - Use TLS 1.3, session resumption
4. **HTTP/2 reduces connections** - Single multiplexed connection
5. **Geography matters** - Deploy close to users (CDN, edge)

## Practice Questions

1. What happens when DNS cache is cold vs warm?
2. Why might you choose UDP over TCP for a real-time application?
3. How does HTTP/2 solve head-of-line blocking? Does HTTP/3 improve on this?
4. What's the latency difference between HTTP and HTTPS?

## Further Reading

- [How DNS Works](https://howdns.works/) - Comic explaining DNS
- [High Performance Browser Networking](https://hpbn.co/) - Ilya Grigorik
- [HTTP/3 Explained](https://http3-explained.haxx.se/)

---

Next: [Client-Server Architecture](../02-client-server/README.md)
