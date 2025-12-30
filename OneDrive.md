# OneDrive - System Design

## 1. Functional Requirements (FR)

### Core Features
1. **File Upload/Download**
   - Users can upload files of various formats and sizes
   - Users can download their files from any device
   - Support for resumable uploads/downloads

2. **File Synchronization**
   - Automatic sync across multiple devices
   - Real-time sync when files are modified
   - Conflict resolution when same file edited on multiple devices

3. **File Sharing**
   - Share files/folders with other users
   - Public link sharing with optional password protection
   - Permission management (view, edit, comment)

4. **File Versioning**
   - Maintain version history of files
   - Ability to restore previous versions
   - Compare versions

5. **File Organization**
   - Create/delete/rename folders
   - Move files between folders
   - Search files by name, type, date

6. **Collaboration**
   - Real-time collaborative editing
   - Comments and notifications
   - Activity tracking

### Out of Scope (for this discussion)
- Mobile app specific features
- Advanced AI-powered search
- Third-party integrations

## 2. Non-Functional Requirements (NFR)

1. **Scalability**
   - Support 1 billion users
   - Handle 100M daily active users
   - Support files up to 250GB per file
   - Handle peak loads (3x normal traffic)

2. **Availability**
   - 99.99% uptime (52 minutes downtime/year)
   - No single point of failure
   - Regional failover capabilities

3. **Performance**
   - Upload/download speed: Limited by user's network bandwidth
   - File sync latency: < 1 second for small files
   - Search results: < 500ms
   - File metadata operations: < 200ms

4. **Consistency**
   - Strong consistency for file metadata
   - Eventual consistency acceptable for file content sync
   - Conflict resolution for concurrent edits

5. **Security**
   - End-to-end encryption for data in transit
   - Encryption at rest
   - Authentication and authorization
   - Audit logs

6. **Durability**
   - 99.999999999% (11 nines) durability
   - No data loss
   - Backup and disaster recovery

## 3. Core Entities

### 3.1 User
```
User {
  userId: UUID
  email: string
  name: string
  passwordHash: string
  storageQuota: long
  usedStorage: long
  createdAt: timestamp
  lastLoginAt: timestamp
  plan: enum (FREE, PREMIUM, BUSINESS)
}
```

### 3.2 File
```
File {
  fileId: UUID
  name: string
  ownerId: UUID (User)
  parentFolderId: UUID (Folder)
  size: long
  mimeType: string
  checksum: string (MD5/SHA256)
  storageLocation: string
  isDeleted: boolean
  createdAt: timestamp
  updatedAt: timestamp
  version: int
}
```

### 3.3 Folder
```
Folder {
  folderId: UUID
  name: string
  ownerId: UUID (User)
  parentFolderId: UUID (Folder, nullable)
  isDeleted: boolean
  createdAt: timestamp
  updatedAt: timestamp
}
```

### 3.4 FileVersion
```
FileVersion {
  versionId: UUID
  fileId: UUID
  version: int
  size: long
  storageLocation: string
  checksum: string
  createdAt: timestamp
  createdBy: UUID (User)
}
```

### 3.5 Permission
```
Permission {
  permissionId: UUID
  resourceId: UUID (File/Folder)
  resourceType: enum (FILE, FOLDER)
  userId: UUID
  accessLevel: enum (OWNER, EDITOR, VIEWER, COMMENTER)
  grantedBy: UUID (User)
  createdAt: timestamp
  expiresAt: timestamp (nullable)
}
```

### 3.6 ShareLink
```
ShareLink {
  linkId: UUID
  resourceId: UUID
  resourceType: enum (FILE, FOLDER)
  createdBy: UUID (User)
  accessLevel: enum (VIEW, EDIT)
  password: string (nullable)
  expiresAt: timestamp (nullable)
  createdAt: timestamp
}
```

### 3.7 SyncQueue
```
SyncQueue {
  queueId: UUID
  userId: UUID
  deviceId: string
  fileId: UUID
  operation: enum (UPLOAD, DOWNLOAD, UPDATE, DELETE)
  status: enum (PENDING, IN_PROGRESS, COMPLETED, FAILED)
  retryCount: int
  createdAt: timestamp
  updatedAt: timestamp
}
```

## 4. API Design

