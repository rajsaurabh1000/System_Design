# Excel (Spreadsheet Application) - System Design

## 1. Functional Requirements (FR)

### Core Features
1. **Cell Operations**
   - Create, read, update, delete cell values
   - Support multiple data types (text, numbers, dates, boolean)
   - Cell formatting (bold, italic, colors, borders)
   - Merge/unmerge cells
   - Copy/paste cells with formatting

2. **Formulas and Functions**
   - Basic arithmetic operations (+, -, *, /)
   - Built-in functions (SUM, AVERAGE, COUNT, IF, VLOOKUP, etc.)
   - Cell references (A1, $A$1, A1:B10)
   - Formula auto-complete and validation
   - Formula error handling

3. **Sheet Management**
   - Create, rename, delete sheets
   - Multiple sheets per workbook
   - Navigate between sheets
   - Copy/move sheets

4. **Real-time Collaboration**
   - Multiple users editing simultaneously
   - See other users' cursors and selections
   - Conflict resolution
   - User presence indicators

5. **Data Operations**
   - Sort data (ascending/descending)
   - Filter data
   - Find and replace
   - Data validation rules

6. **Import/Export**
   - Export to CSV, PDF, Excel format
   - Import from CSV, Excel
   - Copy/paste from external sources

7. **Undo/Redo**
   - Maintain operation history
   - Support undo/redo for multiple operations

### Out of Scope
- Macros and VBA scripting
- Pivot tables (can be discussed as extension)
- Advanced charting
- Mobile app specific features

## 2. Non-Functional Requirements (NFR)

1. **Performance**
   - Cell update latency: < 50ms
   - Formula recalculation: < 100ms for 10K cells
   - Load spreadsheet: < 2 seconds for 1MB file
   - Collaborative updates: < 100ms propagation
   - Support sheets with up to 1M rows × 100 columns

2. **Scalability**
   - Support 1M concurrent users
   - Handle 10K concurrent editors on same sheet
   - Support files up to 100MB
   - Handle 1000 operations per second per sheet

3. **Availability**
   - 99.9% uptime
   - Auto-save every 30 seconds
   - Graceful degradation if services fail

4. **Consistency**
   - Eventual consistency for collaborative edits
   - Strong consistency for formula calculations
   - CRDT or OT for conflict-free concurrent edits

5. **Reliability**
   - No data loss
   - Automatic version history
   - Recover from crashes

6. **Usability**
   - Responsive UI (60 FPS scrolling)
   - Keyboard shortcuts
   - Intuitive formula syntax

## 3. Core Entities

### 3.1 Workbook
```
Workbook {
  workbookId: UUID
  name: string
  ownerId: UUID (User)
  sheets: List<Sheet>
  createdAt: timestamp
  updatedAt: timestamp
  accessLevel: enum (PRIVATE, SHARED, PUBLIC)
  version: int
}
```

### 3.2 Sheet
```
Sheet {
  sheetId: UUID
  workbookId: UUID
  name: string
  index: int
  rowCount: int
  columnCount: int
  cells: Map<CellAddress, Cell>
  createdAt: timestamp
  updatedAt: timestamp
}
```

### 3.3 Cell
```
Cell {
  cellId: string (e.g., "A1")
  sheetId: UUID
  row: int
  column: int
  value: any (string, number, boolean, date)
  formula: string (nullable)
  dataType: enum (TEXT, NUMBER, DATE, BOOLEAN, FORMULA)
  formatting: CellFormat
  updatedAt: timestamp
  updatedBy: UUID (User)
}
```

### 3.4 CellFormat
```
CellFormat {
  fontFamily: string
  fontSize: int
  bold: boolean
  italic: boolean
  underline: boolean
  textColor: string (hex)
  backgroundColor: string (hex)
  textAlign: enum (LEFT, CENTER, RIGHT)
  numberFormat: string (e.g., "#,##0.00")
  borders: BorderStyle
}
```

### 3.5 Formula
```
Formula {
  formulaId: UUID
  expression: string
  dependencies: List<CellAddress>
  result: any
  error: string (nullable)
  calculatedAt: timestamp
}
```

