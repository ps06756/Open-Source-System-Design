# Blob Storage

Blob (Binary Large Object) storage is designed for storing large unstructured data like images, videos, and files at massive scale.

## Table of Contents
1. [What is Blob Storage?](#what-is-blob-storage)
2. [Design Considerations](#design-considerations)
3. [Popular Solutions](#popular-solutions)
4. [Interview Questions](#interview-questions)

---

## What is Blob Storage?

```
┌─────────────────────────────────────────────────────────────┐
│                   Blob Storage                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Object Storage Model:                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Bucket: my-app-images                                 │ │
│  │  ├── users/123/profile.jpg                            │ │
│  │  ├── users/456/profile.jpg                            │ │
│  │  ├── posts/789/image1.png                             │ │
│  │  └── posts/789/image2.png                             │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Each object has:                                            │
│  • Key (path): users/123/profile.jpg                        │
│  • Data (blob): Binary content                              │
│  • Metadata: Content-type, size, custom headers             │
│                                                              │
│  Characteristics:                                            │
│  • Flat namespace (no real folders)                         │
│  • Immutable objects (write once, read many)               │
│  • Unlimited scale                                          │
│  • Cheap storage                                            │
│  • High durability (99.999999999% - 11 nines)              │
│                                                              │
│  Access:                                                     │
│  • REST API: GET /bucket/key                               │
│  • Pre-signed URLs for direct client access                │
│  • CDN integration for fast delivery                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Design Considerations

```
┌─────────────────────────────────────────────────────────────┐
│                Design Considerations                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Upload Strategy:                                            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Small files (<5MB): Direct upload                      │ │
│  │ Large files (>5MB): Multipart upload                   │ │
│  │                                                         │ │
│  │ Client uploads via:                                    │ │
│  │ 1. Through API server (simple but slow)               │ │
│  │ 2. Pre-signed URL (client → S3 direct)               │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Pre-signed URL Flow:                                        │
│  ┌────────┐  1.request  ┌────────┐  2.generate  ┌─────┐    │
│  │ Client │────────────▶│  API   │─────────────▶│ S3  │    │
│  └────────┘             └────────┘              └─────┘    │
│      │                       │                     ▲        │
│      │  3.pre-signed URL    │                     │        │
│      │◀──────────────────────┘                     │        │
│      │                                             │        │
│      │  4.upload directly ─────────────────────────┘        │
│                                                              │
│  Storage Classes:                                            │
│  • Hot: Frequently accessed (standard)                      │
│  • Warm: Infrequent access (cheaper)                       │
│  • Cold: Archive (very cheap, slow retrieval)              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Popular Solutions

| Solution | Best For |
|----------|----------|
| **AWS S3** | Most use cases, AWS ecosystem |
| **Google Cloud Storage** | GCP ecosystem, AI/ML workloads |
| **Azure Blob** | Azure ecosystem |
| **MinIO** | Self-hosted S3-compatible |

---

## Interview Questions

### Basic
1. What is blob storage and when would you use it?

### Intermediate
2. How would you handle large file uploads?

### Advanced
3. Design a file storage system like Dropbox.

---

[← Previous: Message Queues](../06-message-queues/README.md) | [Next: Search Engines →](../08-search-engines/README.md)
