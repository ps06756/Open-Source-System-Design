# Design Dropbox

## 1. Requirements

### Functional Requirements
- Upload/download files
- Sync files across devices
- Share files/folders with others
- File versioning
- Offline access

### Non-Functional Requirements
- High reliability (never lose files)
- Efficient sync (only transfer changes)
- Low latency for sync
- Handle large files (GBs)
- Work on limited bandwidth

### Scale Estimates
- 500M users
- 100M daily active users
- Average user: 2 GB storage
- Average file: 1 MB

```
┌─────────────────────────────────────────────────────────────┐
│                   Scale Summary                             │
├─────────────────────────────────────────────────────────────┤
│  Total storage: 500M × 2 GB = 1 EB (exabyte)              │
│  Daily uploads: ~1B files                                   │
│  Sync events: 10B/day                                       │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. High-Level Design

```
┌─────────────────────────────────────────────────────────────┐
│                  High-Level Architecture                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────┐          ┌────────────────┐            │
│  │ Desktop Client │          │  Mobile Client │            │
│  │  (File Watcher)│          │                │            │
│  └───────┬────────┘          └───────┬────────┘            │
│          │                           │                      │
│          └───────────────────────────┘                      │
│                        │                                     │
│                        ▼                                     │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                  API Gateway                          │  │
│  └──────────────────────────────────────────────────────┘  │
│                        │                                     │
│     ┌──────────────────┼──────────────────┐                 │
│     ▼                  ▼                  ▼                 │
│ ┌────────────┐   ┌────────────┐   ┌────────────┐           │
│ │  Metadata  │   │   Block    │   │   Sync     │           │
│ │  Service   │   │  Service   │   │  Service   │           │
│ └─────┬──────┘   └─────┬──────┘   └─────┬──────┘           │
│       │                │                │                   │
│       ▼                ▼                ▼                   │
│ ┌────────────┐   ┌────────────┐   ┌────────────┐           │
│ │  Metadata  │   │   Block    │   │ Notification│          │
│ │    DB      │   │  Storage   │   │   Service  │           │
│ │  (MySQL)   │   │   (S3)     │   │ (WebSocket)│           │
│ └────────────┘   └────────────┘   └────────────┘           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. File Chunking & Deduplication