### 3.6 Operation (for OT/CRDT)
```
Operation {
  operationId: UUID
  workbookId: UUID
  sheetId: UUID
  type: enum (UPDATE_CELL, INSERT_ROW, DELETE_ROW, FORMAT_CELL)
  cellAddress: string
  oldValue: any
  newValue: any
  timestamp: long
  userId: UUID
  version: int
}
```

### 3.7 User
```
User {
  userId: UUID
  name: string
  email: string
  color: string (for cursor in collaboration)
  currentCell: CellAddress (nullable)
  isActive: boolean
}
```

### 3.8 Version
```
Version {
  versionId: UUID
  workbookId: UUID
  timestamp: timestamp
  userId: UUID
  snapshot: binary (compressed workbook state)
  changeDescription: string
}
```

## 4. API Design

### 4.1 Workbook APIs
```
POST /api/v1/workbooks
Headers: { Authorization: Bearer <token> }
Request: { name: string, template?: string }
Response: { workbookId: UUID, name: string, sheets: [] }

GET /api/v1/workbooks/{workbookId}
Headers: { Authorization: Bearer <token> }
Response: {
  workbookId: UUID,
  name: string,
  sheets: [{ sheetId, name, index }],
  metadata: { createdAt, updatedAt, owner }
}

DELETE /api/v1/workbooks/{workbookId}
Headers: { Authorization: Bearer <token> }
Response: { success: boolean }

PUT /api/v1/workbooks/{workbookId}
Headers: { Authorization: Bearer <token> }
Request: { name?: string }
Response: { workbookId: UUID, name: string }
```

### 4.2 Sheet APIs
```
POST /api/v1/workbooks/{workbookId}/sheets
Headers: { Authorization: Bearer <token> }
Request: { name: string, index?: int }
Response: { sheetId: UUID, name: string }

GET /api/v1/sheets/{sheetId}
Headers: { Authorization: Bearer <token> }
Response: {
  sheetId: UUID,
  name: string,
  dimensions: { rows: int, columns: int },
  cells: { "A1": {...}, "B2": {...} }
}

DELETE /api/v1/sheets/{sheetId}
Headers: { Authorization: Bearer <token> }
Response: { success: boolean }

PUT /api/v1/sheets/{sheetId}
Headers: { Authorization: Bearer <token> }
Request: { name?: string, index?: int }
Response: { sheetId: UUID, name: string }
```

### 4.3 Cell APIs
```
PUT /api/v1/sheets/{sheetId}/cells/{cellAddress}
Headers: { Authorization: Bearer <token> }
Request: {
  value?: any,
  formula?: string,
  format?: CellFormat
}
Response: {
  cellAddress: string,
  value: any,
  calculatedValue: any (if formula),
  updatedAt: timestamp
}

GET /api/v1/sheets/{sheetId}/cells/{cellAddress}
Headers: { Authorization: Bearer <token> }
Response: {
  cellAddress: string,
  value: any,
  formula?: string,
  format: CellFormat
}

POST /api/v1/sheets/{sheetId}/cells/batch
Headers: { Authorization: Bearer <token> }
Request: {
  operations: [
    { cellAddress: "A1", value: 100 },
    { cellAddress: "B1", formula: "=A1*2" }
  ]
}
Response: {
  results: [{ cellAddress, value, success }]
}
```

### 4.4 Collaboration APIs (WebSocket)
```
WS /api/v1/workbooks/{workbookId}/collaborate
Headers: { Authorization: Bearer <token> }

// Client -> Server
{
  type: "OPERATION",
  operation: {
    type: "UPDATE_CELL",
    sheetId: UUID,
    cellAddress: "A1",
    value: 100,
    version: 42
  }
}

// Server -> Client
{
  type: "OPERATION_ACK",
  operationId: UUID,
  serverVersion: 43
}

// Server -> All Clients
{
  type: "BROADCAST_OPERATION",
  userId: UUID,
  userName: "John Doe",
  operation: {...}
}

// Cursor position
{
  type: "CURSOR_MOVE",
  cellAddress: "B5"
}

// Server broadcasts to others
{
  type: "USER_CURSOR",
  userId: UUID,
  userName: "John Doe",
  cellAddress: "B5",
  color: "#FF5733"
}
```

