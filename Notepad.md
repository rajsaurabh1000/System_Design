# Notepad (Collaborative Text Editor) - System Design

## 1. Functional Requirements (FR)

### Core Features
1. **Text Editing**
   - Create, edit, delete text
   - Rich text formatting (bold, italic, underline, colors, fonts)
   - Plain text mode support
   - Copy, cut, paste operations
   - Undo/redo functionality (unlimited history)
   - Find and replace

2. **Document Management**
   - Create new documents
   - Save documents
   - Auto-save (every few seconds)
   - Version history
   - Export to different formats (PDF, TXT, DOCX)

3. **Real-time Collaboration**
   - Multiple users editing simultaneously
   - See other users' cursors and selections
   - User presence indicators (who's online)
   - Comments and annotations
   - Conflict-free concurrent editing

4. **Organization**
   - Folders and nested folders
   - Search documents
   - Tags and labels
   - Favorites/starred documents
   - Recently opened

5. **Sharing & Permissions**
   - Share documents with specific users
   - Public link sharing
   - Permission levels (view, comment, edit, owner)
   - Access control

### Out of Scope
- Advanced word processing (tables, images embedding)
- Spell check and grammar
- Voice typing
- Mobile-specific features

## 2. Non-Functional Requirements (NFR)

1. **Performance**
   - Document load time: < 500ms for 100-page document
   - Typing latency: < 50ms
   - Collaborative edit propagation: < 100ms
   - Search results: < 500ms

2. **Scalability**
   - Support 100M registered users
   - Handle 10M concurrent users
   - Support documents up to 100MB (plain text ~10M words)
   - 1000 concurrent editors on popular documents

3. **Availability**
   - 99.9% uptime
   - Auto-save prevents data loss
   - Offline mode with sync when online

4. **Consistency**
   - Strong consistency for document content
   - Eventual consistency for presence/cursors
   - Operational Transformation or CRDT for conflict resolution

5. **Reliability**
   - No data loss
   - Automatic version history
   - Graceful degradation if services fail

6. **Usability**
   - Intuitive UI
   - Keyboard shortcuts
   - Responsive design

## 3. Core Entities

### 3.1 User
```
User {
  userId: UUID
  email: string
  name: string
  passwordHash: string
  avatarUrl: string
  createdAt: timestamp
  lastLoginAt: timestamp
}
```

### 3.2 Document
```
Document {
  documentId: UUID
  title: string
  ownerId: UUID (User)
  content: string (or structured format)
  contentLength: int
  folderId: UUID (Folder, nullable)
  createdAt: timestamp
  updatedAt: timestamp
  lastEditedBy: UUID (User)
  version: int
  isDeleted: boolean
}
```

### 3.3 Folder
```
Folder {
  folderId: UUID
  name: string
  ownerId: UUID (User)
  parentFolderId: UUID (Folder, nullable)
  createdAt: timestamp
  updatedAt: timestamp
}
```

### 3.4 Permission
```
Permission {
  permissionId: UUID
  documentId: UUID
  userId: UUID (or email for pending)
  accessLevel: enum (VIEWER, COMMENTER, EDITOR, OWNER)
  grantedBy: UUID (User)
  createdAt: timestamp
}
```

### 3.5 Operation (for OT/CRDT)
```
Operation {
  operationId: UUID
  documentId: UUID
  userId: UUID
  type: enum (INSERT, DELETE, FORMAT)
  position: int
  content: string
  length: int
  timestamp: long
  version: int
}
```

### 3.6 Comment
```
Comment {
  commentId: UUID
  documentId: UUID
  userId: UUID
  textRange: {start: int, end: int}
  content: string
  isResolved: boolean
  createdAt: timestamp
  updatedAt: timestamp
  replies: List<Comment>
}
```

### 3.7 Version
```
Version {
  versionId: UUID
  documentId: UUID
  content: string (snapshot)
  createdAt: timestamp
  createdBy: UUID (User)
  description: string
}
```

### 3.8 Session
```
Session {
  sessionId: UUID
  userId: UUID
  documentId: UUID
  cursorPosition: int
  selection: {start: int, end: int}
  lastActivity: timestamp
  isActive: boolean
}
```

## 4. API Design

### 4.1 Authentication APIs
```
POST /api/v1/auth/signup
Request: { email, password, name }
Response: { userId, token }

POST /api/v1/auth/login
Request: { email, password }
Response: { userId, token, refreshToken }

POST /api/v1/auth/logout
Headers: { Authorization: Bearer <token> }
Response: { success: true }
```

### 4.2 Document Management APIs
```
POST /api/v1/documents
Headers: { Authorization: Bearer <token> }
Request: { title, folderId?, content? }
Response: {
  documentId, title, ownerId, createdAt
}

GET /api/v1/documents/{documentId}
Headers: { Authorization: Bearer <token> }
Response: {
  documentId, title, content, ownerId,
  permissions, version, updatedAt, lastEditedBy
}

PUT /api/v1/documents/{documentId}
Headers: { Authorization: Bearer <token> }
Request: { title?, folderId? }
Response: { documentId, updatedFields }

DELETE /api/v1/documents/{documentId}
Headers: { Authorization: Bearer <token> }
Response: { success: true }

GET /api/v1/documents
Headers: { Authorization: Bearer <token> }
Query: { folderId?, page, limit, sortBy }
Response: {
  documents: [
    { documentId, title, updatedAt, preview }
  ],
  totalCount: int
}
```

### 4.3 Folder Management APIs
```
POST /api/v1/folders
Headers: { Authorization: Bearer <token> }
Request: { name, parentFolderId? }
Response: { folderId, name, createdAt }

GET /api/v1/folders/{folderId}
Headers: { Authorization: Bearer <token> }
Response: {
  folderId, name, parentFolderId,
  documents: [...],
  subfolders: [...]
}

DELETE /api/v1/folders/{folderId}
Headers: { Authorization: Bearer <token> }
Response: { success: true }
```

### 4.4 Real-time Collaboration (WebSocket)
```
WS /api/v1/documents/{documentId}/collaborate
Headers: { Authorization: Bearer <token> }

// Client -> Server: Edit operation
{
  type: "OPERATION",
  operation: {
    type: "INSERT",
    position: 100,
    content: "hello",
    version: 42
  }
}

// Server -> Client: Operation acknowledgment
{
  type: "ACK",
  operationId: UUID,
  serverVersion: 43
}

// Server -> All Clients: Broadcast operation
{
  type: "BROADCAST_OPERATION",
  userId: UUID,
  userName: "John Doe",
  operation: {
    type: "INSERT",
    position: 100,
    content: "hello",
    version: 43
  }
}

// Client -> Server: Cursor update
{
  type: "CURSOR_UPDATE",
  position: 150,
  selection: { start: 100, end: 150 }
}

// Server -> Other Clients: Broadcast cursor
{
  type: "USER_CURSOR",
  userId: UUID,
  userName: "John Doe",
  position: 150,
  selection: { start: 100, end: 150 },
  color: "#FF5733"
}

// Presence updates
{
  type: "USER_JOINED",
  userId: UUID,
  userName: "John Doe",
  color: "#FF5733"
}

{
  type: "USER_LEFT",
  userId: UUID
}
```

### 4.5 Comment APIs
```
POST /api/v1/documents/{documentId}/comments
Headers: { Authorization: Bearer <token> }
Request: {
  textRange: { start: int, end: int },
  content: string
}
Response: {
  commentId, userId, content, createdAt
}

GET /api/v1/documents/{documentId}/comments
Headers: { Authorization: Bearer <token> }
Response: {
  comments: [
    {
      commentId, userId, userName, content,
      textRange, isResolved, createdAt, replies: [...]
    }
  ]
}

PUT /api/v1/comments/{commentId}/resolve
Headers: { Authorization: Bearer <token> }
Response: { success: true }

POST /api/v1/comments/{commentId}/replies
Headers: { Authorization: Bearer <token> }
Request: { content: string }
Response: { replyId, content, createdAt }
```

### 4.6 Sharing & Permission APIs
```
POST /api/v1/documents/{documentId}/share
Headers: { Authorization: Bearer <token> }
Request: {
  userEmail: string,
  accessLevel: enum (VIEWER, COMMENTER, EDITOR)
}
Response: { permissionId, granted }

DELETE /api/v1/documents/{documentId}/permissions/{permissionId}
Headers: { Authorization: Bearer <token> }
Response: { success: true }

GET /api/v1/documents/{documentId}/permissions
Headers: { Authorization: Bearer <token> }
Response: {
  permissions: [
    { permissionId, userId, userName, email, accessLevel }
  ]
}
```

### 4.7 Version History APIs
```
GET /api/v1/documents/{documentId}/versions
Headers: { Authorization: Bearer <token> }
Response: {
  versions: [
    { versionId, createdAt, createdBy, description }
  ]
}

GET /api/v1/documents/{documentId}/versions/{versionId}
Headers: { Authorization: Bearer <token> }
Response: {
  versionId, content, createdAt, createdBy
}

POST /api/v1/documents/{documentId}/restore
Headers: { Authorization: Bearer <token> }
Request: { versionId }
Response: { success: true, currentVersion: int }
```

### 4.8 Search API
```
GET /api/v1/search
Headers: { Authorization: Bearer <token> }
Query: { query: string, folderId?, page, limit }
Response: {
  results: [
    {
      documentId, title, snippet,
      highlightedContent, relevanceScore
    }
  ],
  totalCount: int
}
```

### 4.9 Export API
```
POST /api/v1/documents/{documentId}/export
Headers: { Authorization: Bearer <token> }
Request: { format: "PDF" | "TXT" | "DOCX" }
Response: {
  downloadUrl: string,
  expiresAt: timestamp
}
```

## 5. High-Level Design (HLD)

### 5.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    CLIENT LAYER                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Web Client  │  │ Desktop App  │  │  Mobile App  │      │
│  │  (React +    │  │  (Electron)  │  │ (iOS/Android)│      │
│  │   Editor)    │  │              │  │              │      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
└─────────┼──────────────────┼──────────────────┼─────────────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │
                ┌────────────▼─────────────┐
                │   Load Balancer          │
                │   (NGINX / AWS ALB)      │
                └────────────┬─────────────┘
                             │
┌────────────────────────────┼──────────────────────────────┐
│               API GATEWAY LAYER                            │
│                ┌───────────▼─────────────┐                 │
│                │   API Gateway           │                 │
│                │   - Authentication      │                 │
│                │   - Rate Limiting       │                 │
│                │   - Request Routing     │                 │
│                └───────────┬─────────────┘                 │
└────────────────────────────┼──────────────────────────────┘
                             │
┌────────────────────────────┼──────────────────────────────┐
│          MICROSERVICES LAYER                               │
│                             │                              │
│    ┌───────────┬───────────┼──────────┬──────────┐        │
│    │           │           │          │          │        │
│ ┌──▼────┐  ┌──▼──────┐ ┌──▼─────┐ ┌──▼─────┐ ┌──▼─────┐  │
│ │ User  │  │Document │ │Collab. │ │Search  │ │Comment │  │
│ │Service│  │ Service │ │Service │ │Service │ │Service │  │
│ └──┬────┘  └──┬──────┘ └──┬─────┘ └──┬─────┘ └──┬─────┘  │
│    │          │           │          │          │        │
│ ┌──▼────┐  ┌──▼──────┐ ┌──▼─────┐ ┌──▼─────┐ ┌──▼─────┐  │
│ │Version│  │  Share  │ │  OT    │ │Export  │ │Notif.  │  │
│ │Service│  │ Service │ │Engine  │ │Service │ │Service │  │
│ └───────┘  └─────────┘ └────────┘ └────────┘ └────────┘  │
└────────────────────────┬──────────────────────────────────┘
                         │
┌────────────────────────┼──────────────────────────────────┐
│               DATA LAYER                                   │
│    ┌────────────┐  ┌───▼──────────┐  ┌──────────────┐    │
│    │ PostgreSQL │  │     Redis    │  │ElasticSearch │    │
│    │(Documents, │  │  (Cache,     │  │  (Search)    │    │
│    │ Users,     │  │  Sessions,   │  │              │    │
│    │Permissions)│  │  Presence)   │  │              │    │
│    └────────────┘  └──────────────┘  └──────────────┘    │
│    ┌────────────┐  ┌──────────────┐  ┌──────────────┐    │
│    │     S3     │  │    Kafka     │  │  Cassandra   │    │
│    │ (Exports,  │  │  (Events,    │  │(Op History,  │    │
│    │ Versions)  │  │  Op Queue)   │  │ Audit Logs)  │    │
│    └────────────┘  └──────────────┘  └──────────────┘    │
└───────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────┐
│                WEBSOCKET SERVER CLUSTER                    │
│          ┌──────────────────────────────┐                 │
│          │   WebSocket Servers          │                 │
│          │   (Socket.io / uWebSockets)  │                 │
│          │   - Sticky sessions          │                 │
│          │   - Pub/Sub for broadcast    │                 │
│          └──────────────────────────────┘                 │
└───────────────────────────────────────────────────────────┘
```

### 5.2 Component Descriptions

**Client Layer:**
- Rich text editor (Quill.js, Draft.js, or ProseMirror)
- WebSocket for real-time collaboration
- Local storage for offline mode
- Debouncing for auto-save

**API Gateway:**
- JWT authentication
- Rate limiting (per user)
- Request routing to services

**Microservices:**
- **User Service**: Authentication, user management
- **Document Service**: CRUD for documents, content storage
- **Collaboration Service**: WebSocket management, real-time updates
- **OT Engine**: Operational transformation for conflict resolution
- **Search Service**: Full-text search on documents
- **Comment Service**: Annotations and discussions
- **Share Service**: Permissions and access control
- **Version Service**: Snapshots and history
- **Export Service**: Convert to PDF/DOCX
- **Notification Service**: Email, push notifications

**Data Layer:**
- **PostgreSQL**: Documents, users, permissions, folders
- **Redis**: Session cache, active users, operation queue
- **ElasticSearch**: Document search indexing
- **Cassandra**: Operation history, audit logs
- **S3**: Exported files, version snapshots
- **Kafka**: Event streaming for operations

### 5.3 Real-time Editing Flow

```
┌─────────┐       ┌──────────┐       ┌────────┐       ┌─────────┐
│Client A │       │   WS     │       │   OT   │       │ Client B│
│         │       │  Server  │       │ Engine │       │         │
└────┬────┘       └────┬─────┘       └────┬───┘       └────┬────┘
     │                 │                  │                 │
     │ Type: Insert    │                  │                 │
     │ "hello" @ pos 10│                  │                 │
     ├────────────────>│                  │                 │
     │                 │  Queue Operation │                 │
     │                 ├─────────────────>│                 │
     │                 │                  │ Transform &     │
     │                 │                  │ Apply           │
     │                 │  Operation OK    │                 │
     │  ACK            │<─────────────────┤                 │
     │<────────────────┤                  │                 │
     │                 │                  │                 │
     │                 │  Broadcast to B  │                 │
     │                 ├──────────────────┼────────────────>│
     │                 │                  │                 │
     │                 │                  │  Type: Insert   │
     │                 │                  │  "world"@pos 10 │
     │                 │  B's Operation   │<────────────────┤
     │                 │<─────────────────┼─────────────────┤
     │                 │                  │                 │
     │                 │  Transform B vs A│                 │
     │                 ├─────────────────>│                 │
     │                 │  Transformed Op  │                 │
     │                 │<─────────────────┤                 │
     │  Broadcast to A │                  │  ACK            │
     │<────────────────┼──────────────────┼────────────────>│
     │                 │                  │                 │
```

### 5.4 Operational Transformation Example

```
Initial Document: "Hello"
Position:          01234

Client A (offline): Insert " World" at position 5
  Local result: "Hello World"

Client B (offline): Insert "!" at position 5
  Local result: "Hello!"

Both connect and send operations:

Server receives:
  Op_A: INSERT(" World", 5) at version 0
  Op_B: INSERT("!", 5) at version 0

Server applies Op_A first:
  Document: "Hello World" (version 1)
  
Server transforms Op_B against Op_A:
  Original Op_B: INSERT("!", 5)
  After transformation: INSERT("!", 5 + length(" World")) = INSERT("!", 11)
  
Final document: "Hello World!"

Server broadcasts:
  To A: Op_B transformed = INSERT("!", 11)
  To B: Op_A = INSERT(" World", 5)

Both clients converge to: "Hello World!"
```

### 5.5 Auto-save Strategy

```
Client Side:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  User Types  ──> Local Buffer  ──> Debounce (2s) ──>   │
│                                                         │
│      Send to Server (if changed)                        │
│                  │                                      │
│                  ▼                                      │
│            Server Auto-save                             │
│                  │                                      │
│                  ▼                                      │
│         Update "Last saved" indicator                   │
│                                                         │
└─────────────────────────────────────────────────────────┘

Offline Mode:
- Save to IndexedDB locally
- Queue operations
- Sync when back online
- Resolve conflicts using OT
```

## 6. Optimizations & Approaches

6.1 Operational Transformation (OT)

OT ensures that concurrent edits from multiple users are transformed so they can be applied in any order but still converge to the same document state. It preserves user intent and guarantees consistency in real-time collaboration.

6.2 CRDT (Alternative to OT)

CRDT assigns a unique ID to every character so operations are naturally commutative. It avoids complex transformations but increases memory usage, making it simpler but heavier than OT.

6.3 Document Storage Optimization

Documents are split into fixed-size chunks so only modified parts are read or written. This reduces I/O, improves performance, and avoids loading large documents fully into memory.

6.4 Caching Strategy

We use multi-level caching with client cache and Redis for metadata, content, and presence. Cache invalidation happens on updates via pub/sub, ensuring low latency and consistency.

6.5 Presence Optimization

Cursor and selection updates are throttled (e.g., every 200ms) to reduce network traffic. Presence data is eventually consistent and inactive users are auto-expired.

6.6 Search Optimization

Documents are indexed in Elasticsearch with boosted fields like title and incremental async updates. Indexing is debounced and batched to reduce write overhead.

6.7 Version Control Strategy

Snapshots are created after every N operations, while intermediate edits are stored as deltas. Older versions are pruned periodically to balance recovery and storage cost.

6.8 Database Sharding

Documents are sharded using consistent hashing on documentId to distribute load evenly. Operation logs are time-partitioned for fast writes and efficient cleanup.

6.9 WebSocket Connection Management

Sticky sessions ensure the same user connects to the same server. Redis Pub/Sub enables broadcasting edits across servers, supporting millions of concurrent users.

6.10 Undo / Redo Handling

Undo and redo are implemented using inverse operations stored in stacks. Each edit can be reversed safely without breaking collaborative consistency.

6.11 Conflict Resolution

All edits are accepted and transformed instead of rejected. Offline or concurrent changes are merged using OT so users never lose work.

## 7. Low-Level Design (LLD)

<img width="1705" height="823" alt="image" src="https://github.com/user-attachments/assets/bf58fd79-b282-462d-ad1f-fb3e032970c0" />

### 7.2 Detailed Component Implementation

#### 7.2.1 OT Engine

```java
public class OTEngine {
    private final Map<UUID, Integer> serverVersions;
    private final Map<UUID, Queue<Operation>> pendingOperations;
    
    public OTEngine() {
        this.serverVersions = new ConcurrentHashMap<>();
        this.pendingOperations = new ConcurrentHashMap<>();
    }
    
    public synchronized OperationResult applyOperation(
        UUID documentId,
        Operation clientOp
    ) {
        // Get current server version
        int serverVersion = serverVersions.getOrDefault(documentId, 0);
        
        // Get pending operations since client's version
        List<Operation> concurrentOps = getPendingOperations(
            documentId,
            clientOp.getVersion()
        );
        
        // Transform client operation against concurrent operations
        Operation transformedOp = clientOp;
        for (Operation serverOp : concurrentOps) {
            transformedOp = transform(transformedOp, serverOp);
        }
        
        // Apply transformed operation
        transformedOp.setVersion(serverVersion + 1);
        
        // Update server version
        serverVersions.put(documentId, serverVersion + 1);
        
        // Add to pending operations
        addPendingOperation(documentId, transformedOp);
        
        return new OperationResult(
            transformedOp,
            serverVersion + 1,
            true
        );
    }
    
    public Operation transform(Operation op1, Operation op2) {
        // Transform op1 against op2
        
        if (op1.getType() == OperationType.INSERT && 
            op2.getType() == OperationType.INSERT) {
            return transformInsertInsert(op1, op2);
        } else if (op1.getType() == OperationType.INSERT && 
                   op2.getType() == OperationType.DELETE) {
            return transformInsertDelete(op1, op2);
        } else if (op1.getType() == OperationType.DELETE && 
                   op2.getType() == OperationType.INSERT) {
            return transformDeleteInsert(op1, op2);
        } else if (op1.getType() == OperationType.DELETE && 
                   op2.getType() == OperationType.DELETE) {
            return transformDeleteDelete(op1, op2);
        }
        
        return op1;
    }
    
    private Operation transformInsertInsert(Operation op1, Operation op2) {
        Operation transformed = op1.copy();
        
        if (op2.getPosition() < op1.getPosition()) {
            // op2 is before op1, shift op1's position
            transformed.setPosition(
                op1.getPosition() + op2.getContent().length()
            );
        } else if (op2.getPosition() == op1.getPosition()) {
            // Same position - tie-break by timestamp or userId
            if (op2.getTimestamp() < op1.getTimestamp()) {
                transformed.setPosition(
                    op1.getPosition() + op2.getContent().length()
                );
            }
        }
        
        return transformed;
    }
    
    private Operation transformInsertDelete(Operation insert, Operation delete) {
        Operation transformed = insert.copy();
        
        int deleteEnd = delete.getPosition() + delete.getLength();
        
        if (insert.getPosition() >= deleteEnd) {
            // Insert is after delete, shift position left
            transformed.setPosition(
                insert.getPosition() - delete.getLength()
            );
        } else if (insert.getPosition() >= delete.getPosition()) {
            // Insert is inside delete range, move to delete start
            transformed.setPosition(delete.getPosition());
        }
        // else: insert is before delete, no change
        
        return transformed;
    }
    
    private Operation transformDeleteInsert(Operation delete, Operation insert) {
        Operation transformed = delete.copy();
        
        if (insert.getPosition() <= delete.getPosition()) {
            // Insert is before delete, shift delete position right
            transformed.setPosition(
                delete.getPosition() + insert.getContent().length()
            );
        } else if (insert.getPosition() < delete.getPosition() + delete.getLength()) {
            // Insert is inside delete range, increase delete length
            transformed.setLength(
                delete.getLength() + insert.getContent().length()
            );
        }
        
        return transformed;
    }
    
    private Operation transformDeleteDelete(Operation op1, Operation op2) {
        Operation transformed = op1.copy();
        
        int op1End = op1.getPosition() + op1.getLength();
        int op2End = op2.getPosition() + op2.getLength();
        
        if (op2End <= op1.getPosition()) {
            // op2 is completely before op1
            transformed.setPosition(op1.getPosition() - op2.getLength());
        } else if (op2.getPosition() >= op1End) {
            // op2 is completely after op1, no change
        } else {
            // Overlapping deletes - complex case
            int overlapStart = Math.max(op1.getPosition(), op2.getPosition());
            int overlapEnd = Math.min(op1End, op2End);
            int overlapLength = overlapEnd - overlapStart;
            
            transformed.setLength(op1.getLength() - overlapLength);
            
            if (op2.getPosition() < op1.getPosition()) {
                transformed.setPosition(op2.getPosition());
            }
        }
        
        return transformed;
    }
    
    private List<Operation> getPendingOperations(
        UUID documentId,
        int fromVersion
    ) {
        Queue<Operation> allOps = pendingOperations.get(documentId);
        if (allOps == null) {
            return Collections.emptyList();
        }
        
        return allOps.stream()
            .filter(op -> op.getVersion() > fromVersion)
            .collect(Collectors.toList());
    }
    
    private void addPendingOperation(UUID documentId, Operation op) {
        pendingOperations
            .computeIfAbsent(documentId, k -> new ConcurrentLinkedQueue<>())
            .add(op);
        
        // Cleanup old operations (keep last 1000)
        Queue<Operation> ops = pendingOperations.get(documentId);
        while (ops.size() > 1000) {
            ops.poll();
        }
    }
}
```

#### 7.2.2 Collaboration Manager

```java
@Component
public class CollaborationManager {
    private final Map<UUID, Set<WebSocketSession>> documentSessions;
    private final Map<UUID, Map<UUID, UserPresence>> presenceByDocument;
    private final OTEngine otEngine;
    private final DocumentService documentService;
    private final RedisTemplate<String, String> redisTemplate;
    
    public CollaborationManager() {
        this.documentSessions = new ConcurrentHashMap<>();
        this.presenceByDocument = new ConcurrentHashMap<>();
        this.otEngine = new OTEngine();
    }
    
    @Override
    public void handleConnection(
        UUID documentId,
        UUID userId,
        WebSocketSession session
    ) {
        // Add to active sessions
        documentSessions
            .computeIfAbsent(documentId, k -> ConcurrentHashMap.newKeySet())
            .add(session);
        
        // Add presence
        UserPresence presence = new UserPresence(
            userId,
            documentId,
            generateUserColor(userId),
            Instant.now()
        );
        
        presenceByDocument
            .computeIfAbsent(documentId, k -> new ConcurrentHashMap<>())
            .put(userId, presence);
        
        // Notify others
        broadcastUserJoined(documentId, userId, presence);
        
        // Send current document state to new user
        sendDocumentState(session, documentId);
        
        // Send list of active users
        sendActiveUsers(session, documentId);
    }
    
    @Override
    public void handleDisconnection(UUID documentId, UUID userId) {
        Set<WebSocketSession> sessions = documentSessions.get(documentId);
        if (sessions != null) {
            sessions.removeIf(s -> getUserId(s).equals(userId));
            
            if (sessions.isEmpty()) {
                documentSessions.remove(documentId);
            }
        }
        
        // Remove presence
        Map<UUID, UserPresence> presence = presenceByDocument.get(documentId);
        if (presence != null) {
            presence.remove(userId);
        }
        
        // Notify others
        broadcastUserLeft(documentId, userId);
    }
    
    public void handleOperation(
        UUID documentId,
        UUID userId,
        Operation operation
    ) {
        try {
            // Apply operation with OT
            OperationResult result = otEngine.applyOperation(
                documentId,
                operation
            );
            
            if (!result.isSuccess()) {
                sendError(documentId, userId, "Operation failed");
                return;
            }
            
            // Apply to document
            documentService.applyOperation(
                documentId,
                result.getTransformedOperation()
            );
            
            // Send acknowledgment to originating user
            sendAck(
                documentId,
                userId,
                operation.getOperationId(),
                result.getServerVersion()
            );
            
            // Broadcast to all other users
            broadcastOperation(
                documentId,
                userId,
                result.getTransformedOperation()
            );
            
            // Publish to Redis for cross-server broadcast
            publishOperation(documentId, result.getTransformedOperation());
            
        } catch (Exception e) {
            sendError(documentId, userId, e.getMessage());
        }
    }
    
    public void handleCursorUpdate(
        UUID documentId,
        UUID userId,
        int position,
        Selection selection
    ) {
        // Update presence
        Map<UUID, UserPresence> presence = presenceByDocument.get(documentId);
        if (presence != null && presence.containsKey(userId)) {
            UserPresence userPresence = presence.get(userId);
            userPresence.setPosition(position);
            userPresence.setSelection(selection);
            userPresence.setLastActivity(Instant.now());
        }
        
        // Broadcast cursor position (throttled)
        String key = "cursor:" + documentId + ":" + userId;
        Boolean shouldBroadcast = redisTemplate.opsForValue()
            .setIfAbsent(key, "1", Duration.ofMillis(200));
        
        if (Boolean.TRUE.equals(shouldBroadcast)) {
            broadcastCursor(documentId, userId, position, selection);
        }
    }
    
    private void broadcastOperation(
        UUID documentId,
        UUID originUserId,
        Operation operation
    ) {
        Set<WebSocketSession> sessions = documentSessions.get(documentId);
        if (sessions == null) return;
        
        OperationMessage message = new OperationMessage(
            "BROADCAST_OPERATION",
            originUserId,
            operation
        );
        
        String json = toJson(message);
        
        sessions.forEach(session -> {
            UUID sessionUserId = getUserId(session);
            if (!sessionUserId.equals(originUserId)) {
                send(session, json);
            }
        });
    }
    
    private void broadcastCursor(
        UUID documentId,
        UUID userId,
        int position,
        Selection selection
    ) {
        Set<WebSocketSession> sessions = documentSessions.get(documentId);
        if (sessions == null) return;
        
        UserPresence presence = presenceByDocument.get(documentId).get(userId);
        
        CursorMessage message = new CursorMessage(
            "USER_CURSOR",
            userId,
            presence.getUserName(),
            position,
            selection,
            presence.getColor()
        );
        
        String json = toJson(message);
        
        sessions.forEach(session -> {
            if (!getUserId(session).equals(userId)) {
                send(session, json);
            }
        });
    }
    
    private void publishOperation(UUID documentId, Operation operation) {
        // Publish to Redis for cross-server synchronization
        String channel = "document:operations:" + documentId;
        redisTemplate.convertAndSend(channel, toJson(operation));
    }
    
    // Cleanup inactive users
    @Scheduled(fixedDelay = 30000) // Every 30 seconds
    public void cleanupInactiveUsers() {
        Instant timeout = Instant.now().minus(Duration.ofSeconds(30));
        
        presenceByDocument.forEach((documentId, presenceMap) -> {
            presenceMap.entrySet().removeIf(entry -> {
                if (entry.getValue().getLastActivity().isBefore(timeout)) {
                    broadcastUserLeft(documentId, entry.getKey());
                    return true;
                }
                return false;
            });
        });
    }
}
```

#### 7.2.3 Document Service

```java
@Service
public class DocumentService {
    private final DocumentRepository documentRepository;
    private final VersionService versionService;
    private final PermissionService permissionService;
    private final SearchService searchService;
    private final CacheManager cacheManager;
    
    @Transactional
    public Document createDocument(String title, UUID ownerId, UUID folderId) {
        Document document = Document.builder()
            .documentId(UUID.randomUUID())
            .title(title)
            .ownerId(ownerId)
            .folderId(folderId)
            .content("")
            .contentLength(0)
            .version(0)
            .createdAt(Instant.now())
            .updatedAt(Instant.now())
            .build();
        
        documentRepository.save(document);
        
        // Create initial version
        versionService.createSnapshot(document);
        
        // Index for search
        searchService.indexDocument(document);
        
        return document;
    }
    
    public Document getDocument(UUID documentId, UUID userId) {
        // Check cache first
        String cacheKey = "document:" + documentId;
        Document cached = cacheManager.get(cacheKey);
        if (cached != null) {
            return cached;
        }
        
        // Fetch from database
        Document document = documentRepository.findById(documentId)
            .orElseThrow(() -> new DocumentNotFoundException(documentId));
        
        // Check permissions
        if (!permissionService.hasAccess(userId, documentId, AccessLevel.VIEWER)) {
            throw new AccessDeniedException();
        }
        
        // Cache it
        cacheManager.set(cacheKey, document, Duration.ofMinutes(30));
        
        return document;
    }
    
    @Transactional
    public void applyOperation(UUID documentId, Operation operation) {
        Document document = documentRepository.findById(documentId)
            .orElseThrow(() -> new DocumentNotFoundException(documentId));
        
        // Apply operation to content
        String newContent = applyOperationToContent(
            document.getContent(),
            operation
        );
        
        document.setContent(newContent);
        document.setContentLength(newContent.length());
        document.setVersion(document.getVersion() + 1);
        document.setUpdatedAt(Instant.now());
        document.setLastEditedBy(operation.getUserId());
        
        documentRepository.save(document);
        
        // Invalidate cache
        cacheManager.invalidate("document:" + documentId);
        
        // Check if snapshot needed (every 100 operations)
        if (document.getVersion() % 100 == 0) {
            versionService.createSnapshot(document);
        }
        
        // Update search index (debounced)
        searchService.updateIndex(documentId);
    }
    
    private String applyOperationToContent(String content, Operation operation) {
        switch (operation.getType()) {
            case INSERT:
                return content.substring(0, operation.getPosition())
                    + operation.getContent()
                    + content.substring(operation.getPosition());
            
            case DELETE:
                int endPos = operation.getPosition() + operation.getLength();
                return content.substring(0, operation.getPosition())
                    + content.substring(endPos);
            
            case FORMAT:
                // For rich text, would apply formatting metadata
                return content;
            
            default:
                throw new UnsupportedOperationException(
                    "Unknown operation type: " + operation.getType()
                );
        }
    }
    
    @Transactional
    public void deleteDocument(UUID documentId, UUID userId) {
        Document document = documentRepository.findById(documentId)
            .orElseThrow(() -> new DocumentNotFoundException(documentId));
        
        // Check ownership
        if (!document.getOwnerId().equals(userId)) {
            throw new AccessDeniedException("Only owner can delete");
        }
        
        // Soft delete
        document.setDeleted(true);
        document.setUpdatedAt(Instant.now());
        documentRepository.save(document);
        
        // Remove from cache and search index
        cacheManager.invalidate("document:" + documentId);
        searchService.deleteFromIndex(documentId);
    }
}
```

### 7.3 Database Schema

```sql
-- Users table
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    avatar_url VARCHAR(500),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    last_login_at TIMESTAMP,
    INDEX idx_email (email)
);