```
┌─────────────────────────────────────────────────────────────┐
│             File Chunking & Deduplication                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Why chunk files?                                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  1. Resume interrupted uploads                         │ │
│  │  2. Only upload changed chunks                         │ │
│  │  3. Deduplicate across users                          │ │
│  │  4. Parallel upload/download                          │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Chunking Strategy:                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  File: 100 MB document.pdf                             │ │
│  │                                                         │ │
│  │  ┌───────┬───────┬───────┬───────┬─────┐              │ │
│  │  │Chunk 1│Chunk 2│Chunk 3│Chunk 4│...  │              │ │
│  │  │ 4 MB  │ 4 MB  │ 4 MB  │ 4 MB  │     │              │ │
│  │  └───┬───┴───┬───┴───┬───┴───┬───┴─────┘              │ │
│  │      │       │       │       │                         │ │
│  │      ▼       ▼       ▼       ▼                         │ │
│  │   SHA-256  SHA-256 SHA-256 SHA-256                     │ │
│  │   (hash)   (hash)  (hash)  (hash)                      │ │
│  │                                                         │ │
│  │  Chunk size: 4 MB (configurable)                       │ │
│  │  Hash = unique identifier for chunk                    │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Deduplication:                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  User A uploads: photo.jpg → chunk hash: abc123       │ │
│  │  User B uploads: same photo.jpg                        │ │
│  │                                                         │ │
│  │  1. Client computes hash: abc123                       │ │
│  │  2. Client asks server: "Do you have abc123?"         │ │
│  │  3. Server: "Yes, already exists"                     │ │
│  │  4. No upload needed! Just link to existing chunk     │ │
│  │                                                         │ │
│  │  Deduplication rate: 30-50% storage savings           │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Content-Defined Chunking (CDC):                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Fixed-size chunks problem:                            │ │
│  │  Insert 1 byte at start → ALL chunks different!       │ │
│  │                                                         │ │
│  │  CDC solution:                                          │ │
│  │  Use rolling hash to find chunk boundaries            │ │
│  │  When hash matches pattern → chunk boundary           │ │
│  │                                                         │ │
│  │  Insert 1 byte → Only nearby chunks affected          │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Sync Protocol

```
┌─────────────────────────────────────────────────────────────┐
│                    Sync Protocol                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Upload Flow:                                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. File changed locally                               │ │
│  │     ┌─────────────────────────────────────────────┐   │ │
│  │     │ File Watcher detects: document.pdf modified │   │ │
│  │     └─────────────────────────────────────────────┘   │ │
│  │                                                         │ │
│  │  2. Compute chunk hashes                               │ │
│  │     ┌─────────────────────────────────────────────┐   │ │
│  │     │ [hash1, hash2, hash3, hash4, ...]           │   │ │
│  │     └─────────────────────────────────────────────┘   │ │
│  │                                                         │ │
│  │  3. Compare with server's chunk list                  │ │
│  │     ┌─────────────────────────────────────────────┐   │ │
│  │     │ Server has: [hash1, hash2]                   │   │ │
│  │     │ Need to upload: [hash3, hash4]               │   │ │
│  │     └─────────────────────────────────────────────┘   │ │
│  │                                                         │ │
│  │  4. Upload only new/changed chunks                    │ │
│  │     ┌─────────────────────────────────────────────┐   │ │
│  │     │ POST /chunks/hash3 → S3                      │   │ │
│  │     │ POST /chunks/hash4 → S3                      │   │ │
│  │     └─────────────────────────────────────────────┘   │ │
│  │                                                         │ │
│  │  5. Update file metadata                               │ │
│  │     ┌─────────────────────────────────────────────┐   │ │
│  │     │ PUT /files/document.pdf                      │   │ │
│  │     │ chunks: [hash1, hash2, hash3, hash4]        │   │ │
│  │     │ version: 5                                   │   │ │
│  │     └─────────────────────────────────────────────┘   │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Download/Sync Flow:                                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Receive notification: "document.pdf changed"      │ │
│  │                                                         │ │
│  │  2. Get new file metadata                              │ │
│  │     { chunks: [hash1, hash2, hash3, hash4] }          │ │
│  │                                                         │ │
│  │  3. Compare with local chunks                         │ │
│  │     Local: [hash1, hash2, hashOLD3, hashOLD4]        │ │
│  │     Need: [hash3, hash4]                              │ │
│  │                                                         │ │
│  │  4. Download only missing chunks                      │ │
│  │     GET /chunks/hash3                                 │ │
│  │     GET /chunks/hash4                                 │ │
│  │                                                         │ │
│  │  5. Reconstruct file locally                          │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Notification Service