### 4.5 Formula APIs
```
POST /api/v1/formulas/validate
Headers: { Authorization: Bearer <token> }
Request: { formula: "=SUM(A1:A10)" }
Response: {
  valid: boolean,
  error?: string,
  dependencies: ["A1", "A2", ..., "A10"]
}

POST /api/v1/formulas/calculate
Headers: { Authorization: Bearer <token> }
Request: {
  sheetId: UUID,
  formula: "=SUM(A1:A10)",
  context: { "A1": 10, "A2": 20, ... }
}
Response: {
  result: 550,
  dataType: "NUMBER"
}
```

### 4.6 Import/Export APIs
```
POST /api/v1/workbooks/{workbookId}/export
Headers: { Authorization: Bearer <token> }
Request: { format: "CSV" | "PDF" | "XLSX" }
Response: { downloadUrl: string, expiresAt: timestamp }

POST /api/v1/workbooks/import
Headers: { Authorization: Bearer <token> }
Request: multipart/form-data
  - file: binary
  - format: "CSV" | "XLSX"
Response: { workbookId: UUID, name: string }
```

### 4.7 Version History APIs
```
GET /api/v1/workbooks/{workbookId}/versions
Headers: { Authorization: Bearer <token> }
Response: {
  versions: [
    { versionId, timestamp, userId, description }
  ]
}

POST /api/v1/workbooks/{workbookId}/restore
Headers: { Authorization: Bearer <token> }
Request: { versionId: UUID }
Response: { success: boolean, currentVersion: int }
```

## 5. High-Level Design (HLD)

### 5.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      CLIENT LAYER                               │
│  ┌──────────────────┐  ┌──────────────────┐                     │
│  │   Web Browser    │  │   Desktop App    │                     │
│  │  (React + Grid)  │  │   (Electron)     │                     │
│  └────────┬─────────┘  └────────┬─────────┘                     │
└───────────┼────────────────────  ┼──────────────────────────────┘
            │                      │
            └──────────┬───────────┘
                       │
          ┌────────────▼─────────────┐
          │    Load Balancer         │
          └────────────┬─────────────┘
                       │
┌──────────────────────┼───────────────────────────────────────────┐
│              API GATEWAY LAYER                                   │
│          ┌───────────▼────────────┐                              │
│          │    API Gateway         │                              │
│          │  (Auth, Rate Limit)    │                              │
│          └───────────┬────────────┘                              │
└──────────────────────┼───────────────────────────────────────────┘
                       │
┌──────────────────────┼───────────────────────────────────────────┐
│         APPLICATION SERVICE LAYER                                │
│                      │                                           │
│    ┌─────────────────┼─────────────────┐                        │
│    │                 │                 │                         │
│ ┌──▼──────────┐  ┌───▼────────┐  ┌────▼────────┐                │
│ │  Workbook   │  │   Sheet    │  │Collaboration│                │
│ │  Service    │  │  Service   │  │  Service    │                │
│ └──┬──────────┘  └───┬────────┘  └────┬────────┘                │
│    │                 │                 │                         │
│ ┌──▼──────────┐  ┌───▼────────┐  ┌────▼────────┐                │
│ │   Cell      │  │  Formula   │  │   Export    │                │
│ │  Service    │  │  Engine    │  │  Service    │                │
│ └──┬──────────┘  └───┬────────┘  └────┬────────┘                │
│    │                 │                 │                         │
│ ┌──▼──────────┐  ┌───▼────────┐  ┌────▼────────┐                │
│ │  Version    │  │   OT/CRDT  │  │Notification │                │
│ │  Service    │  │  Resolver  │  │  Service    │                │
│ └─────────────┘  └────────────┘  └─────────────┘                │
└──────────────────────┬───────────────────────────────────────────┘
                       │