-- Documents table
CREATE TABLE documents (
    document_id UUID PRIMARY KEY,
    title VARCHAR(500) NOT NULL,
    owner_id UUID NOT NULL,
    folder_id UUID,
    content TEXT,
    content_length INT NOT NULL DEFAULT 0,
    version INT NOT NULL DEFAULT 0,
    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    last_edited_by UUID,
    FOREIGN KEY (owner_id) REFERENCES users(user_id),
    FOREIGN KEY (folder_id) REFERENCES folders(folder_id),
    INDEX idx_owner (owner_id),
    INDEX idx_folder (folder_id),
    INDEX idx_updated (updated_at)
);

-- Folders table
CREATE TABLE folders (
    folder_id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    owner_id UUID NOT NULL,
    parent_folder_id UUID,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    FOREIGN KEY (owner_id) REFERENCES users(user_id),
    FOREIGN KEY (parent_folder_id) REFERENCES folders(folder_id),
    INDEX idx_owner (owner_id),
    INDEX idx_parent (parent_folder_id)
);

-- Permissions table
CREATE TABLE permissions (
    permission_id UUID PRIMARY KEY,
    document_id UUID NOT NULL,
    user_id UUID NOT NULL,
    access_level VARCHAR(20) NOT NULL,
    granted_by UUID NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    FOREIGN KEY (document_id) REFERENCES documents(document_id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (granted_by) REFERENCES users(user_id),
    UNIQUE KEY uk_doc_user (document_id, user_id)
);

-- Versions table
CREATE TABLE versions (
    version_id UUID PRIMARY KEY,
    document_id UUID NOT NULL,
    content TEXT NOT NULL,
    version_number INT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    created_by UUID NOT NULL,
    description VARCHAR(500),
    FOREIGN KEY (document_id) REFERENCES documents(document_id) ON DELETE CASCADE,
    FOREIGN KEY (created_by) REFERENCES users(user_id),
    INDEX idx_document (document_id),
    UNIQUE KEY uk_doc_version (document_id, version_number)
);

-- Comments table
CREATE TABLE comments (
    comment_id UUID PRIMARY KEY,
    document_id UUID NOT NULL,
    user_id UUID NOT NULL,
    content TEXT NOT NULL,
    text_range_start INT NOT NULL,
    text_range_end INT NOT NULL,
    is_resolved BOOLEAN NOT NULL DEFAULT FALSE,
    parent_comment_id UUID,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    FOREIGN KEY (document_id) REFERENCES documents(document_id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (parent_comment_id) REFERENCES comments(comment_id),
    INDEX idx_document (document_id),
    INDEX idx_resolved (is_resolved)
);

-- Cassandra: Operation history
CREATE TABLE operation_history (
    document_id UUID,
    timestamp BIGINT,
    operation_id UUID,
    user_id UUID,
    operation_type TEXT,
    position INT,
    content TEXT,
    length INT,
    version INT,
    PRIMARY KEY (document_id, timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC);
```

### 7.4 Scalability Calculations

**User Load:**
- 100M registered users
- 10M daily active users
- 1M concurrent editors
- Average 10 operations/minute per user = ~167K ops/second

**Data Volume:**
- Average document: 10KB
- 100M documents = 1TB
- With versions: 5TB

**WebSocket Connections:**
- 1M concurrent connections
- 100 servers, 10K connections each
- Redis pub/sub for cross-server communication

**Operation Processing:**
- 167K operations/second
- Each operation: Transform + persist + broadcast
- Average latency: 50ms

---

## Interview Discussion Points

### Clarifying Questions:
1. Scale: Concurrent users? Document sizes?
2. Real-time: How critical is low latency?
3. Offline: Is offline mode required?
4. Collaboration: Max simultaneous editors per document?
5. Features: Rich text or plain text focus?

### Trade-offs:
1. **OT vs CRDT**: OT preserves intent, CRDT simpler
2. **Storage**: Chunks vs full document
3. **Consistency**: Strong vs eventual
4. **Versioning**: Granularity vs storage cost

### Extensions:
1. **Voice/Video Chat**: Integrated communication
2. **AI Features**: Auto-complete, summarization
3. **Templates**: Pre-made document templates
4. **Mobile**: Touch-optimized editor
5. **LaTeX Support**: Math equations

This design provides a solid foundation for a collaborative text editor like Google Docs or Notion!

