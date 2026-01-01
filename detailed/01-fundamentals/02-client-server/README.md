# Client-Server Architecture

The client-server model is the foundation of modern web applications. Understanding different API styles and their trade-offs is crucial for system design.

## Table of Contents
1. [Client-Server Model](#client-server-model)
2. [REST APIs](#rest-apis)
3. [GraphQL](#graphql)
4. [gRPC](#grpc)
5. [Comparison and When to Use What](#comparison)
6. [API Design Best Practices](#api-design-best-practices)
7. [Interview Questions](#interview-questions)

---

## Client-Server Model

### Basic Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  Client-Server Model                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────┐         Request          ┌─────────────────┐   │
│  │         │ ─────────────────────────▶│                 │   │
│  │ Client  │                           │     Server      │   │
│  │         │ ◀─────────────────────────│                 │   │
│  └─────────┘         Response          └─────────────────┘   │
│                                                              │
│  Examples:                                                   │
│  • Web browser ↔ Web server                                 │
│  • Mobile app ↔ Backend API                                 │
│  • Microservice ↔ Microservice                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Request-Response Cycle

```
┌──────────┐                                    ┌──────────┐
│  Client  │                                    │  Server  │
└────┬─────┘                                    └────┬─────┘
     │                                               │
     │  1. Build Request                             │
     │  ├── Choose method (GET, POST, etc.)         │
     │  ├── Set headers                              │
     │  └── Add body (if needed)                     │
     │                                               │
     │  2. Send Request ────────────────────────────▶│
     │                                               │
     │                          3. Process Request   │
     │                          ├── Parse request    │
     │                          ├── Validate input   │
     │                          ├── Business logic   │
     │                          ├── Database ops     │
     │                          └── Build response   │
     │                                               │
     │  4. Receive Response ◀────────────────────────│
     │                                               │
     │  5. Handle Response                           │
     │  ├── Parse response                           │
     │  ├── Update UI                                │
     │  └── Handle errors                            │
     │                                               │
```

---

## REST APIs

### What is REST?

REST (Representational State Transfer) is an architectural style for designing networked applications. It uses HTTP methods to perform CRUD operations on resources.

### REST Principles

1. **Stateless**: Each request contains all information needed
2. **Client-Server**: Separation of concerns
3. **Cacheable**: Responses can be cached
4. **Uniform Interface**: Consistent way to interact with resources
5. **Layered System**: Client doesn't know if it's connected directly to server

### RESTful Resource Design

```
┌─────────────────────────────────────────────────────────────┐
│                   RESTful URL Design                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Resources are nouns, HTTP methods are verbs                 │
│                                                              │
│  Collection:  /users                                         │
│  Item:        /users/{id}                                    │
│  Nested:      /users/{id}/posts                              │
│  Sub-item:    /users/{id}/posts/{postId}                     │
│                                                              │
│  ┌──────────┬──────────────────┬────────────────────────┐   │
│  │ Method   │ Endpoint          │ Action                 │   │
│  ├──────────┼──────────────────┼────────────────────────┤   │
│  │ GET      │ /users            │ List all users         │   │
│  │ GET      │ /users/123        │ Get user 123           │   │
│  │ POST     │ /users            │ Create new user        │   │
│  │ PUT      │ /users/123        │ Replace user 123       │   │
│  │ PATCH    │ /users/123        │ Update user 123        │   │
│  │ DELETE   │ /users/123        │ Delete user 123        │   │
│  └──────────┴──────────────────┴────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### REST Example

```python
# Flask REST API Example

from flask import Flask, request, jsonify

app = Flask(__name__)

users = {}  # In-memory storage

# GET /users - List all users
@app.route('/users', methods=['GET'])
def list_users():
    return jsonify(list(users.values())), 200

# GET /users/<id> - Get single user
@app.route('/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    user = users.get(user_id)
    if not user:
        return jsonify({'error': 'User not found'}), 404
    return jsonify(user), 200

# POST /users - Create user
@app.route('/users', methods=['POST'])
def create_user():
    data = request.json
    user_id = len(users) + 1
    users[user_id] = {
        'id': user_id,
        'name': data['name'],
        'email': data['email']
    }
    return jsonify(users[user_id]), 201

# PUT /users/<id> - Replace user
@app.route('/users/<int:user_id>', methods=['PUT'])
def replace_user(user_id):
    if user_id not in users:
        return jsonify({'error': 'User not found'}), 404
    data = request.json
    users[user_id] = {
        'id': user_id,
        'name': data['name'],
        'email': data['email']
    }
    return jsonify(users[user_id]), 200

# DELETE /users/<id> - Delete user
@app.route('/users/<int:user_id>', methods=['DELETE'])
def delete_user(user_id):
    if user_id not in users:
        return jsonify({'error': 'User not found'}), 404
    del users[user_id]
    return '', 204
```

### REST Response Format

```json
// Success Response
{
    "data": {
        "id": 123,
        "name": "John Doe",
        "email": "john@example.com",
        "createdAt": "2024-01-15T10:30:00Z"
    },
    "meta": {
        "requestId": "abc-123"
    }
}

// Error Response
{
    "error": {
        "code": "USER_NOT_FOUND",
        "message": "User with ID 123 was not found",
        "details": {
            "field": "id",
            "value": 123
        }
    }
}

// Paginated Response
{
    "data": [...],
    "pagination": {
        "page": 1,
        "pageSize": 20,
        "totalItems": 150,
        "totalPages": 8,
        "hasNext": true,
        "hasPrev": false
    }
}
```

---

## GraphQL

### What is GraphQL?

GraphQL is a query language for APIs that allows clients to request exactly the data they need.

### GraphQL Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   GraphQL Architecture                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────┐     GraphQL Query     ┌─────────────────┐      │
│  │         │ ─────────────────────▶│                 │      │
│  │ Client  │                       │  GraphQL Server │      │
│  │         │ ◀─────────────────────│                 │      │
│  └─────────┘     Exact Response    └────────┬────────┘      │
│                                              │               │
│                                              │               │
│                      ┌───────────────────────┼───────────────┤
│                      │                       │               │
│                      ▼                       ▼               │
│               ┌─────────────┐        ┌─────────────┐        │
│               │   Database  │        │  REST APIs  │        │
│               └─────────────┘        └─────────────┘        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### GraphQL Schema

```graphql
# Schema Definition

type User {
    id: ID!
    name: String!
    email: String!
    posts: [Post!]!
    followers: [User!]!
    createdAt: DateTime!
}

type Post {
    id: ID!
    title: String!
    content: String!
    author: User!
    comments: [Comment!]!
    likes: Int!
    createdAt: DateTime!
}

type Comment {
    id: ID!
    text: String!
    author: User!
    post: Post!
}

type Query {
    user(id: ID!): User
    users(limit: Int, offset: Int): [User!]!
    post(id: ID!): Post
    posts(authorId: ID, limit: Int): [Post!]!
}

type Mutation {
    createUser(name: String!, email: String!): User!
    updateUser(id: ID!, name: String, email: String): User!
    deleteUser(id: ID!): Boolean!
    createPost(title: String!, content: String!): Post!
}

type Subscription {
    postCreated: Post!
    commentAdded(postId: ID!): Comment!
}
```

### GraphQL Queries

```graphql
# Query - Get exactly what you need

# REST would require multiple requests:
# GET /users/123
# GET /users/123/posts
# GET /posts/{id}/comments for each post

# GraphQL - Single request:
query GetUserWithPosts {
    user(id: "123") {
        name
        email
        posts {
            title
            content
            comments {
                text
                author {
                    name
                }
            }
        }
    }
}

# Response - Exact shape of query
{
    "data": {
        "user": {
            "name": "John Doe",
            "email": "john@example.com",
            "posts": [
                {
                    "title": "My First Post",
                    "content": "Hello World!",
                    "comments": [
                        {
                            "text": "Great post!",
                            "author": {
                                "name": "Jane Smith"
                            }
                        }
                    ]
                }
            ]
        }
    }
}
```

### GraphQL Mutations

```graphql
# Create a new post
mutation CreatePost {
    createPost(
        title: "GraphQL is Amazing"
        content: "Let me tell you why..."
    ) {
        id
        title
        createdAt
    }
}

# Update a user
mutation UpdateUser {
    updateUser(id: "123", name: "John Smith") {
        id
        name
        email
    }
}
```

### N+1 Problem and DataLoader

```
┌─────────────────────────────────────────────────────────────┐
│                   N+1 Problem                                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Query:                                                      │
│  {                                                           │
│      users {          # 1 query for users                   │
│          name                                                │
│          posts {      # N queries for posts (one per user)  │
│              title                                           │
│          }                                                   │
│      }                                                       │
│  }                                                           │
│                                                              │
│  Without DataLoader: 1 + N database queries                  │
│  With DataLoader: 2 database queries (batched)               │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ DataLoader batches requests:                           │ │
│  │                                                         │ │
│  │ Instead of:                                            │ │
│  │   SELECT * FROM posts WHERE user_id = 1                │ │
│  │   SELECT * FROM posts WHERE user_id = 2                │ │
│  │   SELECT * FROM posts WHERE user_id = 3                │ │
│  │                                                         │ │
│  │ It does:                                               │ │
│  │   SELECT * FROM posts WHERE user_id IN (1, 2, 3)       │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## gRPC

### What is gRPC?

gRPC is a high-performance RPC (Remote Procedure Call) framework that uses Protocol Buffers for serialization.

### gRPC Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    gRPC Architecture                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────┐           ┌─────────────────┐          │
│  │   gRPC Client   │           │   gRPC Server   │          │
│  │                 │           │                 │          │
│  │  ┌───────────┐  │           │  ┌───────────┐  │          │
│  │  │ Generated │  │  HTTP/2   │  │ Generated │  │          │
│  │  │   Stub    │──┼───────────┼──│   Stub    │  │          │
│  │  └───────────┘  │  Binary   │  └───────────┘  │          │
│  │                 │  Protobuf │                 │          │
│  └─────────────────┘           └─────────────────┘          │
│                                                              │
│  Features:                                                   │
│  • Binary protocol (efficient)                               │
│  • HTTP/2 (multiplexing, streaming)                         │
│  • Strongly typed (generated from .proto)                   │
│  • Bidirectional streaming                                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Protocol Buffers Definition

```protobuf
// user.proto

syntax = "proto3";

package user;

// Service definition
service UserService {
    // Unary RPC
    rpc GetUser(GetUserRequest) returns (User);
    rpc CreateUser(CreateUserRequest) returns (User);

    // Server streaming
    rpc ListUsers(ListUsersRequest) returns (stream User);

    // Client streaming
    rpc UploadUsers(stream User) returns (UploadResponse);

    // Bidirectional streaming
    rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

// Message definitions
message User {
    int64 id = 1;
    string name = 2;
    string email = 3;
    repeated Post posts = 4;
    google.protobuf.Timestamp created_at = 5;
}

message Post {
    int64 id = 1;
    string title = 2;
    string content = 3;
}

message GetUserRequest {
    int64 user_id = 1;
}

message CreateUserRequest {
    string name = 1;
    string email = 2;
}

message ListUsersRequest {
    int32 page_size = 1;
    string page_token = 2;
}

message UploadResponse {
    int32 count = 1;
}

message ChatMessage {
    string text = 1;
    int64 sender_id = 2;
}
```

### gRPC Streaming Types

```
┌─────────────────────────────────────────────────────────────┐
│                  gRPC Streaming Modes                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Unary RPC (Request-Response)                            │
│     Client ────[Request]────▶ Server                        │
│     Client ◀───[Response]──── Server                        │
│                                                              │
│  2. Server Streaming                                         │
│     Client ────[Request]────▶ Server                        │
│     Client ◀───[Response 1]── Server                        │
│     Client ◀───[Response 2]── Server                        │
│     Client ◀───[Response N]── Server                        │
│                                                              │
│  3. Client Streaming                                         │
│     Client ────[Request 1]──▶ Server                        │
│     Client ────[Request 2]──▶ Server                        │
│     Client ────[Request N]──▶ Server                        │
│     Client ◀───[Response]──── Server                        │
│                                                              │
│  4. Bidirectional Streaming                                  │
│     Client ◀───────────────▶ Server                         │
│     (Both send streams independently)                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### gRPC Example (Python)

```python
# Server implementation
import grpc
from concurrent import futures
import user_pb2
import user_pb2_grpc

class UserServicer(user_pb2_grpc.UserServiceServicer):
    def __init__(self):
        self.users = {}

    def GetUser(self, request, context):
        user = self.users.get(request.user_id)
        if not user:
            context.set_code(grpc.StatusCode.NOT_FOUND)
            context.set_details('User not found')
            return user_pb2.User()
        return user

    def CreateUser(self, request, context):
        user_id = len(self.users) + 1
        user = user_pb2.User(
            id=user_id,
            name=request.name,
            email=request.email
        )
        self.users[user_id] = user
        return user

    def ListUsers(self, request, context):
        # Server streaming - yield multiple responses
        for user in self.users.values():
            yield user

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    user_pb2_grpc.add_UserServiceServicer_to_server(
        UserServicer(), server
    )
    server.add_insecure_port('[::]:50051')
    server.start()
    server.wait_for_termination()

# Client usage
import grpc
import user_pb2
import user_pb2_grpc

def main():
    channel = grpc.insecure_channel('localhost:50051')
    stub = user_pb2_grpc.UserServiceStub(channel)

    # Create user
    response = stub.CreateUser(
        user_pb2.CreateUserRequest(name="John", email="john@example.com")
    )
    print(f"Created user: {response.id}")

    # Get user
    user = stub.GetUser(user_pb2.GetUserRequest(user_id=1))
    print(f"User: {user.name}")

    # List users (streaming)
    for user in stub.ListUsers(user_pb2.ListUsersRequest()):
        print(f"User: {user.name}")
```

---

## Comparison

### REST vs GraphQL vs gRPC

```
┌────────────────────────────────────────────────────────────────────────────┐
│                     API Style Comparison                                    │
├─────────────────┬─────────────────┬─────────────────┬──────────────────────┤
│ Aspect          │ REST            │ GraphQL         │ gRPC                 │
├─────────────────┼─────────────────┼─────────────────┼──────────────────────┤
│ Protocol        │ HTTP/1.1, HTTP/2│ HTTP            │ HTTP/2               │
│ Format          │ JSON (usually)  │ JSON            │ Protocol Buffers     │
│ Schema          │ OpenAPI (opt)   │ Required        │ Required (.proto)    │
│ Type Safety     │ Runtime         │ Runtime         │ Compile-time         │
│ Payload Size    │ Larger          │ Flexible        │ Smallest             │
│ Streaming       │ Limited         │ Subscriptions   │ Full support         │
│ Caching         │ HTTP caching    │ Complex         │ Manual               │
│ Learning Curve  │ Low             │ Medium          │ High                 │
│ Browser Support │ Native          │ Native          │ gRPC-Web needed      │
├─────────────────┼─────────────────┼─────────────────┼──────────────────────┤
│ Best For        │ Public APIs,    │ Mobile apps,    │ Microservices,       │
│                 │ CRUD, Simple    │ complex queries │ High performance     │
└─────────────────┴─────────────────┴─────────────────┴──────────────────────┘
```

### When to Use What

```
┌─────────────────────────────────────────────────────────────┐
│                    Decision Guide                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Use REST when:                                              │
│  ✓ Building public APIs                                     │
│  ✓ Simple CRUD operations                                   │
│  ✓ Caching is important                                     │
│  ✓ Team is familiar with REST                               │
│  ✓ Clients are varied (browsers, mobile, third-party)       │
│                                                              │
│  Use GraphQL when:                                           │
│  ✓ Multiple clients with different data needs               │
│  ✓ Nested/related data is common                            │
│  ✓ Want to reduce over-fetching                             │
│  ✓ Rapid frontend development                               │
│  ✓ Strong typing is desired                                 │
│                                                              │
│  Use gRPC when:                                              │
│  ✓ Microservices communication                              │
│  ✓ High performance is critical                             │
│  ✓ Streaming is needed                                      │
│  ✓ Both client and server are internal                      │
│  ✓ Polyglot environment (multiple languages)                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## API Design Best Practices

### Versioning

```
┌─────────────────────────────────────────────────────────────┐
│                 API Versioning Strategies                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. URL Path Versioning (Recommended)                        │
│     GET /api/v1/users                                        │
│     GET /api/v2/users                                        │
│     ✓ Clear, explicit                                        │
│     ✓ Easy to route                                          │
│     ✗ Not RESTful (resources change)                         │
│                                                              │
│  2. Header Versioning                                        │
│     GET /api/users                                           │
│     Accept: application/vnd.myapi.v1+json                    │
│     ✓ RESTful                                                │
│     ✗ Less discoverable                                      │
│     ✗ Harder to test                                         │
│                                                              │
│  3. Query Parameter                                          │
│     GET /api/users?version=1                                 │
│     ✓ Easy to implement                                      │
│     ✗ Can be forgotten                                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Pagination

```
┌─────────────────────────────────────────────────────────────┐
│                  Pagination Strategies                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Offset-based (Simple, but issues at scale)               │
│     GET /users?offset=100&limit=20                           │
│     SELECT * FROM users LIMIT 20 OFFSET 100                  │
│     Issues: Slow for large offsets, inconsistent with inserts│
│                                                              │
│  2. Cursor-based (Recommended for large datasets)            │
│     GET /users?cursor=abc123&limit=20                        │
│     SELECT * FROM users WHERE id > 'abc123' LIMIT 20         │
│     ✓ Consistent results                                     │
│     ✓ Fast at any position                                   │
│                                                              │
│  Response format:                                            │
│  {                                                           │
│      "data": [...],                                          │
│      "cursors": {                                            │
│          "next": "xyz789",                                   │
│          "prev": "abc123"                                    │
│      },                                                      │
│      "hasMore": true                                         │
│  }                                                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Error Handling

```json
// Standardized error response
{
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "Invalid request parameters",
        "details": [
            {
                "field": "email",
                "message": "Invalid email format",
                "value": "not-an-email"
            },
            {
                "field": "age",
                "message": "Must be a positive number",
                "value": -5
            }
        ],
        "requestId": "req-abc-123",
        "timestamp": "2024-01-15T10:30:00Z",
        "documentation": "https://api.example.com/docs/errors#VALIDATION_ERROR"
    }
}
```

### Rate Limiting Headers

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1673784000
Retry-After: 3600  (when rate limited)
```

---

## Interview Questions

### Basic
1. What is REST and what are its principles?
2. What are the common HTTP methods and when to use each?
3. What is the difference between PUT and PATCH?

### Intermediate
4. Compare REST, GraphQL, and gRPC. When would you use each?
5. How would you design a pagination system for an API with millions of records?
6. What is the N+1 problem in GraphQL and how do you solve it?

### Advanced
7. How would you version your API to support backward compatibility?
8. Design an API rate limiting system. What headers would you use?
9. How would you implement real-time updates in a REST API vs GraphQL?

---

[← Previous: How the Internet Works](../01-how-internet-works/README.md) | [Next: Networking Basics →](../03-networking/README.md)