┌──────────────────────┼───────────────────────────────────────────┐
│              DATA LAYER                                          │
│    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│    │  PostgreSQL  │  │    Redis     │  │   MongoDB    │         │
│    │  (Metadata)  │  │   (Cache,    │  │ (Cell Data)  │         │
│    │              │  │  Sessions)   │  │              │         │
│    └──────────────┘  └──────────────┘  └──────────────┘         │
│    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│    │   S3/Blob    │  │    Kafka     │  │ ElasticCache │         │
│    │  (Exports,   │  │  (Message    │  │  (Sessions)  │         │
│    │  Versions)   │  │   Queue)     │  │              │         │
│    └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                  WEBSOCKET LAYER                                │
│          ┌──────────────────────────────┐                       │
│          │   WebSocket Server Cluster   │                       │
│          │   (Socket.io / uWebSockets)  │                       │
│          └──────────────────────────────┘                       │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 Component Descriptions

**Client Layer:**
- Web browser with optimized grid rendering (virtual scrolling)
- Desktop app with local caching
- Real-time WebSocket connection for collaboration

**API Gateway:**
- Authentication and authorization
- Rate limiting per user
- Request routing

**Application Services:**
- **Workbook Service**: CRUD operations for workbooks
- **Sheet Service**: Manages sheets and layout
- **Cell Service**: Handles cell updates and queries
- **Formula Engine**: Parses, validates, and calculates formulas
- **Collaboration Service**: Manages WebSocket connections, broadcasts operations
- **OT/CRDT Resolver**: Resolves concurrent edit conflicts
- **Version Service**: Creates and manages snapshots
- **Export Service**: Converts to different formats
- **Notification Service**: Sends real-time updates

**Data Layer:**
- **PostgreSQL**: Stores workbook and sheet metadata
- **MongoDB**: Stores cell data (document per sheet)
- **Redis**: Caches frequently accessed cells, user sessions, operation queue
- **S3**: Stores exported files and version snapshots
- **Kafka**: Event streaming for operations

### 5.3 Cell Update Flow

```
┌────────┐        ┌──────────┐       ┌──────────┐       ┌────────┐
│ Client │        │   WS     │       │   Cell   │       │ Formula│
│        │        │  Server  │       │  Service │       │ Engine │
└───┬────┘        └────┬─────┘       └────┬─────┘       └────┬───┘
    │                  │                  │                  │
    │ Update Cell A1   │                  │                  │
    ├─────────────────>│                  │                  │
    │                  │  Save Operation  │                  │
    │                  ├─────────────────>│                  │
    │                  │                  │  Update DB       │
    │                  │                  ├──────────────┐   │
    │                  │                  │              │   │
    │                  │                  │<─────────────┘   │
    │                  │                  │                  │
    │                  │                  │  Find Dependents │
    │                  │                  ├─────────────────>│
    │                  │                  │  Cells: [B1, C1] │
    │                  │                  │<─────────────────┤
    │                  │                  │                  │
    │                  │                  │ Recalculate B1,C1│
    │                  │                  ├─────────────────>│
    │                  │                  │  Results         │
    │                  │                  │<─────────────────┤
    │                  │                  │                  │
    │                  │  Broadcast All   │                  │
    │  ACK + Updates   │  Changes         │                  │
    │<─────────────────┤<─────────────────┤                  │
    │                  │                  │                  │
    │                  │  Notify Other    │                  │
    │                  │  Clients         │                  │
    │                  ├──────────────────>                  │
    │                  │  (WebSocket)     │                  │
```

### 5.4 Collaboration Flow (Operational Transformation)

```
Client A                 Server                   Client B
   │                        │                         │
   │ Op1: A1="Hello"        │                         │
   │ (local version: 5)     │                         │
   ├───────────────────────>│                         │
   │                        │ Transform & Apply       │
   │                        │ (server version: 6)     │
   │                        │                         │
   │                        │ Broadcast Op1           │
   │                        ├────────────────────────>│
   │                        │                         │
   │                        │  Op2: B1="World"        │
   │                        │  (local version: 5)     │
   │                        │<────────────────────────┤
   │                        │                         │
   │                        │ Transform Op2 against Op1│
   │                        │ Apply (server version: 7)│
   │                        │                         │
   │ Broadcast Op2          │                         │
   │<───────────────────────┤                         │
   │ Transform & Apply      │                         │
   │                        │                         │
```

