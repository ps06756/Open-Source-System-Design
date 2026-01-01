# Client-Server Architecture

The client-server model is the foundation of modern web applications. This chapter covers how clients and servers communicate through APIs.

## Table of Contents
- [Overview](#overview)
- [Request-Response Model](#request-response-model)
- [REST APIs](#rest-apis)
- [GraphQL](#graphql)
- [gRPC](#grpc)
- [Comparison](#comparison)
- [API Design Best Practices](#api-design-best-practices)
- [Key Takeaways](#key-takeaways)

## Overview

In a client-server architecture:
- **Client**: Requests services (browser, mobile app, another server)
- **Server**: Provides services (web server, API server, database)

```
┌──────────────────┐                    ┌──────────────────┐
│                  │                    │                  │
│     Clients      │                    │     Servers      │
│                  │                    │                  │
│  ┌───────────┐   │     Request        │  ┌───────────┐   │
│  │  Browser  │───┼───────────────────▶│──│    API    │   │
│  └───────────┘   │                    │  │   Server  │   │
│                  │     Response       │  └───────────┘   │
│  ┌───────────┐   │◀───────────────────┼──      │         │
│  │Mobile App │───┼───────────────────▶│──      ▼         │
│  └───────────┘   │                    │  ┌───────────┐   │
│                  │                    │  │  Database │   │
│  ┌───────────┐   │                    │  └───────────┘   │
│  │   CLI     │───┼───────────────────▶│                  │
│  └───────────┘   │                    │                  │
│                  │                    │                  │
└──────────────────┘                    └──────────────────┘
```

## Request-Response Model

The fundamental pattern of client-server communication:

```
┌────────┐                           ┌────────┐
│ Client │                           │ Server │
└───┬────┘                           └───┬────┘
    │                                    │
    │─────── HTTP Request ──────────────▶│
    │        (Method, Path, Headers,     │
    │         Body)                      │
    │                                    │
    │        ┌─────────────────┐         │
    │        │   Processing    │         │
    │        │   - Validate    │         │
    │        │   - Business    │         │
    │        │     Logic       │         │
    │        │   - Database    │         │
    │        └─────────────────┘         │
    │                                    │
    │◀────── HTTP Response ──────────────│
    │        (Status, Headers, Body)     │
    │                                    │
```

### Synchronous vs Asynchronous

**Synchronous**: Client waits for response
```
Client: Send request → Wait → Receive response → Continue
```

**Asynchronous**: Client doesn't wait
```
Client: Send request → Continue working → Handle response when ready
```

### Stateless vs Stateful

**Stateless** (REST):
- Each request contains all information needed
- Server doesn't store client session
- Easier to scale horizontally

**Stateful** (WebSocket connections):
- Server maintains client state
- More complex to scale
- Useful for real-time applications

## REST APIs

REST (Representational State Transfer) is the most common API architecture.

### REST Principles

1. **Client-Server**: Separation of concerns
2. **Stateless**: No client context stored on server
3. **Cacheable**: Responses must define cacheability
4. **Uniform Interface**: Standardized communication
5. **Layered System**: Client can't tell if connected directly to server

### Resource-Based URLs

```
# Good - Resource-oriented
GET    /users              # List users
GET    /users/123          # Get user 123
POST   /users              # Create user
PUT    /users/123          # Update user 123
DELETE /users/123          # Delete user 123

# Nested resources
GET    /users/123/orders   # Get orders for user 123
POST   /users/123/orders   # Create order for user 123

# Bad - Action-oriented (not RESTful)
GET    /getUsers
POST   /createUser
POST   /deleteUser/123
```

### HTTP Methods in REST

| Method | Action | Request Body | Response Body | Idempotent |
|--------|--------|--------------|---------------|------------|
| GET | Read | No | Yes | Yes |
| POST | Create | Yes | Yes | No |
| PUT | Replace | Yes | Optional | Yes |
| PATCH | Partial Update | Yes | Optional | No |
| DELETE | Delete | No | Optional | Yes |

### Request Example

```http
POST /api/v1/users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

{
  "name": "John Doe",
  "email": "john@example.com"
}
```

### Response Example

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /api/v1/users/12345

{
  "id": "12345",
  "name": "John Doe",
  "email": "john@example.com",
  "created_at": "2024-01-15T10:30:00Z"
}
```

### Pagination

```http
GET /api/v1/users?page=2&limit=20

Response:
{
  "data": [...],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total": 150,
    "pages": 8
  }
}
```

**Cursor-based pagination** (better for large datasets):
```http
GET /api/v1/users?cursor=abc123&limit=20

Response:
{
  "data": [...],
  "next_cursor": "def456"
}
```

### Filtering and Sorting

```http
# Filtering
GET /api/v1/users?status=active&role=admin

# Sorting
GET /api/v1/users?sort=created_at&order=desc

# Combined
GET /api/v1/users?status=active&sort=-created_at
```

## GraphQL

GraphQL is a query language for APIs developed by Facebook.

### Core Concepts

**Schema Definition**:
```graphql
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
}

type Query {
  user(id: ID!): User
  users: [User!]!
}

type Mutation {
  createUser(name: String!, email: String!): User!
}
```

### Query Example

**Request**:
```graphql
query {
  user(id: "123") {
    name
    email
    posts {
      title
    }
  }
}
```

**Response**:
```json
{
  "data": {
    "user": {
      "name": "John Doe",
      "email": "john@example.com",
      "posts": [
        { "title": "Hello World" },
        { "title": "GraphQL is great" }
      ]
    }
  }
}
```

### Mutations

```graphql
mutation {
  createUser(name: "Jane", email: "jane@example.com") {
    id
    name
  }
}
```

### Advantages of GraphQL

| Advantage | Description |
|-----------|-------------|
| No over-fetching | Get exactly what you need |
| No under-fetching | Get all related data in one request |
| Strong typing | Schema defines all types |
| Introspection | API is self-documenting |
| Single endpoint | All queries go to `/graphql` |

### Disadvantages

| Disadvantage | Description |
|--------------|-------------|
| Complexity | More complex than REST |
| Caching | HTTP caching doesn't work well |
| N+1 queries | Can cause performance issues |
| File uploads | Not straightforward |

## gRPC

gRPC is a high-performance RPC framework using Protocol Buffers.

### Protocol Buffers

```protobuf
syntax = "proto3";

message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
}

message GetUserRequest {
  int32 id = 1;
}

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc ListUsers(Empty) returns (stream User);
  rpc CreateUser(User) returns (User);
}
```

### Communication Patterns

```
1. Unary RPC (like REST)
   Client ──Request──▶ Server
   Client ◀─Response── Server

2. Server Streaming
   Client ──Request──▶ Server
   Client ◀─Response─── Server
   Client ◀─Response─── Server
   Client ◀─Response─── Server

3. Client Streaming
   Client ──Request──▶ Server
   Client ──Request──▶ Server
   Client ──Request──▶ Server
   Client ◀─Response── Server

4. Bidirectional Streaming
   Client ──Request──▶ Server
   Client ◀─Response── Server
   Client ──Request──▶ Server
   Client ◀─Response── Server
```

### Advantages of gRPC

| Advantage | Description |
|-----------|-------------|
| Performance | Binary format, 7-10x faster than JSON |
| Streaming | Native support for streaming |
| Code generation | Generate client/server code |
| Strong typing | Compile-time type checking |
| HTTP/2 | Multiplexing, compression |

### Disadvantages

| Disadvantage | Description |
|--------------|-------------|
| Browser support | Limited (needs grpc-web) |
| Human readability | Binary format not readable |
| Tooling | Less mature than REST |
| Learning curve | Protocol Buffers syntax |

## Comparison

### When to Use Each

| Use Case | Recommended |
|----------|-------------|
| Public API | REST |
| Mobile apps with variable data needs | GraphQL |
| Microservices communication | gRPC |
| Real-time streaming | gRPC or WebSocket |
| Simple CRUD operations | REST |
| Complex queries with relationships | GraphQL |
| High-performance internal services | gRPC |

### Performance Comparison

| Metric | REST (JSON) | GraphQL | gRPC |
|--------|-------------|---------|------|
| Payload size | Large | Medium | Small |
| Serialization speed | Slow | Medium | Fast |
| Latency | Higher | Medium | Lower |
| Bandwidth usage | Higher | Medium | Lower |

### Feature Comparison

| Feature | REST | GraphQL | gRPC |
|---------|------|---------|------|
| Protocol | HTTP/1.1+ | HTTP/1.1+ | HTTP/2 |
| Data format | JSON/XML | JSON | Protocol Buffers |
| Schema | Optional (OpenAPI) | Required | Required (.proto) |
| Caching | Easy (HTTP) | Complex | Complex |
| Browser support | Full | Full | Limited |
| Streaming | Workarounds | Subscriptions | Native |

## API Design Best Practices

### Versioning

```
# URL versioning (most common)
/api/v1/users
/api/v2/users

# Header versioning
Accept: application/vnd.api+json; version=1

# Query parameter
/api/users?version=1
```

### Error Handling

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid email format",
    "details": [
      {
        "field": "email",
        "message": "Must be a valid email address"
      }
    ]
  }
}
```

### Authentication

**API Keys**:
```http
Authorization: ApiKey abc123
```

**JWT (JSON Web Tokens)**:
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

**OAuth 2.0**:
```
1. Client redirects user to auth server
2. User authenticates
3. Auth server redirects back with code
4. Client exchanges code for token
5. Client uses token for API requests
```

### Rate Limiting Headers

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 998
X-RateLimit-Reset: 1640000000
```

### HATEOAS (Hypermedia)

```json
{
  "id": "123",
  "name": "John",
  "_links": {
    "self": { "href": "/users/123" },
    "orders": { "href": "/users/123/orders" },
    "update": { "href": "/users/123", "method": "PUT" }
  }
}
```

## Key Takeaways

1. **REST is the default choice** for public APIs due to simplicity and tooling
2. **GraphQL shines** when clients have varying data needs
3. **gRPC excels** in microservices with high-performance requirements
4. **Statelessness enables scaling** - avoid server-side session state
5. **Version your APIs** from the start
6. **Design for failure** - include proper error responses

## Practice Questions

1. Design a REST API for a blog platform (posts, comments, users)
2. When would you choose GraphQL over REST?
3. How would you handle API versioning for breaking changes?
4. What are the trade-offs of cursor vs offset pagination?

## Further Reading

- [REST API Design Best Practices](https://restfulapi.net/)
- [GraphQL Official Documentation](https://graphql.org/learn/)
- [gRPC Documentation](https://grpc.io/docs/)
- [API Design Patterns](https://www.manning.com/books/api-design-patterns)

---

Next: [Networking Basics](../03-networking/README.md)
