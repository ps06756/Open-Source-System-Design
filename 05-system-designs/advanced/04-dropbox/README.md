# Design Dropbox

Design a cloud file storage and synchronization service like Dropbox.

## 1. Requirements

### Functional Requirements
- Upload/download files
- Automatic sync across devices
- File versioning
- Share files/folders
- Offline access
- Conflict resolution

### Non-Functional Requirements
- High reliability (no data loss)
- Efficient bandwidth usage (delta sync)
- Low latency for small files
- Scale: 500M users, 1B files uploaded/day

### Capacity Estimation

```
Users: 500M total, 100M DAU
Files: 1B uploads/day
Average file size: 1 MB
Storage: 1B × 1 MB = 1 PB/day new data

Metadata: 100 files/user × 500M = 50B file records
With versions: 50B × 5 versions = 250B records
```

## 2. High-Level Design

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌────────────┐                             ┌────────────┐    │
│  │   Client   │                             │   Client   │    │
│  │ (Desktop)  │                             │  (Mobile)  │    │
│  └─────┬──────┘                             └──────┬─────┘    │
│        │                                           │          │
│        └─────────────────┬─────────────────────────┘          │
│                          ▼                                     │
│                   ┌─────────────┐                             │
│                   │   API GW    │                             │
│                   └──────┬──────┘                             │
│                          │                                     │
│          ┌───────────────┼───────────────┐                    │
│          │               │               │                    │
│          ▼               ▼               ▼                    │
│   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐            │
│   │  Metadata   │ │   Block     │ │   Sync      │            │
│   │  Service    │ │   Service   │ │   Service   │            │
│   └──────┬──────┘ └──────┬──────┘ └──────┬──────┘            │
│          │               │               │                    │
│          ▼               ▼               ▼                    │
│   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐            │
│   │ PostgreSQL  │ │     S3      │ │   Redis     │            │
│   │ (Metadata)  │ │  (Blocks)   │ │   (State)   │            │
│   └─────────────┘ └─────────────┘ └─────────────┘            │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 3. Block-Level Sync

```
┌────────────────────────────────────────────────────────────────┐
│                    Block-Level Sync                            │
│                                                                │
│  File split into 4 MB blocks:                                 │
│                                                                │
│  ┌────────────────────────────────────────────────────────┐   │
│  │  Large File (16 MB)                                     │   │
│  │  ┌────────┬────────┬────────┬────────┐                 │   │
│  │  │Block 1 │Block 2 │Block 3 │Block 4 │                 │   │
│  │  │ 4 MB   │ 4 MB   │ 4 MB   │ 4 MB   │                 │   │
│  │  └────────┴────────┴────────┴────────┘                 │   │
│  │                                                         │   │
│  │  Each block has SHA256 hash                            │   │
│  │  Block 1: abc123...                                    │   │
│  │  Block 2: def456...                                    │   │
│  │  Block 3: ghi789...                                    │   │
│  │  Block 4: jkl012...                                    │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                                │
│  Benefits:                                                     │
│  1. Delta sync: Only upload changed blocks                   │
│  2. Deduplication: Same block stored once                    │
│  3. Resume: Interrupted upload continues from last block     │
│  4. Parallel: Upload multiple blocks simultaneously          │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Client-Side Chunking

```python
class FileChunker:
    BLOCK_SIZE = 4 * 1024 * 1024  # 4 MB

    def chunk_file(self, file_path):
        """Split file into blocks with hashes"""
        blocks = []

        with open(file_path, 'rb') as f:
            index = 0
            while True:
                data = f.read(self.BLOCK_SIZE)
                if not data:
                    break

                block_hash = hashlib.sha256(data).hexdigest()
                blocks.append({
                    'index': index,
                    'hash': block_hash,
                    'size': len(data),
                    'data': data
                })
                index += 1

        return blocks

    def get_delta(self, local_blocks, remote_blocks):
        """Find blocks that need uploading"""
        remote_hashes = {b['hash'] for b in remote_blocks}

        to_upload = []
        for block in local_blocks:
            if block['hash'] not in remote_hashes:
                to_upload.append(block)

        return to_upload