### 4.1 Authentication APIs
```
POST /api/v1/auth/register
Request: { email, password, name }
Response: { userId, token }

POST /api/v1/auth/login
Request: { email, password }
Response: { userId, token, refreshToken }

POST /api/v1/auth/logout
Headers: { Authorization: Bearer <token> }
Response: { success: true }
```

### 4.2 File Management APIs
```
POST /api/v1/files/upload
Headers: { Authorization: Bearer <token> }
Request: multipart/form-data
  - file: binary
  - parentFolderId: UUID
  - metadata: { name, size, checksum }
Response: { fileId, name, size, uploadedAt }

GET /api/v1/files/{fileId}/download
Headers: { Authorization: Bearer <token> }
Response: Binary stream

PUT /api/v1/files/{fileId}
Headers: { Authorization: Bearer <token> }
Request: { name?, parentFolderId? }
Response: { fileId, updatedFields }

DELETE /api/v1/files/{fileId}
Headers: { Authorization: Bearer <token> }
Response: { success: true }

GET /api/v1/files/{fileId}/metadata
Headers: { Authorization: Bearer <token> }
Response: { fileId, name, size, ownerId, createdAt, updatedAt, version }
```

### 4.3 Folder Management APIs
```
POST /api/v1/folders
Headers: { Authorization: Bearer <token> }
Request: { name, parentFolderId? }
Response: { folderId, name, createdAt }

GET /api/v1/folders/{folderId}/contents
Headers: { Authorization: Bearer <token> }
Query: { page, limit, sortBy, order }
Response: {
  files: [...],
  folders: [...],
  totalCount: int,
  page: int
}
```

### 4.4 Sync APIs
```
POST /api/v1/sync/register-device
Headers: { Authorization: Bearer <token> }
Request: { deviceId, deviceName, platform }
Response: { deviceId, syncToken }

GET /api/v1/sync/changes
Headers: { Authorization: Bearer <token> }
Query: { lastSyncTimestamp, deviceId }
Response: {
  changes: [
    { fileId, operation, timestamp, metadata }
  ],
  nextSyncTimestamp: timestamp
}

POST /api/v1/sync/acknowledge
Headers: { Authorization: Bearer <token> }
Request: { deviceId, syncedChanges: [changeId] }
Response: { success: true }
```

### 4.5 Sharing APIs
```
POST /api/v1/share/permission
Headers: { Authorization: Bearer <token> }
Request: { resourceId, resourceType, userEmail, accessLevel }
Response: { permissionId, granted }

POST /api/v1/share/link
Headers: { Authorization: Bearer <token> }
Request: { resourceId, resourceType, accessLevel, password?, expiresAt? }
Response: { shareUrl, linkId }

GET /api/v1/share/{linkId}
Request: { password? }
Response: { resourceId, accessLevel, metadata }
```

### 4.6 Version Control APIs
```
GET /api/v1/files/{fileId}/versions
Headers: { Authorization: Bearer <token> }
Response: {
  versions: [
    { versionId, version, size, createdAt, createdBy }
  ]
}

POST /api/v1/files/{fileId}/restore
Headers: { Authorization: Bearer <token> }
Request: { versionId }
Response: { fileId, currentVersion }
```

### 4.7 Search API
```
GET /api/v1/search
Headers: { Authorization: Bearer <token> }
Query: { query, type?, dateFrom?, dateTo?, limit, offset }
Response: {
  results: [
    { resourceId, type, name, path, relevanceScore }
  ],
  totalCount: int
}
```

## 5. High-Level Design (HLD)

### 5.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                          CLIENT LAYER                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │  Web Client  │  │ Desktop App  │  │  Mobile App  │              │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘              │
└─────────┼──────────────────┼──────────────────┼──────────────────────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │
                    ┌────────▼────────┐
                    │   CDN / Edge    │
                    │   Locations     │
                    └────────┬────────┘
                             │
┌────────────────────────────┼────────────────────────────────────────┐
│                   API GATEWAY LAYER                                  │
│                    ┌────────▼────────┐                               │
│                    │  Load Balancer  │                               │
│                    └────────┬────────┘                               │
│              ┌──────────────┼──────────────┐                        │
│              │              │              │                         │
│      ┌───────▼──────┐ ┌────▼─────┐ ┌─────▼──────┐                  │
│      │  API Gateway │ │   Auth   │ │Rate Limiter│                  │
│      │   (Kong/AWS) │ │  Service │ │            │                  │
│      └───────┬──────┘ └────┬─────┘ └─────┬──────┘                  │
└──────────────┼─────────────┼─────────────┼─────────────────────────┘
               │             │             │