### 5.5 Formula Dependency Graph

```
      A1 (10)
       │
       ├─────────┐
       │         │
       ▼         ▼
      B1        C1
    =A1*2     =A1+5
     (20)      (15)
       │         │
       └────┬────┘
            ▼
           D1
        =B1+C1
          (35)

When A1 changes:
1. Identify dependents: [B1, C1]
2. Recalculate B1, C1
3. Identify second-level dependents: [D1]
4. Recalculate D1
5. Broadcast all changes
```

## 6. Optimizations & Approaches

### 6.1 Virtual Scrolling
**Problem**: Rendering millions of cells causes browser to freeze
**Solution**:
- Render only visible cells (viewport + buffer)
- Use canvas or WebGL for rendering large grids
- Lazy load cell data on scroll
- Typical viewport: 20 rows × 10 columns = 200 cells rendered

### 6.2 Cell Data Storage Strategy
**Problem**: Storing individual cells is expensive
**Solution**:
- Store only non-empty cells (sparse matrix)
- Use MongoDB document per sheet
- Structure:
```json
{
  "sheetId": "uuid",
  "cells": {
    "A1": { "value": 100, "format": {...} },
    "B5": { "value": "text", "format": {...} }
  },
  "metadata": { ... }
}
```
- Index on sheetId for fast retrieval
- For very large sheets, partition into chunks (1000 rows per chunk)

### 6.3 Formula Calculation Optimization
**Problem**: Recalculating all formulas on every change is slow
**Solution**:
- **Dependency Graph**: Build directed acyclic graph (DAG) of cell dependencies
- **Incremental Calculation**: Only recalculate cells affected by changed cell
- **Topological Sort**: Calculate in correct order
- **Caching**: Cache formula results until dependencies change
- **Lazy Evaluation**: Calculate only visible cells, defer others


### 6.4 Operational Transformation (OT)
**Problem**: Multiple users editing same cell simultaneously
**Solution**: Use OT to transform concurrent operations

**Example**:
```
Initial state: A1 = ""

Client A: Insert "Hello" at position 0
Client B: Insert "World" at position 0 (concurrent)

Server receives:
1. Op_A: Insert("Hello", 0)
   Result: "Hello"
   
2. Op_B: Insert("World", 0)
   Transform Op_B against Op_A:
   New position = 0 + length("Hello") = 5
   Op_B': Insert("World", 5)
   Result: "HelloWorld"

Broadcast Op_B' to Client A
```

### 6.5 CRDT Alternative
For simpler conflict resolution, use CRDT (Conflict-free Replicated Data Type):
- Each cell has unique identifier
- Operations are commutative
- Last-Write-Wins (LWW) with timestamps
- Simpler than OT but less precise for text editing

### 6.6 Caching Strategy
**Multi-Level Cache**:
1. **Client Cache**: Cache entire visible sheet
2. **Redis Cache**: 
   - Hot cells (frequently accessed): TTL 5 minutes
   - Formula results: Invalidate on dependency change
   - User sessions: TTL 30 minutes
3. **CDN**: Cache static assets (UI, fonts)

**Cache Invalidation**:
- Write-through: Update cache on write
- Event-driven: Invalidate via pub/sub when cell changes

### 6.7 Database Sharding
**Strategy**: Shard by workbookId
- Each shard contains all sheets for subset of workbooks
- Range-based sharding: workbookId hash % num_shards
- Collaborative editing on same workbook stays in one shard

### 6.8 Formula Parsing Optimization
**Problem**: Parsing formulas on every calculation is slow
**Solution**:
- Parse formula once, store AST (Abstract Syntax Tree)
- Cache parsed formulas in Redis
- Validate syntax on input, reject invalid formulas early