class SyncClient:
    async def upload_file(self, file_path, file_id):
        """Upload file using delta sync"""
        # 1. Chunk file locally
        local_blocks = self.chunker.chunk_file(file_path)

        # 2. Get remote file blocks
        remote_blocks = await self.api.get_file_blocks(file_id)

        # 3. Calculate delta
        to_upload = self.chunker.get_delta(local_blocks, remote_blocks)

        # 4. Upload only changed blocks (parallel)
        tasks = []
        for block in to_upload:
            task = self.upload_block(block)
            tasks.append(task)

        await asyncio.gather(*tasks)

        # 5. Commit new file version
        block_list = [{'index': b['index'], 'hash': b['hash']}
                     for b in local_blocks]
        await self.api.commit_file(file_id, block_list)
```

## 4. Metadata Service

```sql
-- Files and folders
CREATE TABLE files (
    file_id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    parent_folder_id UUID,
    name VARCHAR(255),
    is_folder BOOLEAN DEFAULT false,
    size BIGINT,
    content_hash VARCHAR(64),  -- Hash of all block hashes
    created_at TIMESTAMP,
    modified_at TIMESTAMP,
    deleted_at TIMESTAMP,  -- Soft delete
    INDEX idx_user_parent (user_id, parent_folder_id)
);

-- File versions
CREATE TABLE file_versions (
    version_id UUID PRIMARY KEY,
    file_id UUID NOT NULL,
    version_number INT,
    size BIGINT,
    content_hash VARCHAR(64),
    block_list JSONB,  -- [{index: 0, hash: "abc..."}, ...]
    created_at TIMESTAMP,
    created_by UUID,
    FOREIGN KEY (file_id) REFERENCES files(file_id)
);

-- Blocks (for deduplication)
CREATE TABLE blocks (
    block_hash VARCHAR(64) PRIMARY KEY,
    size INT,
    storage_path TEXT,  -- S3 path
    reference_count INT DEFAULT 1,
    created_at TIMESTAMP
);

-- Sharing
CREATE TABLE shares (
    share_id UUID PRIMARY KEY,
    file_id UUID NOT NULL,
    shared_by UUID,
    shared_with UUID,  -- NULL for link sharing
    permission VARCHAR(20),  -- view, edit
    share_link VARCHAR(64),  -- For public links
    expires_at TIMESTAMP,
    created_at TIMESTAMP
);
```

## 5. Sync Protocol

```
┌────────────────────────────────────────────────────────────────┐
│                    Sync Protocol                               │
│                                                                │
│  Client maintains local state:                                │
│  - Local file index with hashes                              │
│  - Last sync cursor                                           │
│                                                                │
│  Sync process:                                                 │
│                                                                │
│  1. Client: GET /sync?cursor={last_cursor}                   │
│  2. Server: Returns changes since cursor                      │
│     {                                                          │
│       "entries": [                                             │
│         {"type": "file", "action": "update", ...},           │
│         {"type": "file", "action": "delete", ...}            │
│       ],                                                       │
│       "cursor": "new_cursor",                                 │
│       "has_more": false                                        │
│     }                                                          │
│  3. Client: Apply changes locally                             │
│  4. Client: Upload local changes                              │
│  5. Store new cursor                                           │
│                                                                │
│  Long-polling for real-time:                                  │
│  GET /sync/longpoll?cursor={cursor}&timeout=60               │
│  - Server holds connection until changes or timeout          │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Sync Service

```python
class SyncService:
    async def get_changes(self, user_id, cursor, limit=1000):
        """Get changes since cursor"""
        if cursor:
            # Decode cursor (timestamp + sequence)
            since = self.decode_cursor(cursor)
        else:
            since = datetime.min

        # Get changes from database
        changes = await self.db.query('''
            SELECT * FROM file_changes
            WHERE user_id = $1 AND changed_at > $2
            ORDER BY changed_at, sequence
            LIMIT $3
        ''', user_id, since, limit + 1)

        has_more = len(changes) > limit
        entries = changes[:limit]

        # Generate new cursor
        if entries:
            new_cursor = self.encode_cursor(entries[-1])
        else:
            new_cursor = cursor

        return {
            'entries': [self.format_entry(e) for e in entries],
            'cursor': new_cursor,
            'has_more': has_more
        }

    async def long_poll(self, user_id, cursor, timeout=60):
        """Wait for changes or timeout"""
        end_time = time.time() + timeout

        while time.time() < end_time:
            # Check for changes
            changes = await self.get_changes(user_id, cursor, limit=1)

            if changes['entries']:
                return changes

            # Wait before checking again
            await asyncio.sleep(1)

        # Timeout - return empty result with same cursor
        return {
            'entries': [],
            'cursor': cursor,
            'has_more': False
        }
```

