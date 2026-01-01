# Blob Storage

Blob (Binary Large Object) storage is designed for storing unstructured data like images, videos, and files at massive scale.

## Table of Contents
- [What is Blob Storage?](#what-is-blob-storage)
- [Key Features](#key-features)
- [Architecture](#architecture)
- [Access Patterns](#access-patterns)
- [Providers](#providers)
- [Key Takeaways](#key-takeaways)

## What is Blob Storage?

Blob storage (object storage) stores data as objects with metadata, unlike file systems (hierarchical) or block storage (raw blocks).

```
┌────────────────────────────────────────────────────────┐
│                  Object Storage                        │
│                                                        │
│  Object = Data + Metadata + Unique ID                 │
│                                                        │
│  ┌─────────────────────────────────────────┐          │
│  │  Object: images/profile/user123.jpg     │          │
│  │  ├── Data: [binary content]             │          │
│  │  ├── Metadata:                          │          │
│  │  │   ├── Content-Type: image/jpeg       │          │
│  │  │   ├── Size: 245760                   │          │
│  │  │   └── Created: 2024-01-15T10:30:00Z  │          │
│  │  └── ETag: "abc123..."                  │          │
│  └─────────────────────────────────────────┘          │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Object vs File vs Block Storage

| Type | Access | Use Case |
|------|--------|----------|
| Object | HTTP API, key-based | Images, videos, backups |
| File | Mount, path-based | Shared files, NFS |
| Block | Raw blocks, mount | Databases, VMs |

## Key Features

### Scalability

```
┌────────────────────────────────────────────────────────┐
│              Virtually Unlimited Scale                 │
│                                                        │
│  S3: "designed to store any amount of data"          │
│                                                        │
│  Single object: up to 5 TB                            │
│  Total storage: unlimited                              │
│  Request rate: 3,500 PUT/s, 5,500 GET/s per prefix   │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Durability

```
AWS S3: 99.999999999% (11 9's) durability

What does this mean?
- Store 10,000,000 objects
- Expect to lose 1 object every 10,000 years

Achieved via:
- Replication across multiple facilities
- Checksums for data integrity
- Automatic repair
```

### Storage Classes

```
┌────────────────────────────────────────────────────────┐
│                  Storage Classes                       │
├────────────────────────────────────────────────────────┤
│                                                        │
│  Hot (Standard)                                        │
│  - Frequent access                                    │
│  - $0.023/GB/month                                    │
│                                                        │
│  Warm (Infrequent Access)                             │
│  - Monthly access                                      │
│  - $0.0125/GB/month + retrieval fee                  │
│                                                        │
│  Cold (Glacier)                                        │
│  - Yearly access                                       │
│  - $0.004/GB/month + retrieval fee                   │
│                                                        │
│  Archive (Deep Archive)                                │
│  - Rarely accessed                                     │
│  - $0.00099/GB/month                                  │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## Architecture

### Simple Upload Flow

```
┌────────────────────────────────────────────────────────┐
│                                                        │
│  Client ──(1)──▶ Your Server ──(2)──▶ S3             │
│                                                        │
│  1. Client uploads to your server                     │
│  2. Server uploads to S3                              │
│                                                        │
│  Problem: Double bandwidth, server bottleneck         │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Direct Upload (Pre-signed URL)

```
┌────────────────────────────────────────────────────────┐
│                                                        │
│  Client ──(1)──▶ Server ──(2)──▶ Generate pre-signed │
│         ◀──────────────────────  URL                  │
│                                                        │
│  Client ──(3)──▶ S3 (direct upload)                  │
│                                                        │
│  1. Client requests upload URL                        │
│  2. Server generates pre-signed URL (expires)        │
│  3. Client uploads directly to S3                    │
│                                                        │
│  Benefit: No server bandwidth, scales infinitely     │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Multipart Upload

For large files (>100MB), upload in parallel parts.

```
┌────────────────────────────────────────────────────────┐
│                  Multipart Upload                      │
│                                                        │
│  5 GB file split into 100 MB parts                   │
│                                                        │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐             │
│  │Part 1│  │Part 2│  │Part 3│  │ ...  │ (parallel)  │
│  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘             │
│     │         │         │         │                   │
│     └─────────┴─────────┴─────────┘                   │
│                    │                                   │
│                    ▼                                   │
│              Complete upload                           │
│                                                        │
│  Benefits:                                             │
│  - Parallel upload (faster)                           │
│  - Resume on failure                                   │
│  - Upload parts in any order                          │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## Access Patterns

### Serving Files

```
Option 1: Direct S3 (public bucket)
https://bucket.s3.amazonaws.com/image.jpg
- Simple but less control

Option 2: CloudFront + S3
https://cdn.example.com/image.jpg
- Caching, HTTPS, custom domain

Option 3: Pre-signed URLs (private)
https://bucket.s3.amazonaws.com/image.jpg?X-Amz-Signature=...
- Time-limited access
```

### Access Control

```python
# Bucket policy (JSON)
{
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/public/*"
}

# Pre-signed URL (Python)
url = s3.generate_presigned_url(
    'get_object',
    Params={'Bucket': 'my-bucket', 'Key': 'private/file.pdf'},
    ExpiresIn=3600  # 1 hour
)
```

### Lifecycle Policies

```yaml
# Automatically transition/delete objects
Rules:
  - ID: MoveToGlacier
    Prefix: logs/
    Transitions:
      - Days: 30
        StorageClass: GLACIER
    Expiration:
      Days: 365

  - ID: CleanupTemp
    Prefix: temp/
    Expiration:
      Days: 7
```

## Providers

### Comparison

| Provider | Service | Notes |
|----------|---------|-------|
| AWS | S3 | Industry standard, most features |
| GCP | Cloud Storage | Similar to S3, good for GCP users |
| Azure | Blob Storage | Azure integration |
| Cloudflare | R2 | No egress fees, S3 compatible |
| MinIO | Self-hosted | S3 compatible, open source |
| Backblaze | B2 | Cheap, S3 compatible |

### Pricing Comparison (per GB/month)

```
Storage:
- S3 Standard: $0.023
- GCS Standard: $0.020
- Azure Hot: $0.0184
- Cloudflare R2: $0.015
- Backblaze B2: $0.006

Egress (per GB):
- S3: $0.09
- GCS: $0.12
- Azure: $0.087
- Cloudflare R2: $0 (!)
- Backblaze B2: $0.01
```

## Design Considerations

### Naming/Partitioning

```
Bad (hot partition):
2024-01-15/image1.jpg
2024-01-15/image2.jpg
2024-01-15/image3.jpg
(All requests hit same prefix)

Good (distributed):
a1b2c3/2024-01-15/image1.jpg
d4e5f6/2024-01-15/image2.jpg
g7h8i9/2024-01-15/image3.jpg
(Requests distributed by hash prefix)
```

### Consistency

```
S3 (since Dec 2020):
- Strong read-after-write consistency
- All operations immediately consistent

Before:
- Eventual consistency for overwrites/deletes
- (This is no longer an issue)
```

## Key Takeaways

1. **Object storage for unstructured data** - images, videos, files
2. **Pre-signed URLs** for direct upload/download
3. **Multipart upload** for large files
4. **Storage classes** to optimize costs
5. **Lifecycle policies** for automatic transitions
6. **CDN in front** for serving content

## Practice Questions

1. Design an image upload service for a social media app.
2. How would you handle a 10 GB video upload?
3. Calculate storage costs for 1 PB with 50% hot, 50% cold data.
4. How do you secure private files while allowing temporary access?

## Further Reading

- [AWS S3 Documentation](https://docs.aws.amazon.com/s3/)
- [Google Cloud Storage Documentation](https://cloud.google.com/storage/docs)
- [MinIO Documentation](https://min.io/docs/)

---

Next: [Search Engines](../08-search-engines/README.md)