```
┌─────────────────────────────────────────────────────────────┐
│                 Notification Service                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  How clients know when to sync:                             │
│                                                              │
│  Option 1: Polling (simple but inefficient)                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Client: "Any changes?" (every 30 seconds)            │ │
│  │  Server: "Yes/No"                                      │ │
│  │                                                         │ │
│  │  ✗ High latency (up to 30 seconds)                    │ │
│  │  ✗ Wasted requests when no changes                    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Option 2: Long Polling                                     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Client: "Any changes?" (wait up to 60 seconds)       │ │
│  │  Server: Holds connection until change or timeout      │ │
│  │                                                         │ │
│  │  ✓ Faster than polling                                │ │
│  │  ✗ Still not real-time                                │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Option 3: WebSocket (recommended)                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Client ◀──────────────────▶ Notification Server       │ │
│  │         Persistent connection                          │ │
│  │                                                         │ │
│  │  Server pushes: { "event": "file_changed",            │ │
│  │                   "path": "/docs/report.pdf",         │ │
│  │                   "version": 5 }                       │ │
│  │                                                         │ │
│  │  ✓ Real-time                                          │ │
│  │  ✓ Efficient                                          │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Implementation:                                            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ┌────────┐    ┌─────────────┐    ┌─────────────┐     │ │
│  │  │Client 1│◀──▶│ Notification│◀───│   Redis     │     │ │
│  │  │        │    │   Server    │    │   Pub/Sub   │     │ │
│  │  └────────┘    └─────────────┘    └──────┬──────┘     │ │
│  │                                          │             │ │
│  │  ┌────────┐    ┌─────────────┐           │             │ │
│  │  │Client 2│◀──▶│ Notification│◀──────────┘             │ │
│  │  │        │    │   Server    │                         │ │
│  │  └────────┘    └─────────────┘                         │ │
│  │                                                         │ │
│  │  On file change:                                       │ │
│  │  1. Metadata Service → Redis.PUBLISH(user_channel)    │ │
│  │  2. Redis → All Notification Servers                  │ │
│  │  3. Notification Servers → Connected clients          │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Data Model

```
┌─────────────────────────────────────────────────────────────┐
│                     Data Model                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Files Metadata (MySQL)                                     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ file_id         │ BIGINT (PK)                           │ │
│  │ user_id         │ BIGINT (FK)                           │ │
│  │ path            │ VARCHAR(4096)                         │ │
│  │ name            │ VARCHAR(255)                          │ │
│  │ size_bytes      │ BIGINT                                │ │
│  │ checksum        │ CHAR(64) (SHA-256)                    │ │
│  │ latest_version  │ INT                                   │ │
│  │ is_folder       │ BOOLEAN                               │ │
│  │ parent_id       │ BIGINT (FK, nullable)                 │ │
│  │ created_at      │ TIMESTAMP                             │ │
│  │ modified_at     │ TIMESTAMP                             │ │
│  │ deleted_at      │ TIMESTAMP (soft delete)               │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  File Versions                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ version_id      │ BIGINT (PK)                           │ │
│  │ file_id         │ BIGINT (FK)                           │ │
│  │ version_number  │ INT                                   │ │
│  │ size_bytes      │ BIGINT                                │ │
│  │ checksum        │ CHAR(64)                              │ │
│  │ created_at      │ TIMESTAMP                             │ │
│  │ created_by      │ BIGINT                                │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  File Chunks (version → chunks mapping)                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ version_id      │ BIGINT (FK)                           │ │
│  │ chunk_index     │ INT                                   │ │
│  │ chunk_hash      │ CHAR(64) (SHA-256)                    │ │
│  │ PRIMARY KEY (version_id, chunk_index)                  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Chunks (deduplicated storage)                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ chunk_hash      │ CHAR(64) (PK)                         │ │
│  │ size_bytes      │ INT                                   │ │
│  │ s3_path         │ VARCHAR(500)                          │ │
│  │ reference_count │ INT (for garbage collection)          │ │
│  │ created_at      │ TIMESTAMP                             │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Conflict Resolution

```
┌─────────────────────────────────────────────────────────────┐
│                  Conflict Resolution                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Scenario: Same file edited on two offline devices         │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                                                         ││
│  │  Device A (offline)    Server    Device B (offline)    ││
│  │  Edit: Add line 1      v1        Edit: Add line 2      ││
│  │         │                              │                ││
│  │         │                              │                ││
│  │  (goes online)                  (goes online)          ││
│  │         │                              │                ││
│  │         ▼                              │                ││
│  │  Upload v2 ───────────▶ v2             │                ││
│  │         │                              ▼                ││
│  │         │              Upload fails! Conflict detected ││
│  │                                                         ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  Resolution Strategies:                                     │
│                                                              │
│  1. Last Writer Wins (simple, data loss risk)              │
│     Not recommended for important files                    │
│                                                              │
│  2. Create Conflict Copy (Dropbox approach)                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Original: document.pdf                                 │ │
│  │  Conflict: document (John's conflicted copy).pdf       │ │
│  │                                                         │ │
│  │  User manually merges                                  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  3. Operational Transform (Google Docs approach)           │
│     Real-time collaboration, complex to implement          │
│     Better for text documents                              │
│                                                              │
│  Detection:                                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Upload includes: "based_on_version: 1"                │ │
│  │  Server current: "version: 2"                          │ │
│  │                                                         │ │
│  │  If based_on_version != current_version - 1            │ │
│  │     → Conflict detected                                │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. Summary

| Component | Choice | Reason |
|-----------|--------|--------|
| **Block Storage** | S3 | Durable, scalable |
| **Metadata** | MySQL (sharded) | ACID, relationships |
| **Sync Notifications** | WebSocket + Redis Pub/Sub | Real-time |
| **Deduplication** | Content-addressed chunks | Storage efficiency |
| **Conflict** | Create conflict copy | Simple, no data loss |

---

[Back to System Designs](../README.md)
