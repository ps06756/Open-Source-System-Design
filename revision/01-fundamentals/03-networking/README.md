# Networking Basics

Understanding network communication patterns is essential for designing real-time and scalable systems. This chapter covers sockets, protocols, and communication patterns.

## Table of Contents
- [Sockets](#sockets)
- [Communication Patterns](#communication-patterns)
- [WebSockets](#websockets)
- [Long Polling](#long-polling)
- [Server-Sent Events (SSE)](#server-sent-events-sse)
- [Comparison](#comparison)
- [Key Takeaways](#key-takeaways)

## Sockets

A socket is an endpoint for communication between two machines.

### Socket Basics

```
┌─────────────────┐                    ┌─────────────────┐
│     Client      │                    │     Server      │
│                 │                    │                 │
│  ┌───────────┐  │                    │  ┌───────────┐  │
│  │  Socket   │──┼────────────────────┼──│  Socket   │  │
│  │           │  │     TCP/UDP        │  │           │  │
│  │ IP:Port   │  │    Connection      │  │ IP:Port   │  │
│  └───────────┘  │                    │  └───────────┘  │
│                 │                    │                 │
│ 192.168.1.5     │                    │ 10.0.0.1        │
│ :54321          │                    │ :80             │
└─────────────────┘                    └─────────────────┘
```

### Socket Lifecycle (TCP)

**Server:**
```
1. socket()    - Create socket
2. bind()      - Bind to IP:Port
3. listen()    - Start listening
4. accept()    - Accept connection (blocks)
5. recv/send() - Communicate
6. close()     - Close connection
```

**Client:**
```
1. socket()    - Create socket
2. connect()   - Connect to server
3. send/recv() - Communicate
4. close()     - Close connection
```

### Socket Programming Example (Python)

**Server:**
```python
import socket

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.bind(('0.0.0.0', 8080))
server.listen(5)

while True:
    client, address = server.accept()
    data = client.recv(1024)
    client.send(b'Hello from server!')
    client.close()
```

**Client:**
```python
import socket

client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
client.connect(('localhost', 8080))
client.send(b'Hello from client!')
response = client.recv(1024)
client.close()
```

### Port Numbers

| Range | Type | Examples |
|-------|------|----------|
| 0-1023 | Well-known | 80 (HTTP), 443 (HTTPS), 22 (SSH) |
| 1024-49151 | Registered | 3306 (MySQL), 5432 (PostgreSQL) |
| 49152-65535 | Dynamic/Private | Client-side ports |

### Connection Limits

**Per-server limits:**
- File descriptors (default ~1024, can increase to ~1M)
- Memory per connection (~10KB minimum)
- Port exhaustion (client-side, 65535 ports)

**C10K Problem**: Handling 10,000 concurrent connections
- Solution: Event-driven architecture (epoll, kqueue)
- Modern servers handle 1M+ connections

## Communication Patterns

### Request-Response (HTTP)

```
Client                    Server
   │                         │
   │──── Request ───────────▶│
   │                         │
   │◀─── Response ───────────│
   │                         │
   │──── Request ───────────▶│
   │                         │
   │◀─── Response ───────────│
```

**Characteristics:**
- Client initiates every interaction
- One response per request
- Connection can be reused (HTTP keep-alive)

### Push (Server-Initiated)

```
Client                    Server
   │                         │
   │◀─── Push ───────────────│
   │                         │
   │◀─── Push ───────────────│
   │                         │
   │◀─── Push ───────────────│
```

**Use cases:**
- Real-time notifications
- Live updates
- Chat applications

### Bidirectional

```
Client                    Server
   │                         │
   │──── Message ───────────▶│
   │                         │
   │◀─── Message ────────────│
   │                         │
   │──── Message ───────────▶│
   │◀─── Message ────────────│
```

**Use cases:**
- Chat applications
- Collaborative editing
- Gaming

## WebSockets

WebSockets provide full-duplex communication over a single TCP connection.

### WebSocket Handshake

```
Client                              Server
   │                                   │
   │──── HTTP Upgrade Request ────────▶│
   │     GET /chat HTTP/1.1            │
   │     Upgrade: websocket            │
   │     Connection: Upgrade           │
   │     Sec-WebSocket-Key: x3JJ...    │
   │                                   │
   │◀─── HTTP 101 Switching ───────────│
   │     Upgrade: websocket            │
   │     Connection: Upgrade           │
   │     Sec-WebSocket-Accept: HS...   │
   │                                   │
   │     (WebSocket connection open)   │
   │                                   │
   │◀────── Frame ─────────────────────│
   │──────── Frame ───────────────────▶│
   │◀────── Frame ─────────────────────│
```

### WebSocket Frame Format

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - - +-------------------------------+
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - - +
:                     Payload Data continued ...                :
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
|                     Payload Data continued ...                |
+---------------------------------------------------------------+
```

### WebSocket Example (JavaScript)

**Client:**
```javascript
const ws = new WebSocket('wss://example.com/chat');

ws.onopen = () => {
  ws.send(JSON.stringify({ type: 'join', room: 'general' }));
};

ws.onmessage = (event) => {
  const message = JSON.parse(event.data);
  console.log('Received:', message);
};

ws.onclose = () => {
  console.log('Connection closed');
};

ws.onerror = (error) => {
  console.error('WebSocket error:', error);
};
```

**Server (Node.js with ws library):**
```javascript
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', (ws) => {
  ws.on('message', (message) => {
    // Broadcast to all clients
    wss.clients.forEach((client) => {
      if (client.readyState === WebSocket.OPEN) {
        client.send(message);
      }
    });
  });
});
```

### WebSocket Scaling Challenges

```
┌─────────┐     ┌──────────────┐     ┌─────────┐
│ Client1 │────▶│   Server 1   │     │ Client3 │
└─────────┘     └──────────────┘     └─────────┘
                       │                   │
┌─────────┐            │             ┌─────────┐
│ Client2 │────────────┘             │ Server 2│◀────┘
└─────────┘                          └─────────┘

Problem: Client1 and Client3 are on different servers.
         How do they communicate?
```

**Solutions:**
1. **Sticky sessions**: Route same user to same server
2. **Pub/Sub backbone**: Redis Pub/Sub, Kafka
3. **Shared state**: Store connections in Redis

```
┌─────────┐     ┌──────────┐     ┌─────────────┐     ┌──────────┐     ┌─────────┐
│ Client1 │────▶│ Server 1 │────▶│ Redis       │◀────│ Server 2 │◀────│ Client3 │
└─────────┘     └──────────┘     │ Pub/Sub     │     └──────────┘     └─────────┘
                                 └─────────────┘
```

## Long Polling

Long polling is a technique where the client makes a request and the server holds it until data is available.

### How It Works

```
Client                              Server
   │                                   │
   │──── Request ─────────────────────▶│
   │                                   │
   │     (Server holds connection      │
   │      until data available         │
   │      or timeout)                  │
   │                                   │
   │◀─── Response (with data) ─────────│
   │                                   │
   │──── New Request ─────────────────▶│
   │            ...                    │
```

### Long Polling Example

**Server (Express):**
```javascript
const waitingClients = [];

app.get('/poll', (req, res) => {
  // Add client to waiting list
  waitingClients.push(res);

  // Set timeout
  setTimeout(() => {
    const index = waitingClients.indexOf(res);
    if (index > -1) {
      waitingClients.splice(index, 1);
      res.json({ data: null, timeout: true });
    }
  }, 30000);
});

// When new data arrives
function notifyClients(data) {
  waitingClients.forEach(res => {
    res.json({ data });
  });
  waitingClients.length = 0;
}
```

**Client:**
```javascript
async function poll() {
  try {
    const response = await fetch('/poll');
    const data = await response.json();

    if (data.data) {
      handleNewData(data.data);
    }

    // Immediately start next poll
    poll();
  } catch (error) {
    // Retry after delay on error
    setTimeout(poll, 5000);
  }
}

poll();
```

### Advantages and Disadvantages

| Advantages | Disadvantages |
|------------|---------------|
| Works with HTTP/1.1 | Higher latency than WebSocket |
| No special server support | Connection overhead |
| Works through proxies | Server resource usage |
| Simple implementation | Not truly real-time |

## Server-Sent Events (SSE)

SSE is a standard for server-to-client streaming over HTTP.

### How It Works

```
Client                              Server
   │                                   │
   │──── GET /events ─────────────────▶│
   │     Accept: text/event-stream     │
   │                                   │
   │◀─── HTTP 200 ─────────────────────│
   │     Content-Type: text/event-stream
   │                                   │
   │◀─── data: {"msg": "hello"}\n\n ───│
   │                                   │
   │◀─── data: {"msg": "world"}\n\n ───│
   │                                   │
   │     (Connection stays open)       │
```

### SSE Message Format

```
event: message
id: 1
data: {"user": "John", "text": "Hello"}

event: notification
id: 2
data: {"type": "alert", "message": "New update"}

: this is a comment (ignored)

retry: 5000
data: connection will retry in 5 seconds if lost
```

### SSE Example

**Server (Node.js):**
```javascript
app.get('/events', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  // Send event every second
  const interval = setInterval(() => {
    res.write(`data: ${JSON.stringify({ time: Date.now() })}\n\n`);
  }, 1000);

  req.on('close', () => {
    clearInterval(interval);
  });
});
```

**Client:**
```javascript
const eventSource = new EventSource('/events');

eventSource.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log('Received:', data);
};

eventSource.onerror = (error) => {
  console.error('SSE error:', error);
};

// Listen for specific event types
eventSource.addEventListener('notification', (event) => {
  const data = JSON.parse(event.data);
  showNotification(data);
});
```

### SSE Features

| Feature | Description |
|---------|-------------|
| Automatic reconnection | Browser handles reconnects |
| Event IDs | Resume from last event on reconnect |
| Custom events | Named event types |
| Retry control | Server controls retry timing |

## Comparison

### When to Use Each

| Technology | Best For |
|------------|----------|
| HTTP Request/Response | Traditional API calls |
| WebSocket | Bidirectional real-time (chat, gaming) |
| Long Polling | Simple push when WebSocket not available |
| SSE | Server-to-client streaming (feeds, notifications) |

### Feature Comparison

| Feature | HTTP | Long Polling | SSE | WebSocket |
|---------|------|--------------|-----|-----------|
| Direction | Client→Server | Client→Server | Server→Client | Bidirectional |
| Connection | New each time | Held | Persistent | Persistent |
| Browser support | Universal | Universal | Good | Good |
| Binary data | Yes | Yes | No | Yes |
| Automatic reconnect | N/A | Manual | Automatic | Manual |
| HTTP/2 compatible | Yes | Yes | Yes | Separate |

### Latency Comparison

```
HTTP:           │◀──────── ~100ms+ ────────▶│
                Request → Server → Response

Long Polling:   │◀── 0-30s wait ──▶│◀─ ~50ms ─▶│
                Waiting            Response

SSE:            │◀──── ~10ms ────▶│
                Instant push

WebSocket:      │◀──── ~10ms ────▶│
                Instant bidirectional
```

### Resource Usage

| Technology | Server Memory | Connections | Bandwidth |
|------------|---------------|-------------|-----------|
| HTTP | Low (per request) | Many short | Higher |
| Long Polling | Medium | Many held | Medium |
| SSE | Medium | Persistent | Lower |
| WebSocket | Higher | Persistent | Lowest |

## Key Takeaways

1. **WebSocket for bidirectional** real-time communication
2. **SSE for server→client** streaming with simpler implementation
3. **Long polling as fallback** when WebSocket/SSE unavailable
4. **Consider infrastructure** - WebSocket needs sticky sessions or pub/sub
5. **Mobile considerations** - WebSocket drains battery, use wisely
6. **Proxy compatibility** - Some proxies don't support WebSocket

## Practice Questions

1. Design a real-time notification system. Which technology would you use?
2. How would you scale WebSocket connections across multiple servers?
3. What happens when an SSE connection drops? How does recovery work?
4. Compare the battery impact of long polling vs WebSocket on mobile.

## Further Reading

- [WebSocket Protocol RFC 6455](https://tools.ietf.org/html/rfc6455)
- [Server-Sent Events Specification](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- [The WebSocket Handbook](https://ably.com/topic/websockets)

---

Next: [Operating System Concepts](../04-os-concepts/README.md)