**Example**:
```
Formula: =SUM(A1:A10)*2+B1

AST:
    +
   / \
  *   B1
 / \
SUM  2
 |
A1:A10

Store AST, evaluate by traversing tree
```

### 6.9 Debouncing Updates
**Problem**: Rapid typing causes too many updates
**Solution**:
- Debounce cell updates (250ms delay)
- Send batch updates instead of individual cells
- Local-first: Update UI immediately, sync to server asynchronously

### 6.10 Compression
- Compress cell data in database (MongoDB snappy compression)
- Compress WebSocket messages (per-message deflate)
- Compress exports before download
- Typical compression ratio: 5:1 for spreadsheets

### 6.11 Auto-save Strategy
**Problem**: Don't want to lose data, but can't save on every keystroke
**Solution**:
- Local storage: Save to browser localStorage immediately
- Server sync: Save to server every 30 seconds
- Operation log: Save each operation to queue, batch process
- Version snapshots: Create full snapshot every 100 operations

## 7. Low-Level Design (LLD)

### 7.1 Class Diagram

<img width="1618" height="1491" alt="image" src="https://github.com/user-attachments/assets/f3c7e548-e98d-438b-a6dd-71419eaf124d" />



🧩 Design Patterns Used
Pattern	Where Used	Purpose
Strategy	Formula evaluation, sorting, filtering, recalculation	Plug different algorithms dynamically
Factory	Formula parsers, exporters (CSV/XLSX/PDF)	Centralized object creation
Singleton	WorkbookManager, CollaborationManager, CacheManager	Ensure single shared coordinator
Facade	WorkbookService	Simplifies access to complex subsystems
Command	Cell update / row insert / delete operations	Encapsulates user actions
State	CellState, SheetState, CollaborationState	Controlled state transitions
Observer	UI updates, cursor tracking, live collaboration	Push real-time updates
Adapter	External integrations (import/export, payment, file formats)	Interface compatibility
Template Method	Formula execution pipeline	Shared algorithm skeleton
Repository	WorkbookRepository, SheetRepository	Abstract persistence layer
Builder	Workbook / Sheet / Cell creation	Safe object construction
Prototype	Sheet / workbook duplication	Fast cloning
Mediator	CollaborationManager	Coordinates users & operations
Flyweight	Cell formatting styles	Memory optimization
Decorator	Cell formatting layers	Add formatting dynamically
Chain of Responsibility	Formula validation & execution	Sequential processing
Observer + Pub/Sub	WebSocket collaboration	Real-time synchronization
📦 Data Structures Used
Component	Data Structure	Purpose
Cells	Map<CellAddress, Cell>	O(1) lookup for any cell
Sheet data	Sparse map / document model	Efficient storage of empty grids
Rows / Sheets	List / ArrayList	Ordered traversal
Active users	ConcurrentHashMap<UserId, Session>	Thread-safe collaboration
Dependency graph	Directed Graph (Adjacency List)	Formula dependency tracking
Dependency set	Set<CellAddress>	Prevent duplicates
Recalculation order	Topological Sort	Correct formula evaluation
Formula AST	Tree structure	Expression parsing
Formula cache	LRU Cache	Faster recomputation
Operations log	Queue / Deque	Undo / redo handling
Version history	Append-only list	Snapshot tracking
Change events	Event stream	Broadcasting updates
Priority updates	PriorityQueue	Recalculation scheduling
Formatting styles	Flyweight objects	Memory-efficient reuse
Cell index	TreeMap / TreeSet	Sorted access
Collaboration ops	Queue	OT / CRDT processing
Time-series metrics	Append-only log	Analytics & monitoring

### 7.3 Database Schema