┌──────────────┼─────────────┼─────────────┼─────────────────────────┐
│              │   APPLICATION SERVICE LAYER │                        │
│    ┌─────────▼──────┐  ┌──▼──────────┐  ┌▼──────────────┐          │
│    │ Metadata       │  │   Sync      │  │  Notification │          │
│    │ Service        │  │  Service    │  │   Service     │          │
│    └─────────┬──────┘  └──┬──────────┘  └┬──────────────┘          │
│    ┌─────────▼──────┐  ┌──▼──────────┐  ┌▼──────────────┐          │
│    │ Upload/Download│  │  Search     │  │   Sharing     │          │
│    │ Service        │  │  Service    │  │   Service     │          │
│    └─────────┬──────┘  └──┬──────────┘  └┬──────────────┘          │
│    ┌─────────▼──────┐  ┌──▼──────────┐  ┌▼──────────────┐          │
│    │  Version       │  │ Collaboration│ │   Webhook     │          │
│    │  Control       │  │   Service   │  │   Service     │          │
│    └─────────┬──────┘  └──┬──────────┘  └┬──────────────┘          │
└──────────────┼────────────┼──────────────┼─────────────────────────┘
               │            │              │
┌──────────────┼────────────┼──────────────┼─────────────────────────┐
│              │     DATA LAYER            │                          │
│    ┌─────────▼──────┐  ┌──▼──────────┐  ┌▼──────────────┐          │
│    │   PostgreSQL   │  │   Redis     │  │   ElasticSearch│         │
│    │   (Metadata)   │  │  (Cache)    │  │   (Search)    │          │
│    └────────────────┘  └─────────────┘  └───────────────┘          │
│    ┌────────────────┐  ┌─────────────┐  ┌───────────────┐          │
│    │  Amazon S3/    │  │  Message    │  │   Cassandra   │          │
│    │  Blob Storage  │  │  Queue      │  │   (Logs)      │          │
│    │  (File Data)   │  │ (Kafka/SQS) │  └───────────────┘          │
│    └────────────────┘  └─────────────┘                             │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    BACKGROUND WORKERS                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │ Sync Worker  │  │ Index Worker │  │Cleanup Worker│              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 Component Descriptions

**Client Layer:**
- Web, Desktop, Mobile clients for file access
- Desktop app includes file watcher for real-time sync

**API Gateway Layer:**
- Load balancer distributes traffic
- API Gateway handles routing, SSL termination
- Auth Service validates tokens
- Rate Limiter prevents abuse

**Application Services:**
- **Metadata Service**: Manages file/folder metadata, permissions
- **Upload/Download Service**: Handles file transfers with chunking
- **Sync Service**: Coordinates file synchronization across devices
- **Search Service**: Indexes and searches file metadata
- **Sharing Service**: Manages permissions and share links
- **Version Control**: Maintains file versions
- **Notification Service**: Sends real-time updates via WebSockets
- **Collaboration Service**: Handles real-time editing
- **Webhook Service**: Triggers external integrations

**Data Layer:**
- **PostgreSQL**: Stores structured metadata (users, files, permissions)
- **Redis**: Caches frequently accessed data, session management
- **ElasticSearch**: Full-text search on file names and content
- **S3/Blob Storage**: Stores actual file data
- **Message Queue**: Asynchronous task processing
- **Cassandra**: Stores logs and activity history

**Background Workers:**
- Sync workers for device synchronization
- Index workers for search indexing
- Cleanup workers for deleted files and expired shares

### 5.3 Upload Flow

