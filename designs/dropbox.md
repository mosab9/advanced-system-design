# Design Dropbox (File Storage & Sync)

Cloud file storage with cross-device synchronization.

---

## 1. Requirements

### Functional
- Upload/download files
- Sync files across devices
- Share files/folders
- Version history
- Offline access

### Non-Functional
- High reliability (no data loss)
- Low sync latency
- Bandwidth efficient
- Handle large files (GB+)

---

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Desktop/Mobile Clients                         │
│                   (File watcher, sync engine)                       │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        ▼                       ▼                       ▼
┌───────────────┐      ┌───────────────┐      ┌───────────────┐
│   Metadata    │      │   Block       │      │  Notification │
│   Service     │      │   Service     │      │   Service     │
└───────┬───────┘      └───────┬───────┘      └───────────────┘
        │                      │
        ▼                      ▼
┌───────────────┐      ┌───────────────┐
│  Metadata DB  │      │ Block Storage │
│  (PostgreSQL) │      │    (S3)       │
└───────────────┘      └───────────────┘
```

---

## 3. File Chunking

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Chunking Strategy                               │
│                                                                     │
│   Large file split into fixed-size chunks (4MB typical)           │
│                                                                     │
│   File: document.pdf (50 MB)                                       │
│   ┌────────┬────────┬────────┬────────┬───────┐                   │
│   │ Chunk1 │ Chunk2 │ Chunk3 │ ...    │Chunk13│                   │
│   │  4MB   │  4MB   │  4MB   │        │  2MB  │                   │
│   └────────┴────────┴────────┴────────┴───────┘                   │
│                                                                     │
│   Each chunk:                                                      │
│   - Hash computed (SHA-256)                                        │
│   - Stored with hash as key                                       │
│   - Enables deduplication                                          │
│                                                                     │
│   Benefits:                                                        │
│   1. Only changed chunks uploaded on edit                         │
│   2. Dedup across users (same content = same hash)               │
│   3. Parallel upload/download                                     │
│   4. Resume interrupted transfers                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. Sync Protocol

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Sync Flow                                       │
│                                                                     │
│   Upload (file changed locally):                                   │
│   1. Detect file change (file watcher)                            │
│   2. Compute chunk hashes                                          │
│   3. Compare with server metadata                                 │
│   4. Upload only new/changed chunks                               │
│   5. Update metadata (file → chunks mapping)                      │
│   6. Notify other clients                                          │
│                                                                     │
│   Download (file changed remotely):                                │
│   1. Receive notification (long polling/WebSocket)                │
│   2. Fetch updated metadata                                        │
│   3. Identify missing chunks                                      │
│   4. Download only missing chunks                                 │
│   5. Reconstruct file                                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. Metadata Model

```sql
-- Files/Folders
CREATE TABLE files (
    id              UUID PRIMARY KEY,
    user_id         UUID,
    parent_id       UUID,           -- Folder hierarchy
    name            VARCHAR(500),
    is_folder       BOOLEAN,
    size            BIGINT,
    checksum        VARCHAR(64),    -- File hash
    version         INT,
    created_at      TIMESTAMP,
    modified_at     TIMESTAMP,
    deleted_at      TIMESTAMP       -- Soft delete
);

-- Chunks
CREATE TABLE chunks (
    hash            VARCHAR(64) PRIMARY KEY,  -- SHA-256
    size            INT,
    reference_count INT,            -- For dedup
    storage_path    VARCHAR(500)
);

-- File-Chunk mapping
CREATE TABLE file_chunks (
    file_id         UUID,
    chunk_index     INT,
    chunk_hash      VARCHAR(64),
    PRIMARY KEY (file_id, chunk_index)
);
```

---

## 6. Conflict Resolution

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Conflict Handling                               │
│                                                                     │
│   Scenario: Same file edited on two devices while offline          │
│                                                                     │
│   Original: v1                                                     │
│   Device A (offline): v1 → v2a                                     │
│   Device B (offline): v1 → v2b                                     │
│                                                                     │
│   Both come online:                                                │
│   Device A syncs first → Server has v2a                           │
│   Device B tries to sync → Conflict detected!                     │
│                                                                     │
│   Resolution options:                                              │
│   1. Last writer wins (simple, may lose data)                     │
│   2. Create conflicted copy:                                      │
│      - document.pdf (original from A)                             │
│      - document (B's conflicted copy).pdf                         │
│   3. Operational Transform (for real-time collab)                 │
│                                                                     │
│   Dropbox uses option 2 (conflicted copies)                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. Deduplication

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Deduplication                                   │
│                                                                     │
│   User A uploads: photo.jpg (hash: abc123)                        │
│   Storage: S3/abc123 ← photo data                                 │
│   Metadata: file_A → chunk(abc123)                                │
│                                                                     │
│   User B uploads: same_photo.jpg (hash: abc123)                   │
│   Storage: Already exists!                                         │
│   Metadata: file_B → chunk(abc123)                                │
│                                                                     │
│   Result:                                                          │
│   - Only one copy stored                                          │
│   - Both users reference same chunk                               │
│   - Significant storage savings                                   │
│                                                                     │
│   Reference counting for deletion:                                 │
│   - Delete file_A → decrement ref count                          │
│   - Delete file_B → ref count = 0 → delete chunk                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 8. Interview Tips

**Key Points:**
- Chunking for efficient sync
- Content-addressable storage (hash as key)
- Deduplication across users
- Delta sync (only changed chunks)
- Conflict resolution strategy
- Long polling/WebSocket for notifications

**Questions to Ask:**
- Average file size?
- Real-time collaboration needed?
- Version history retention?
- Sharing permissions model?
