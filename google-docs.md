# Google Docs System Design

## Overview

Design a browser-based collaborative document editor supporting real-time multi-user editing.

## Requirements

### Functional

- Create new documents
- Concurrent editing by multiple users
- Real-time change synchronization
- Cursor position & presence awareness

### Non-Functional

- Eventual consistency across all clients
- Low latency updates (< 100ms)
- Scale to millions of users and billions of documents
- Max 100 concurrent editors per document
- Durability and availability

## Core Entities

- **Editor**: User editing a document
- **Document**: Text collection managed by editors
- **Edit**: Change made to document
- **Cursor**: Editor's cursor position and presence

## API Design

### REST Endpoints

```
POST /docs { title: string } -> { docId }
```

### WebSocket Messages

```
WS /docs/{docId}

// Client sends
SEND { type: "insert", position, text, version }
SEND { type: "delete", position, length, version }
SEND { type: "updateCursor", position, selection }

// Client receives
RECV { type: "update", operations, version }
RECV { type: "cursorUpdate", userId, position, color }
```

## High-Level Design

### 1. Document Creation

- API Gateway → Document Service → Postgres DB
- Store document metadata (title, permissions, etc.)
- Use horizontally scaled CRUD service

### 2. Collaborative Editing

**Problem**: Multiple users making concurrent edits creates consistency issues.

**Solutions Considered**:

- ❌ Sending snapshots: Inefficient, loses concurrent edits
- ❌ Sending raw edits: Context-dependent, causes conflicts
- ✅ **Operational Transformation (OT)**: Transforms operations to ensure consistency

**Architecture**:

- Client sends operations (INSERT/DELETE) to Document Service
- Server applies OT to resolve conflicts
- Operations stored in Cassandra (partitioned by docId, ordered by timestamp)
- Server broadcasts transformed operations to all connected clients

**Why OT over CRDTs?**

- Lower memory usage
- Better for text editing
- Works well with centralized server
- (CRDTs better for P2P or massive collaborator count)

### 3. Real-Time Updates

**On Document Load**:

- Client connects via WebSocket
- Server pushes all previous operations
- Client applies operations to build current state

**On Updates**:

- Editor makes change → local immediate update
- Operation sent to server → OT applied → stored in DB
- Server broadcasts to all other clients
- Clients apply OT to maintain consistency

**Why client-side OT?**

User edits are applied locally first for responsiveness. If another user's edit arrives before ours is confirmed, OT ensures consistent ordering regardless of arrival sequence.

### 4. Cursor & Presence

**Properties**:

- Ephemeral (only matters while connected)
- No need for persistent storage

**Implementation**:

- Store cursor positions in-memory on Document Service
- Broadcast cursor updates via WebSocket
- On new connection: send all active cursors
- On disconnect: remove cursor, broadcast to others

## Scaling Deep Dives

### 1. Scaling WebSocket Connections

**Challenge**: Single server can't handle millions of connections.

**Solution: Consistent Hashing**

- Use Apache ZooKeeper to maintain hash ring
- Each Document Service server handles specific document ID ranges
- Client connection flow:
  1. Connect to any server (round robin)
  2. Server checks hash ring for document ID
  3. If wrong server, redirect to correct one
  4. Upgrade to WebSocket on correct server
  5. Load operations from DB if needed

**Benefits**:

- All editors for same document on same server
- Adding/removing servers only redistributes affected connections
- No cross-server coordination needed for broadcasts

**Tradeoffs**:

- Complex scaling events (connection migration)
- Need robust reconnection logic
- Server failures require quick redistribution

### 2. Storage Optimization

**Challenge**: Billions of documents × operations = massive storage and processing overhead.

**Solution: Operation Compaction**

Options:

1. **Periodic Snapshots**: Merge N operations into snapshot, delete old ops
2. **Background Compaction**: Async job compacts inactive documents
3. **Hybrid**: Recent ops + snapshot for faster loading

**Benefits**:

- Reduces storage
- Faster document loading
- Lower memory usage in Document Service

## Key Learnings & Insights

### Design Patterns