```sql
-- Workbooks table
CREATE TABLE workbooks (
    workbook_id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    owner_id UUID NOT NULL,
    access_level VARCHAR(20) NOT NULL DEFAULT 'PRIVATE',
    version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    FOREIGN KEY (owner_id) REFERENCES users(user_id),
    INDEX idx_owner (owner_id)
);

-- Sheets table
CREATE TABLE sheets (
    sheet_id UUID PRIMARY KEY,
    workbook_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    index INT NOT NULL,
    row_count INT NOT NULL DEFAULT 1000,
    column_count INT NOT NULL DEFAULT 26,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    FOREIGN KEY (workbook_id) REFERENCES workbooks(workbook_id) ON DELETE CASCADE,
    INDEX idx_workbook (workbook_id),
    UNIQUE KEY uk_workbook_index (workbook_id, index)
);

-- MongoDB: Cell data stored as documents
// Collection: sheet_cells
{
  "_id": "sheet_uuid",
  "sheetId": "sheet_uuid",
  "cells": {
    "A1": {
      "value": 100,
      "dataType": "NUMBER",
      "format": {
        "bold": true,
        "fontSize": 12
      },
      "updatedAt": ISODate("2025-11-18T10:00:00Z"),
      "updatedBy": "user_uuid"
    },
    "B1": {
      "value": 200,
      "formula": "=A1*2",
      "dataType": "FORMULA",
      "format": {},
      "updatedAt": ISODate("2025-11-18T10:00:00Z"),
      "updatedBy": "user_uuid"
    }
  },
  "metadata": {
    "cellCount": 2,
    "lastModified": ISODate("2025-11-18T10:00:00Z")
  }
}

-- Versions table (snapshots)
CREATE TABLE workbook_versions (
    version_id UUID PRIMARY KEY,
    workbook_id UUID NOT NULL,
    version_number INT NOT NULL,
    snapshot_location VARCHAR(500) NOT NULL, -- S3 path
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    created_by UUID NOT NULL,
    description TEXT,
    FOREIGN KEY (workbook_id) REFERENCES workbooks(workbook_id) ON DELETE CASCADE,
    FOREIGN KEY (created_by) REFERENCES users(user_id),
    INDEX idx_workbook (workbook_id),
    UNIQUE KEY uk_workbook_version (workbook_id, version_number)
);

-- Permissions table
CREATE TABLE workbook_permissions (
    permission_id UUID PRIMARY KEY,
    workbook_id UUID NOT NULL,
    user_id UUID NOT NULL,
    access_level VARCHAR(20) NOT NULL, -- OWNER, EDITOR, VIEWER
    granted_by UUID NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    FOREIGN KEY (workbook_id) REFERENCES workbooks(workbook_id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (granted_by) REFERENCES users(user_id),
    UNIQUE KEY uk_workbook_user (workbook_id, user_id)
);
```

### 7.4 Scalability Calculations

**User Load:**
- 1M concurrent users
- 10K concurrent editors on popular sheets
- Average 10 operations/minute per active user
- Peak: 100K operations/second

**Data Volume:**
- Average workbook: 10 sheets × 10,000 cells = 100K cells
- Average cell: 100 bytes
- 10MB per workbook
- 1M workbooks = 10TB data

**WebSocket Connections:**
- 1M concurrent connections
- 1000 WS servers, 1000 connections each
- Connection pooling and load balancing

**Formula Calculation:**
- Average formula has 10 dependencies
- Cache results, recalculate only on change
- Parallel calculation for independent formulas

---

## Interview Discussion Points

### Clarifying Questions:
1. Scale: How many concurrent users? Size of spreadsheets?
2. Collaboration: How many simultaneous editors?
3. Formulas: What complexity of formulas to support?
4. Latency: What's acceptable for updates to propagate?
5. Offline: Do we need offline support?
6. Import/Export: Which formats?

### Trade-offs Discussed:
1. **OT vs CRDT**: OT more complex but preserves intent, CRDT simpler but may lose some semantics
2. **Storage**: Relational vs Document DB for cells
3. **Real-time vs Polling**: WebSocket overhead vs simpler HTTP
4. **Formula Calculation**: On-demand vs pre-calculated

### Extensions:
1. **Charts and Visualizations**
2. **Pivot Tables**
3. **Macros/Scripting**
4. **Mobile App**
5. **AI-powered insights**

This design provides a robust foundation for a collaborative spreadsheet application like Excel/Google Sheets.