```
┌──────┐         ┌─────────┐       ┌─────────┐       ┌──────────┐
│Client│         │  API    │       │ Upload  │       │ Metadata │
│      │         │ Gateway │       │ Service │       │ Service  │
└──┬───┘         └────┬────┘       └────┬────┘       └────┬─────┘
   │                  │                 │                  │
   │ 1. Init Upload   │                 │                  │
   ├─────────────────>│                 │                  │
   │                  │  Authenticate   │                  │
   │                  ├────────────────>│                  │
   │                  │                 │  Check Quota    │
   │                  │                 ├─────────────────>│
   │                  │                 │  Quota OK        │
   │                  │                 │<─────────────────┤
   │                  │  Upload URL +   │                  │
   │  Upload URL      │  Chunk Info     │                  │
   │<─────────────────┤<────────────────┤                  │
   │                  │                 │                  │
   │ 2. Upload Chunks │                 │                  │
   ├──────────────────┼────────────────>│                  │
   │    (to S3)       │                 │                  │
   │                  │                 │                  │
   │ 3. Complete      │                 │                  │
   ├─────────────────>│                 │                  │
   │                  │  Verify Upload  │                  │
   │                  ├────────────────>│                  │
   │                  │                 │  Save Metadata  │
   │                  │                 ├─────────────────>│
   │                  │                 │                  │
   │                  │                 │  Trigger Sync   │
   │                  │                 ├─────────────────>│
   │  Success         │                 │                  │
   │<─────────────────┤<────────────────┤                  │
   │                  │                 │                  │
```

### 5.4 Sync Flow

```
┌─────────┐      ┌──────────┐      ┌────────────┐      ┌──────┐
│Device A │      │   Sync   │      │  Message   │      │Device│
│         │      │  Service │      │   Queue    │      │  B   │
└────┬────┘      └────┬─────┘      └─────┬──────┘      └───┬──┘
     │                │                   │                 │
     │ File Modified  │                   │                 │
     ├───────────────>│                   │                 │
     │                │  Publish Event    │                 │
     │                ├──────────────────>│                 │
     │                │                   │  Notify Device B│
     │                │                   ├────────────────>│
     │                │                   │                 │
     │                │  Poll for Changes │                 │
     │                │<──────────────────┼─────────────────┤
     │                │  Return Delta     │                 │
     │                ├───────────────────┼────────────────>│
     │                │                   │                 │
     │                │                   │  Download File  │
     │                │<──────────────────┼─────────────────┤
     │                │  Send File Data   │                 │
     │                ├───────────────────┼────────────────>│
     │                │                   │                 │
```

## 6. Optimizations & Approaches

### 6.1 File Chunking Strategy
**Problem**: Large files are difficult to upload/download in one request
**Solution**: 
- Split files into 4MB chunks
- Upload chunks in parallel
- Resume from last successful chunk on failure
- Use multipart upload API (S3)

### 6.2 Deduplication
**Problem**: Multiple users storing same files wastes storage
**Solution**:
- Calculate SHA-256 hash for each file
- Before upload, check if hash exists in system
- If exists, create metadata entry pointing to existing storage
- Reference counting for deletions
- Saves ~30-40% storage in practice

### 6.3 Delta Sync
**Problem**: Re-uploading entire file on small changes is inefficient
**Solution**:
- Use algorithms like rsync or binary diff
- Client sends only changed blocks
- Server reconstructs file from deltas
- Particularly useful for large files with small edits

### 6.4 Caching Strategy
**Layers**:
1. **Client-side cache**: Local file system cache
2. **CDN cache**: Static content and popular files
3. **Redis cache**: 
   - User metadata (TTL: 1 hour)
   - File metadata (TTL: 30 minutes)
   - Permissions (TTL: 15 minutes)
   - Share links (TTL: 1 hour)

**Cache Invalidation**:
- Write-through cache for metadata updates
- Event-driven invalidation using message queue
- TTL-based expiration as fallback

### 6.5 Database Sharding
**Metadata Database Sharding**:
- Shard by userId (consistent hashing)
- Each shard contains all files/folders for subset of users
- Shared files stored in owner's shard with permission references

**Advantages**:
- Horizontal scalability
- Fault isolation
- Better query performance

### 6.6 Hot/Cold Storage Tiering
**Strategy**:
- Hot storage (SSD/Premium S3): Files accessed in last 30 days
- Warm storage (Standard S3): Files accessed in last 90 days
- Cold storage (Glacier): Files older than 90 days
- Archive: Files older than 1 year

**Implementation**:
- Background job analyzes access patterns
- Automatically moves files between tiers
- Transparent to users (may have retrieval delay for cold files)

### 6.7 Conflict Resolution
**Scenarios**:
1. **Same file edited on multiple devices offline**
2. **Concurrent edits by multiple users**

**Resolution Strategy**:
- Last-Write-Wins (LWW) with version vectors
- For conflicts, create copies: "file (Device A's conflicted copy).txt"
- Notify user to manually resolve
- For collaborative editing, use Operational Transformation (OT) or CRDT