**Real-Time Updates Pattern**:
- WebSockets are bi-directional - think about BOTH left-to-right (client→server) AND right-to-left (server→client) arrows
- Handle WebSockets at the "edge" of your design to keep internal services stateless
- Expose internal API so other services don't need to know about WebSocket connection details

**Collaborative Editing Evolution**:
1. **Sending Snapshots** ❌
   - Problem: 100s of KB per keystroke, last-write-wins loses concurrent edits
   - Example: "Hello!" → User A adds ", world", User B deletes "!" → One change lost

2. **Sending Edits** ⚠️
   - Problem: Operations are context-dependent
   - Example: DELETE(6) might delete wrong character if another INSERT happens first

3. **Operational Transformation** ✅
   - Solution: Transform operations based on document state
   - Guarantees: Same final state regardless of operation order
   - Where: BOTH server-side (for persistence) AND client-side (for responsiveness)

**Why Client-Side OT Matters**:
- Users expect immediate local feedback
- Local edits applied first, then sent to server
- If another edit arrives during transit, client must transform to maintain consistency
- Example ordering:
  - Server sees: Ea, Eb
  - User A sees: Ea, Eb  
  - User B sees: Eb, Ea
  - Result: All converge to same document state!

### Consistency vs Performance Trade-offs

**100 Concurrent Editor Limit**:
- This is a deliberate constraint (Google Docs does this too!)
- Beyond limit: new users join as read-only
- Why: Avoids massive throughput issues on single document
- Lets us focus on consistency over extreme scale per document

**Eventual Consistency is OK**:
- Users don't need instant global consistency
- < 100ms latency feels real-time to humans
- OT guarantees eventual convergence

### Scaling Insights

**Consistent Hashing Benefits**:
- All collaborators on same document connect to same server
- No cross-server coordination needed for broadcasts
- Adding/removing servers only affects small portion of connections
- Alternative considered: Service mesh with message passing (more complex)

**State Management**:
- WebSockets are stateful - this is unavoidable
- Minimize state transfer during scaling events
- Client reconnection logic is critical
- Monitor connection distribution to prevent hotspots

**Storage Growth**:
- Billions of documents × thousands of operations = unsustainable
- Solution: Periodic compaction/snapshots
- Trade-off: Lose fine-grained history vs. manageable storage
- For versioning: Keep snapshots at intervals (hourly/daily)

### Architecture Principles

**Separation of Concerns**:
- Document metadata (Postgres) vs. Operations (Cassandra)
- Different access patterns, different databases
- Metadata: Flexible queries, moderate writes
- Operations: Append-only, high write throughput

**Ephemeral vs Persistent Data**:
- Cursor positions: Ephemeral, in-memory only
- Document edits: Persistent, durability required
- Don't persist what you don't need!

**Read vs Write Paths**:
- Read-only mode can scale to millions (CDN + snapshots)
- Write path limited to 100 concurrent users
- Design differently based on access patterns

### Interview Tips

**Common Pitfalls**:
- Not thinking about right-to-left data flow (server pushing to clients)
- Assuming single server can handle all WebSocket connections
- Forgetting client-side transformation needs
- Overlooking storage growth over time

**What Interviewers Look For**:
- Understanding of WebSocket statefulness challenges
- Knowledge of collaborative editing algorithms (OT vs CRDT)
- Ability to identify scaling bottlenecks
- Separation of concerns (metadata vs operations)
- Read vs write path optimization

**When to Deep Dive**:
- Start simple, then iterate on non-functional requirements
- Acknowledge trade-offs explicitly
- Note what you're deferring for later discussion
- Use constraints (like 100 user limit) to simplify problem space

## Additional Considerations

- **Read-Only Mode**: Cache snapshots at CDN, skip OT for viewers
- **Versioning**: Keep snapshot history with timestamps
- **Offline Mode**: Queue operations locally, sync on reconnect with conflict resolution
- **Memory Optimization**: LRU cache for active documents, evict inactive ones

## Technology Choices

- **Metadata DB**: Postgres (flexible querying, scalable with sharding)
- **Operations DB**: Cassandra (fast append-only writes, partition by docId)
- **Coordination**: Apache ZooKeeper (hash ring management)
- **Protocol**: WebSocket (bi-directional, low latency)
- **Consistency**: Operational Transformation (proven for text editing)