## 6. Conflict Resolution

```
┌────────────────────────────────────────────────────────────────┐
│                  Conflict Resolution                           │
│                                                                │
│  Scenario: Two devices edit same file offline                │
│                                                                │
│  Device A: edit file.txt → "Hello World"                      │
│  Device B: edit file.txt → "Hello Universe"                   │
│                                                                │
│  Both come online and sync:                                   │
│                                                                │
│  Option 1: Last Write Wins                                    │
│  - Later timestamp overwrites                                 │
│  - Simple but may lose data                                   │
│                                                                │
│  Option 2: Create Conflict Copy (Dropbox approach)           │
│  - Keep both versions                                         │
│  - file.txt                                                    │
│  - file (conflicted copy from Device A).txt                  │
│                                                                │
│  Implementation:                                               │
│  1. Client uploads with expected parent version              │
│  2. If parent version mismatch → conflict                    │
│  3. Create conflict copy with device info                    │
│  4. User manually resolves                                    │
│                                                                │
│  Detection:                                                    │
│  - Compare content_hash                                       │
│  - Check parent_version                                       │
│  - Track device_id for changes                               │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 7. Deduplication

```python
class BlockService:
    async def store_block(self, block_hash, data):
        """Store block with deduplication"""
        # Check if block exists
        existing = await self.db.get('blocks', block_hash)

        if existing:
            # Block already exists, increment reference
            await self.db.execute('''
                UPDATE blocks
                SET reference_count = reference_count + 1
                WHERE block_hash = $1
            ''', block_hash)
            return existing['storage_path']

        # New block - store in S3
        storage_path = f"blocks/{block_hash[:2]}/{block_hash}"
        await self.s3.put_object(
            Bucket='dropbox-blocks',
            Key=storage_path,
            Body=data
        )

        # Record in database
        await self.db.insert('blocks', {
            'block_hash': block_hash,
            'size': len(data),
            'storage_path': storage_path,
            'reference_count': 1
        })

        return storage_path

    async def delete_block_reference(self, block_hash):
        """Decrement reference, delete if zero"""
        result = await self.db.execute('''
            UPDATE blocks
            SET reference_count = reference_count - 1
            WHERE block_hash = $1
            RETURNING reference_count
        ''', block_hash)

        if result['reference_count'] == 0:
            # Schedule deletion (with grace period)
            await self.queue.publish('block-cleanup', {
                'block_hash': block_hash,
                'scheduled_at': time.time() + 86400  # 24 hour delay
            })
```

## 8. Notification System

```
┌────────────────────────────────────────────────────────────────┐
│              Real-Time Notifications                           │
│                                                                │
│  When file changes:                                           │
│  1. Update database                                           │
│  2. Publish to notification service                          │
│  3. Push to all user's connected devices                     │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Notification Flow                                       │  │
│  │                                                          │  │
│  │  File Change ──▶ Kafka ──▶ Notification Service         │  │
│  │                                                          │  │
│  │  Notification Service:                                   │  │
│  │  1. Look up all devices for user                        │  │
│  │  2. For each connected device:                          │  │
│  │     - Push via WebSocket                                │  │
│  │  3. For disconnected devices:                           │  │
│  │     - Queue for next sync                               │  │
│  │                                                          │  │
│  │  Device connections tracked in Redis:                   │  │
│  │  Key: user:devices:{user_id}                            │  │
│  │  Value: Set of {device_id, server_id, connected_at}    │  │
│  │                                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 9. Key Takeaways

1. **Block-level chunking** - enables delta sync and deduplication
2. **Content-addressed storage** - store blocks by hash
3. **Cursor-based sync** - efficient incremental updates
4. **Conflict copies** - preserve data, let user resolve
5. **Long-polling** - real-time updates without WebSocket complexity
6. **Reference counting** - safe block garbage collection

---

Next: [Distributed Cache](../05-distributed-cache/README.md)
