# Networking Basics

Real-time communication patterns are essential for modern applications. This chapter covers different approaches to bidirectional and real-time communication.

## Table of Contents
1. [Sockets](#sockets)
2. [WebSockets](#websockets)
3. [Long Polling](#long-polling)
4. [Server-Sent Events (SSE)](#server-sent-events)
5. [Comparison](#comparison)
6. [Interview Questions](#interview-questions)

---

## Sockets

### What is a Socket?

A socket is an endpoint for communication between two machines. It's the fundamental building block of network programming.

```
┌─────────────────────────────────────────────────────────────┐
│                     Socket Communication                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────┐          ┌──────────────────┐         │
│  │     Client       │          │     Server       │         │
│  │                  │          │                  │         │
│  │  ┌────────────┐  │          │  ┌────────────┐  │         │
│  │  │   Socket   │──┼──────────┼──│   Socket   │  │         │
│  │  │ 192.168.1.5│  │   TCP    │  │ 93.184.216 │  │         │
│  │  │   :54321   │  │connection│  │    :80     │  │         │
│  │  └────────────┘  │          │  └────────────┘  │         │
│  │                  │          │                  │         │
│  └──────────────────┘          └──────────────────┘         │
│                                                              │
│  Socket = IP Address + Port Number                           │
│  Connection = Client Socket + Server Socket                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Socket Programming Example (Python)

```python
# TCP Server
import socket

def tcp_server():
    # Create socket
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

    # Allow address reuse
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

    # Bind to address and port
    server_socket.bind(('0.0.0.0', 8080))

    # Listen for connections (backlog of 5)
    server_socket.listen(5)
    print("Server listening on port 8080...")

    while True:
        # Accept connection (blocking)
        client_socket, address = server_socket.accept()
        print(f"Connection from {address}")

        # Receive data
        data = client_socket.recv(1024)
        print(f"Received: {data.decode()}")

        # Send response
        client_socket.send(b"Hello from server!")

        # Close connection
        client_socket.close()

# TCP Client
def tcp_client():
    # Create socket
    client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

    # Connect to server
    client_socket.connect(('localhost', 8080))

    # Send data
    client_socket.send(b"Hello from client!")

    # Receive response
    response = client_socket.recv(1024)
    print(f"Server says: {response.decode()}")

    # Close connection
    client_socket.close()
```

### Socket States

```
┌─────────────────────────────────────────────────────────────┐
│                   TCP Socket State Diagram                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│                    ┌─────────┐                               │
│                    │ CLOSED  │                               │
│                    └────┬────┘                               │
│           ┌─────────────┼─────────────┐                      │
│           │             │             │                      │
│           ▼             ▼             ▼                      │
│     ┌──────────┐  ┌──────────┐  ┌──────────┐                │
│     │  LISTEN  │  │ SYN_SENT │  │  (bind)  │                │
│     │ (server) │  │ (client) │  └──────────┘                │
│     └────┬─────┘  └────┬─────┘                              │
│          │             │                                     │
│          ▼             ▼                                     │
│     ┌──────────┐  ┌──────────┐                              │
│     │ SYN_RCVD │  │ESTABLISHED│◀─────────────────┐          │
│     └────┬─────┘  └────┬─────┘                   │          │
│          │             │                          │          │
│          └──────┬──────┘                          │          │
│                 │                                 │          │
│                 ▼                                 │          │
│           ┌──────────┐        ┌──────────┐       │          │
│           │FIN_WAIT_1│───────▶│FIN_WAIT_2│       │          │
│           └────┬─────┘        └────┬─────┘       │          │
│                │                   │              │          │
│                ▼                   ▼              │          │
│           ┌──────────┐        ┌──────────┐       │          │
│           │ CLOSING  │        │TIME_WAIT │───────┘          │
│           └────┬─────┘        └────┬─────┘                  │
│                │                   │                         │
│                └───────┬───────────┘                         │
│                        ▼                                     │
│                   ┌─────────┐                                │
│                   │ CLOSED  │                                │
│                   └─────────┘                                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## WebSockets

### What are WebSockets?

WebSockets provide full-duplex, bidirectional communication over a single TCP connection. Unlike HTTP, the connection stays open, allowing real-time data exchange.

```
┌─────────────────────────────────────────────────────────────┐
│                HTTP vs WebSocket                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  HTTP (Request-Response):                                    │
│  Client ──[Request]──▶ Server                               │
│  Client ◀──[Response]── Server                               │
│  (Connection closes after each request)                      │
│                                                              │
│  WebSocket (Full Duplex):                                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Client ◀═══════════════════════════════════▶ Server   │ │
│  │          (Persistent bidirectional connection)          │ │
│  │                                                         │ │
│  │  Either side can send messages at any time              │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### WebSocket Handshake

```
┌─────────────────────────────────────────────────────────────┐
│               WebSocket Handshake (Upgrade)                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Client sends HTTP upgrade request:                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ GET /chat HTTP/1.1                                      │ │
│  │ Host: server.example.com                                │ │
│  │ Upgrade: websocket                                      │ │
│  │ Connection: Upgrade                                     │ │
│  │ Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==            │ │
│  │ Sec-WebSocket-Version: 13                               │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  2. Server responds with 101 Switching Protocols:            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ HTTP/1.1 101 Switching Protocols                        │ │
│  │ Upgrade: websocket                                      │ │
│  │ Connection: Upgrade                                     │ │
│  │ Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=     │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  3. Connection is now WebSocket (binary frames)              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### WebSocket Example (JavaScript + Python)

```javascript
// Client (Browser)
const socket = new WebSocket('wss://example.com/chat');

// Connection opened
socket.onopen = function(event) {
    console.log('Connected to server');
    socket.send(JSON.stringify({
        type: 'join',
        room: 'general'
    }));
};

// Receive message
socket.onmessage = function(event) {
    const message = JSON.parse(event.data);
    console.log('Received:', message);
};

// Connection closed
socket.onclose = function(event) {
    console.log('Disconnected');
};

// Error handling
socket.onerror = function(error) {
    console.error('WebSocket error:', error);
};

// Send message
function sendMessage(text) {
    socket.send(JSON.stringify({
        type: 'message',
        text: text,
        timestamp: Date.now()
    }));
}
```

```python
# Server (Python with websockets library)
import asyncio
import websockets
import json

connected_clients = set()

async def handler(websocket, path):
    # Register client
    connected_clients.add(websocket)

    try:
        async for message in websocket:
            data = json.loads(message)

            if data['type'] == 'message':
                # Broadcast to all connected clients
                await broadcast(json.dumps({
                    'type': 'message',
                    'text': data['text'],
                    'timestamp': data['timestamp']
                }))

    finally:
        # Unregister client
        connected_clients.remove(websocket)

async def broadcast(message):
    if connected_clients:
        await asyncio.gather(
            *[client.send(message) for client in connected_clients]
        )

async def main():
    async with websockets.serve(handler, "0.0.0.0", 8765):
        await asyncio.Future()  # Run forever

asyncio.run(main())
```

### WebSocket Frame Structure

```
┌─────────────────────────────────────────────────────────────┐
│                 WebSocket Frame Format                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   0                   1                   2                  │
│   0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3           │
│  +-+-+-+-+-------+-+-------------+-------------------------------+
│  |F|R|R|R| opcode|M| Payload len |    Extended payload length    │
│  |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           │
│  |N|V|V|V|       |S|             |   (if payload len==126/127)   │
│  | |1|2|3|       |K|             |                               │
│  +-+-+-+-+-------+-+-------------+-------------------------------+
│  |     Extended payload length continued, if payload len == 127  │
│  +-------------------------------+-------------------------------+
│  |                               |Masking-key, if MASK set to 1  │
│  +-------------------------------+-------------------------------+
│  | Masking-key (continued)       |          Payload Data         │
│  +-------------------------------+-------------------------------+
│  |                     Payload Data continued ...                 │
│  +---------------------------------------------------------------+
│                                                              │
│  Opcodes:                                                    │
│  0x0 = Continuation                                          │
│  0x1 = Text frame                                            │
│  0x2 = Binary frame                                          │
│  0x8 = Close                                                 │
│  0x9 = Ping                                                  │
│  0xA = Pong                                                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Long Polling

### How Long Polling Works

Long polling is a technique where the client makes a request and the server holds it open until new data is available.

```
┌─────────────────────────────────────────────────────────────┐
│                     Long Polling                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Client                                Server                │
│    │                                      │                  │
│    │  1. Request (GET /updates)           │                  │
│    │─────────────────────────────────────▶│                  │
│    │                                      │                  │
│    │         (Server holds connection     │                  │
│    │          until data available        │                  │
│    │          or timeout ~30 seconds)     │                  │
│    │                                      │                  │
│    │  2. Response (with new data)         │                  │
│    │◀─────────────────────────────────────│                  │
│    │                                      │                  │
│    │  3. Immediately make new request     │                  │
│    │─────────────────────────────────────▶│                  │
│    │                                      │                  │
│    │         (repeat...)                  │                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Long Polling Example

```javascript
// Client
async function longPoll() {
    while (true) {
        try {
            const response = await fetch('/api/updates', {
                method: 'GET',
                headers: {
                    'X-Last-Event-Id': lastEventId
                }
            });

            if (response.status === 200) {
                const data = await response.json();
                handleUpdates(data.events);
                lastEventId = data.lastEventId;
            } else if (response.status === 204) {
                // No new data, timeout occurred
                console.log('No updates, reconnecting...');
            }
        } catch (error) {
            console.error('Polling error:', error);
            // Wait before retrying on error
            await new Promise(r => setTimeout(r, 5000));
        }
    }
}

let lastEventId = 0;
longPoll();
```

```python
# Server (Flask)
from flask import Flask, jsonify, request
import time
import threading

app = Flask(__name__)
events = []
events_lock = threading.Lock()

@app.route('/api/updates')
def get_updates():
    last_id = int(request.headers.get('X-Last-Event-Id', 0))
    timeout = 30  # seconds
    start_time = time.time()

    while time.time() - start_time < timeout:
        with events_lock:
            new_events = [e for e in events if e['id'] > last_id]
            if new_events:
                return jsonify({
                    'events': new_events,
                    'lastEventId': new_events[-1]['id']
                })

        time.sleep(0.5)  # Check every 500ms

    # Timeout - no new events
    return '', 204
```

---

## Server-Sent Events

### What are Server-Sent Events?

SSE is a standard for servers to push updates to clients over HTTP. It's simpler than WebSockets but only allows server-to-client communication.

```
┌─────────────────────────────────────────────────────────────┐
│                 Server-Sent Events (SSE)                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────┐       HTTP connection (kept open)  ┌────────┐   │
│  │        │ ◀═══════════════════════════════════│        │   │
│  │ Client │       Server pushes events          │ Server │   │
│  │        │ ◀───── event: message ──────────────│        │   │
│  │        │ ◀───── event: update ───────────────│        │   │
│  │        │ ◀───── event: notification ─────────│        │   │
│  └────────┘                                     └────────┘   │
│                                                              │
│  Characteristics:                                            │
│  • Unidirectional (server → client only)                    │
│  • Uses standard HTTP                                        │
│  • Auto-reconnection built-in                               │
│  • Text-based (UTF-8)                                       │
│  • Simpler than WebSockets                                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### SSE Event Format

```
┌─────────────────────────────────────────────────────────────┐
│                    SSE Event Format                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  HTTP Response:                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ HTTP/1.1 200 OK                                        │ │
│  │ Content-Type: text/event-stream                        │ │
│  │ Cache-Control: no-cache                                │ │
│  │ Connection: keep-alive                                 │ │
│  │                                                        │ │
│  │ event: message                                         │ │
│  │ id: 1                                                  │ │
│  │ data: {"user": "John", "text": "Hello!"}              │ │
│  │                                                        │ │
│  │ event: notification                                    │ │
│  │ id: 2                                                  │ │
│  │ retry: 5000                                            │ │
│  │ data: {"type": "like", "count": 5}                    │ │
│  │                                                        │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Fields:                                                     │
│  • event: Event type (custom name)                          │
│  • id: Event ID (for resumption)                            │
│  • data: Event payload (can span multiple lines)            │
│  • retry: Reconnection time in milliseconds                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### SSE Example

```javascript
// Client
const eventSource = new EventSource('/api/stream');

// Listen for all messages
eventSource.onmessage = function(event) {
    console.log('Message:', event.data);
};

// Listen for specific event types
eventSource.addEventListener('notification', function(event) {
    const data = JSON.parse(event.data);
    showNotification(data);
});

eventSource.addEventListener('update', function(event) {
    const data = JSON.parse(event.data);
    updateUI(data);
});

// Error handling
eventSource.onerror = function(event) {
    if (eventSource.readyState === EventSource.CLOSED) {
        console.log('Connection closed');
    } else {
        console.error('Error occurred');
    }
};

// Close connection when done
// eventSource.close();
```

```python
# Server (Flask)
from flask import Flask, Response
import time
import json

app = Flask(__name__)

@app.route('/api/stream')
def stream():
    def generate():
        event_id = 0
        while True:
            # Get new data (from database, queue, etc.)
            data = get_new_data()

            if data:
                event_id += 1
                yield f"event: message\n"
                yield f"id: {event_id}\n"
                yield f"data: {json.dumps(data)}\n\n"

            time.sleep(1)  # Check for updates every second

    return Response(
        generate(),
        mimetype='text/event-stream',
        headers={
            'Cache-Control': 'no-cache',
            'Connection': 'keep-alive',
            'X-Accel-Buffering': 'no'  # Disable nginx buffering
        }
    )
```

---

## Comparison

### Feature Comparison

```
┌──────────────────────────────────────────────────────────────────────────┐
│              Real-Time Communication Comparison                           │
├─────────────────┬─────────────┬─────────────┬─────────────┬──────────────┤
│ Feature         │ Polling     │Long Polling │ SSE         │ WebSocket    │
├─────────────────┼─────────────┼─────────────┼─────────────┼──────────────┤
│ Direction       │ Client→Srv  │ Client→Srv  │ Srv→Client  │ Bidirectional│
│ Connection      │ New each    │ Held open   │ Persistent  │ Persistent   │
│ Protocol        │ HTTP        │ HTTP        │ HTTP        │ WS (TCP)     │
│ Browser Support │ All         │ All         │ Most        │ Most         │
│ Latency         │ High        │ Medium      │ Low         │ Very Low     │
│ Server Load     │ High        │ Medium      │ Low         │ Low          │
│ Complexity      │ Simple      │ Simple      │ Simple      │ Complex      │
│ Reconnection    │ Manual      │ Manual      │ Automatic   │ Manual       │
│ Binary Data     │ Yes         │ Yes         │ No          │ Yes          │
│ Proxy Friendly  │ Yes         │ Maybe       │ Maybe       │ Often No     │
├─────────────────┼─────────────┼─────────────┼─────────────┼──────────────┤
│ Use Case        │ Infrequent  │ Near        │ Live feeds, │ Chat, Games, │
│                 │ updates     │ real-time   │ Notifications│ Collaboration│
└─────────────────┴─────────────┴─────────────┴─────────────┴──────────────┘
```

### Decision Guide

```
┌─────────────────────────────────────────────────────────────┐
│                   When to Use What                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Regular Polling:                                            │
│  • Simple requirements                                       │
│  • Updates can be delayed (minutes)                         │
│  • Low traffic                                               │
│  Example: Email check, dashboard refresh                     │
│                                                              │
│  Long Polling:                                               │
│  • Need quick updates without WebSocket complexity          │
│  • Works through all proxies/firewalls                      │
│  • Moderate traffic                                          │
│  Example: Slack (fallback), old chat systems                │
│                                                              │
│  Server-Sent Events:                                         │
│  • Server-to-client updates only                            │
│  • Live feeds, notifications                                 │
│  • Need automatic reconnection                              │
│  Example: Stock tickers, social feeds, live scores          │
│                                                              │
│  WebSockets:                                                 │
│  • Bidirectional real-time communication                    │
│  • Low latency critical                                     │
│  • High-frequency updates                                    │
│  Example: Chat, multiplayer games, collaborative editing     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

### Basic
1. What is a socket? Explain the client-server socket communication.
2. What is the difference between TCP and UDP sockets?
3. How do WebSockets differ from regular HTTP connections?

### Intermediate
4. Explain the WebSocket handshake process.
5. Compare long polling, SSE, and WebSockets. When would you use each?
6. How does SSE handle reconnection?

### Advanced
7. How would you scale a WebSocket-based chat application to millions of users?
8. What are the challenges of WebSockets with load balancers and proxies?
9. Design a real-time notification system. What communication method would you use and why?

---

[← Previous: Client-Server Architecture](../02-client-server/README.md) | [Next: Operating System Concepts →](../04-os-concepts/README.md)