### 6.8 Rate Limiting
**Levels**:
1. **User level**: 1000 API calls/hour
2. **IP level**: 5000 calls/hour
3. **Upload**: 100 GB/day for free users
4. **Download**: Unlimited (bandwidth limited)

**Implementation**: Token bucket algorithm with Redis

### 6.9 Compression
- Compress files before upload (optional, user choice)
- Always compress metadata in transit
- Use Brotli/Gzip for HTTP responses
- Don't compress already compressed formats (jpg, mp4, zip)

### 6.10 Predictive Prefetching
- Analyze user access patterns
- Prefetch files likely to be accessed soon
- Prefetch to client cache during idle time
- Machine learning model for prediction

## 7. Low-Level Design (LLD)

<img width="2012" height="892" alt="image" src="https://github.com/user-attachments/assets/89062222-abd0-4992-9d38-e2cb87f04e49" />


### Design Patterns Used (Short Interview Lines)

Factory → Creates upload sessions, metadata objects, version entities

Singleton → Cache manager, configuration manager

Repository → Abstracts DB access for files, folders, versions

Strategy → Different sync, conflict-resolution, and dedup strategies

Observer (Pub/Sub) → Notify devices of file changes in real time

Command → Upload, delete, move, restore operations

Facade → API Gateway exposes a unified interface

Decorator → Adds compression, encryption layers

Builder → Constructs complex metadata and responses

Chain of Responsibility → Validation → quota → dedup → upload pipeline

Adapter → Integrates storage providers (S3, Azure Blob, GCS)

Prototype → Used for version cloning and conflict copies

### Core Data Structures (One-liners – Interview Friendly)

File metadata store → HashMap / DB index – Fast lookup by fileId

Folder hierarchy → Tree / Adjacency List – Represents nested folders

Permissions → Map<ResourceId, List<Permission>> – Fast access control checks

Version history → List / Append-only log – Tracks file evolution

Sync queue → Queue / PriorityQueue – Manages pending device sync tasks

Dedup index → HashMap<Checksum, Location> – Enables content-based reuse

Search index → Inverted Index – Fast full-text search

Cache → LRU Cache – Stores frequently accessed metadata

Delta blocks → Map<BlockHash, BlockData> – Efficient diff-based sync

Device registry → Map<DeviceId, Device> – Tracks active devices

Event stream → Append-only log (Kafka) – Async propagation of changes

Permissions lookup → HashMap – O(1) access control validation

Version comparison → Directed Graph – Handles version lineage

Conflict tracking → Set / Map – Detects concurrent edits

Quota tracking → Counter + Atomic updates – Enforces storage limits

### 7.3 Database Schema

```sql
-- Users table
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    storage_quota BIGINT NOT NULL DEFAULT 5368709120, -- 5GB
    used_storage BIGINT NOT NULL DEFAULT 0,
    plan VARCHAR(50) NOT NULL DEFAULT 'FREE',
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    last_login_at TIMESTAMP,
    INDEX idx_email (email)
);

-- Files table
CREATE TABLE files (
    file_id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    owner_id UUID NOT NULL,
    parent_folder_id UUID,
    size BIGINT NOT NULL,
    mime_type VARCHAR(100),
    checksum VARCHAR(64) NOT NULL,
    storage_location VARCHAR(500) NOT NULL,
    version INT NOT NULL DEFAULT 1,
    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    FOREIGN KEY (owner_id) REFERENCES users(user_id),
    FOREIGN KEY (parent_folder_id) REFERENCES folders(folder_id),
    INDEX idx_owner (owner_id),
    INDEX idx_parent (parent_folder_id),
    INDEX idx_checksum (checksum),
    INDEX idx_updated (updated_at)
);

-- Folders table
CREATE TABLE folders (
    folder_id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    owner_id UUID NOT NULL,
    parent_folder_id UUID,
    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    FOREIGN KEY (owner_id) REFERENCES users(user_id),
    FOREIGN KEY (parent_folder_id) REFERENCES folders(folder_id),
    INDEX idx_owner (owner_id),
    INDEX idx_parent (parent_folder_id)
);

-- File versions table
CREATE TABLE file_versions (
    version_id UUID PRIMARY KEY,
    file_id UUID NOT NULL,
    version INT NOT NULL,
    size BIGINT NOT NULL,
    storage_location VARCHAR(500) NOT NULL,
    checksum VARCHAR(64) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    created_by UUID NOT NULL,
    FOREIGN KEY (file_id) REFERENCES files(file_id) ON DELETE CASCADE,
    FOREIGN KEY (created_by) REFERENCES users(user_id),
    INDEX idx_file (file_id),
    UNIQUE KEY uk_file_version (file_id, version)
);

-- Permissions table
CREATE TABLE permissions (
    permission_id UUID PRIMARY KEY,
    resource_id UUID NOT NULL,
    resource_type VARCHAR(20) NOT NULL, -- FILE or FOLDER
    user_id UUID NOT NULL,
    access_level VARCHAR(20) NOT NULL, -- OWNER, EDITOR, VIEWER, COMMENTER
    granted_by UUID NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (granted_by) REFERENCES users(user_id),
    INDEX idx_resource (resource_id, resource_type),
    INDEX idx_user (user_id),
    UNIQUE KEY uk_resource_user (resource_id, user_id)
);

-- Share links table
CREATE TABLE share_links (
    link_id UUID PRIMARY KEY,
    resource_id UUID NOT NULL,
    resource_type VARCHAR(20) NOT NULL,
    created_by UUID NOT NULL,
    access_level VARCHAR(20) NOT NULL,
    password_hash VARCHAR(255),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMP,
    access_count INT NOT NULL DEFAULT 0,
    FOREIGN KEY (created_by) REFERENCES users(user_id),
    INDEX idx_resource (resource_id)
);

-- Devices table
CREATE TABLE devices (
    device_id VARCHAR(255) PRIMARY KEY,
    user_id UUID NOT NULL,
    device_name VARCHAR(255) NOT NULL,
    platform VARCHAR(50) NOT NULL,
    last_sync_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    INDEX idx_user (user_id)
);

-- File hash deduplication table
CREATE TABLE file_hashes (
    checksum VARCHAR(64) PRIMARY KEY,
    storage_location VARCHAR(500) NOT NULL,
    reference_count INT NOT NULL DEFAULT 1,
    file_size BIGINT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Sync queue table
CREATE TABLE sync_queue (
    queue_id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    device_id VARCHAR(255) NOT NULL,
    file_id UUID NOT NULL,
    operation VARCHAR(20) NOT NULL, -- UPLOAD, DOWNLOAD, UPDATE, DELETE
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    retry_count INT NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (device_id) REFERENCES devices(device_id),
    INDEX idx_user_device (user_id, device_id),
    INDEX idx_status (status)
);
```

### 7.4 Key Algorithms

#### 7.4.1 Chunking Algorithm

#### 7.4.2 Delta Sync Algorithm (simplified rsync)

### 7.5 Scalability Calculations

**Storage Estimation:**
- 1B users
- Average 10GB per user
- Total: 10 EB (10,000 PB)
- With deduplication (40% savings): ~6 EB

**Bandwidth Estimation:**
- 100M DAU
- Average 100MB upload/download per day per user
- Total: 10 PB/day = ~120 GB/second
- Peak (3x): ~360 GB/second

**Database Sizing:**
- Files: 1B users × 1000 files = 1T file records
- File metadata: ~1KB per record = 1 PB metadata
- With sharding: 1000 shards = 1TB per shard

**Cache Sizing:**
- Hot metadata: 10% of files accessed daily
- 100M files × 1KB = 100GB metadata in cache
- User sessions: 100M users × 1KB = 100GB
- Total Redis: ~200GB (replicated across regions)

---

## Interview Discussion Points

### Clarifying Questions to Ask:
1. How many users and what's the expected growth?
2. What's the average file size?
3. Are we supporting real-time collaboration?
4. What's the read/write ratio?
5. Any compliance requirements (GDPR, HIPAA)?
6. Mobile support requirements?
7. Offline mode needed?

### Follow-up Topics:
1. **Security**: Encryption, key management, compliance
2. **Monitoring**: Metrics, alerts, logging
3. **Disaster Recovery**: Backup strategy, RTO/RPO
4. **Cost Optimization**: Storage tiering, compression
5. **Mobile Considerations**: Bandwidth optimization, selective sync
6. **Enterprise Features**: SSO, admin controls, audit logs

This design provides a solid foundation for a production-grade file storage and synchronization system like OneDrive.

