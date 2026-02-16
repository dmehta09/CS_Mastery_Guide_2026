# Project 02: Real-Time Collaborative Task Manager

## Difficulty Level: Intermediate
**Estimated Time**: 2-3 weeks per language
**Prerequisites**: WebSockets, Event-Driven Architecture, CQRS basics

---

## What You're Building

A real-time collaborative task management system like Trello or Asana where multiple users can work on the same board simultaneously and see changes instantly. Think Google Docs but for task management.

---

## Layman Explanation

Imagine a **digital whiteboard in a meeting room** where multiple people can:

1. Add sticky notes (tasks) to the board
2. Move notes between columns (To Do, In Progress, Done)
3. Everyone sees changes immediately - no refresh needed
4. If someone moves a note, everyone sees it move in real-time
5. You can see who else is currently looking at the board
6. All changes are saved so you can leave and come back later

The magic: Everyone stays synchronized without hitting "refresh" - just like magic!

---

## Real-World Use Cases

1. **Project Management**: Teams tracking sprints and tasks
2. **Customer Support**: Ticket boards with real-time updates
3. **Content Planning**: Editorial calendars with live collaboration
4. **Sales Pipeline**: CRM boards tracking deals
5. **Event Planning**: Coordinating tasks across team members

---

## Key Concepts You'll Learn

### 1. Real-Time Communication
- **WebSockets**: Bidirectional persistent connections
- **Server-Sent Events (SSE)**: One-way real-time updates
- **Long Polling**: Fallback for older browsers
- **Connection Management**: Handling disconnects/reconnects

### 2. Event-Driven Architecture
- **Domain Events**: TaskCreated, TaskMoved, TaskCompleted
- **Event Bus**: Publishing and subscribing to events
- **Event Sourcing**: Storing all changes as events
- **Event Replay**: Reconstructing state from events

### 3. CQRS (Command Query Responsibility Segregation)
- **Commands**: Write operations (CreateTask, MoveTask)
- **Queries**: Read operations (GetBoard, GetTask)
- **Separate Models**: Different models for reads vs writes
- **Eventual Consistency**: Reads might be slightly behind writes

### 4. Concurrency & Conflicts
- **Optimistic Locking**: Version numbers prevent conflicts
- **Last Write Wins**: Simple conflict resolution
- **Operational Transformation**: Complex but deterministic
- **CRDTs**: Conflict-free replicated data types

### 5. State Management
- **Server-Side State**: Current board state
- **Client-Side State**: Local optimistic updates
- **State Synchronization**: Keeping clients in sync
- **State Snapshots**: Periodic full state sync

### 6. Presence & Awareness
- **Online Presence**: Who's currently viewing the board
- **Cursor Tracking**: See where others are working
- **Real-Time Cursors**: Like Google Docs cursors
- **Heartbeat Mechanism**: Detecting disconnections

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                         WEB CLIENTS                               │
│                   (React, Vue, Angular Apps)                      │
└────────────────┬───────────────────────┬─────────────────────────┘
                 │                       │
          [WebSocket]              [WebSocket]
                 │                       │
                 ▼                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API GATEWAY / LOAD BALANCER                 │
│                  (WebSocket-aware load balancing)                │
└────────────┬───────────────────────────────────┬────────────────┘
             │                                    │
    ┌────────▼────────┐                 ┌────────▼────────┐
    │  APP SERVER 1   │                 │  APP SERVER 2   │
    │  (WebSocket +   │◄────────────────►│  (WebSocket +   │
    │   HTTP API)     │   Pub/Sub       │   HTTP API)     │
    └────────┬────────┘                 └────────┬────────┘
             │                                    │
             └──────────────┬─────────────────────┘
                            │
                            ▼
                  ┌───────────────────┐
                  │   REDIS PUB/SUB   │
                  │  (Event broadcast)│
                  └──────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  COMMAND QUEUE  │
                    │  (RabbitMQ)     │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  EVENT STORE    │
                    │  (PostgreSQL)   │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  READ MODEL     │
                    │  (MongoDB)      │
                    └─────────────────┘
```

---

## Database Schema

### Event Store (PostgreSQL) - Write Side

```sql
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    aggregate_id UUID NOT NULL,
    aggregate_type VARCHAR(50) NOT NULL, -- 'board', 'task', 'list'
    event_type VARCHAR(100) NOT NULL,
    event_data JSONB NOT NULL,
    user_id BIGINT NOT NULL,
    version INT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),

    -- Ensure events are in order for each aggregate
    UNIQUE(aggregate_id, version)
);

CREATE INDEX idx_aggregate_id ON events(aggregate_id);
CREATE INDEX idx_event_type ON events(event_type);
CREATE INDEX idx_created_at ON events(created_at);

-- Example events stored:
/*
{
  "event_type": "TaskCreated",
  "aggregate_id": "board-123",
  "event_data": {
    "task_id": "task-456",
    "title": "Implement WebSocket",
    "list_id": "list-789",
    "position": 0
  },
  "version": 15
}
*/
```

### Read Model (MongoDB) - Query Side

```javascript
// Boards Collection
{
  _id: "board-123",
  name: "Sprint Planning",
  workspace_id: "workspace-1",
  owner_id: 123,
  members: [123, 456, 789],
  lists: [
    {
      id: "list-1",
      name: "To Do",
      position: 0,
      tasks: [
        {
          id: "task-1",
          title: "Implement authentication",
          description: "Add JWT auth",
          assigned_to: 123,
          priority: "high",
          due_date: "2026-03-01",
          position: 0,
          labels: ["backend", "security"],
          comments_count: 3,
          attachments_count: 1
        }
      ]
    },
    {
      id: "list-2",
      name: "In Progress",
      position: 1,
      tasks: [...]
    },
    {
      id: "list-3",
      name: "Done",
      position: 2,
      tasks: [...]
    }
  ],
  version: 42,
  updated_at: "2026-02-16T10:30:00Z"
}

// Presence Collection
{
  board_id: "board-123",
  active_users: [
    {
      user_id: 123,
      username: "Alice",
      cursor_position: { x: 450, y: 230 },
      current_task_id: "task-1",
      last_seen: "2026-02-16T10:30:00Z"
    }
  ]
}
```

### Cache (Redis)

```
# Board state cache
Key: board:{board_id}
Value: JSON (full board state)
TTL: 300 seconds

# Active connections by board
Key: board:{board_id}:connections
Value: SET of connection_ids
TTL: -1 (no expiry)

# User presence
Key: board:{board_id}:user:{user_id}
Value: JSON (cursor, current_task, etc.)
TTL: 30 seconds (with heartbeat)
```

---

## WebSocket Events

### Client → Server (Commands)

```javascript
// 1. Join Board
{
  type: "JOIN_BOARD",
  payload: {
    board_id: "board-123",
    user_id: 123,
    auth_token: "jwt_token"
  }
}

// 2. Create Task
{
  type: "CREATE_TASK",
  payload: {
    board_id: "board-123",
    list_id: "list-1",
    title: "New task",
    description: "Task description",
    position: 0
  }
}

// 3. Move Task
{
  type: "MOVE_TASK",
  payload: {
    task_id: "task-1",
    from_list_id: "list-1",
    to_list_id: "list-2",
    new_position: 1,
    version: 42 // For optimistic locking
  }
}

// 4. Update Task
{
  type: "UPDATE_TASK",
  payload: {
    task_id: "task-1",
    updates: {
      title: "Updated title",
      assigned_to: 456
    },
    version: 43
  }
}

// 5. Cursor Movement
{
  type: "CURSOR_MOVE",
  payload: {
    board_id: "board-123",
    x: 450,
    y: 230
  }
}

// 6. Heartbeat
{
  type: "HEARTBEAT",
  payload: {
    board_id: "board-123"
  }
}
```

### Server → Client (Events)

```javascript
// 1. Board State (Initial sync)
{
  type: "BOARD_STATE",
  payload: {
    board: { /* full board object */ },
    version: 42,
    active_users: [...]
  }
}

// 2. Task Created
{
  type: "TASK_CREATED",
  payload: {
    task: { /* new task */ },
    list_id: "list-1",
    user_id: 123,
    username: "Alice",
    timestamp: "2026-02-16T10:30:00Z"
  }
}

// 3. Task Moved
{
  type: "TASK_MOVED",
  payload: {
    task_id: "task-1",
    from_list_id: "list-1",
    to_list_id: "list-2",
    new_position: 1,
    user_id: 123,
    version: 43
  }
}

// 4. User Joined
{
  type: "USER_JOINED",
  payload: {
    user: {
      user_id: 456,
      username: "Bob",
      avatar: "https://..."
    }
  }
}

// 5. User Left
{
  type: "USER_LEFT",
  payload: {
    user_id: 456
  }
}

// 6. Cursor Update
{
  type: "CURSOR_UPDATE",
  payload: {
    user_id: 123,
    username: "Alice",
    x: 450,
    y: 230
  }
}

// 7. Conflict Error
{
  type: "CONFLICT_ERROR",
  payload: {
    command: "MOVE_TASK",
    reason: "Version mismatch",
    expected_version: 43,
    current_version: 45
  }
}
```

---

## CQRS Implementation

### Command Side (Writes)

```
1. Client sends command (CREATE_TASK)
2. API validates command
3. Load current state from event store
4. Apply business rules
5. Generate domain event (TaskCreated)
6. Save event to event store
7. Publish event to message queue
8. Return success to client (optimistic update)
```

### Query Side (Reads)

```
1. Event handler listens for events
2. When TaskCreated event received
3. Update read model (MongoDB)
4. Broadcast to all connected clients via WebSocket
5. Clients update their UI
```

### Event Flow Example

```
User Action: Move task from "To Do" to "In Progress"

WRITE PATH:
1. Client: WebSocket → MoveTask command
2. Server: Validate (user permissions, task exists)
3. Server: Load current state (version 42)
4. Server: Create event { type: "TaskMoved", version: 43 }
5. Server: Save to event store
6. Server: Respond to client with version 43

READ PATH:
7. Event handler: Pick up TaskMoved event
8. Update MongoDB: Move task in board document
9. Publish to Redis Pub/Sub
10. All servers receive event
11. Broadcast to all WebSocket clients on that board
12. Clients update UI (smooth animation)
```

---

## Handling Conflicts

### Scenario: Two users move same task simultaneously

```
Time  | User A (version 42)    | User B (version 42)
------|------------------------|------------------------
T0    | Move Task 1 to List 2  | Move Task 1 to List 3
T1    | Command received       | Command received
T2    | Version check: OK (42) | Version check: OK (42)
T3    | Create event v43       | Create event v43
T4    | Save to DB: SUCCESS    | Save to DB: FAIL
      |                        | (Unique constraint on version)
T5    | Broadcast TaskMoved    | Return conflict error
T6    |                        | Client gets CONFLICT_ERROR
T7    |                        | Client fetches latest state
T8    |                        | User sees task in List 2
T9    |                        | User can retry move to List 3
```

### Optimistic Locking Implementation

```typescript
// Event Store ensures uniqueness
UNIQUE(aggregate_id, version)

// When saving event
try {
  await db.insert({
    aggregate_id: "board-123",
    version: command.version + 1,
    event_data: {...}
  });
} catch (UniqueConstraintError) {
  // Version mismatch - someone else updated
  throw new ConflictError(
    "Board was modified by another user"
  );
}

// Client handles conflict
socket.on("CONFLICT_ERROR", async () => {
  // Fetch latest board state
  const latest = await fetchBoardState();
  // Update local state
  setState(latest);
  // Show notification
  toast.info("Board was updated by another user");
});
```

---

## WebSocket Connection Management

### Connection Lifecycle

```typescript
// Server-side connection handling
class BoardWebSocketHandler {
  async handleConnection(socket, token) {
    // 1. Authenticate
    const user = await auth.verifyToken(token);

    // 2. Store connection
    connections.set(socket.id, {
      user,
      boards: new Set()
    });

    // 3. Set up event handlers
    socket.on("JOIN_BOARD", (data) =>
      this.handleJoinBoard(socket, data)
    );
    socket.on("CREATE_TASK", (data) =>
      this.handleCreateTask(socket, data)
    );
    socket.on("HEARTBEAT", () =>
      this.handleHeartbeat(socket)
    );
    socket.on("disconnect", () =>
      this.handleDisconnect(socket)
    );
  }

  async handleJoinBoard(socket, { board_id }) {
    // 1. Check permissions
    const hasAccess = await checkBoardAccess(
      socket.user.id,
      board_id
    );
    if (!hasAccess) {
      socket.emit("ERROR", {
        message: "Access denied"
      });
      return;
    }

    // 2. Join room
    socket.join(`board:${board_id}`);
    connections.get(socket.id).boards.add(board_id);

    // 3. Add to active users
    await redis.sadd(
      `board:${board_id}:connections`,
      socket.id
    );

    // 4. Send current board state
    const board = await getBoardState(board_id);
    socket.emit("BOARD_STATE", board);

    // 5. Notify others
    socket.to(`board:${board_id}`).emit("USER_JOINED", {
      user_id: socket.user.id,
      username: socket.user.username
    });
  }

  async handleDisconnect(socket) {
    const connection = connections.get(socket.id);

    // Remove from all boards
    for (const board_id of connection.boards) {
      await redis.srem(
        `board:${board_id}:connections`,
        socket.id
      );

      // Notify others
      socket.to(`board:${board_id}`).emit("USER_LEFT", {
        user_id: connection.user.id
      });
    }

    connections.delete(socket.id);
  }
}
```

### Reconnection Handling

```typescript
// Client-side reconnection
class BoardSocket {
  connect(boardId) {
    this.socket = io(url, {
      auth: { token: getAuthToken() },
      reconnection: true,
      reconnectionDelay: 1000,
      reconnectionAttempts: 5
    });

    this.socket.on("connect", () => {
      // Rejoin board
      this.socket.emit("JOIN_BOARD", {
        board_id: boardId
      });
    });

    this.socket.on("BOARD_STATE", (state) => {
      // Full state sync on reconnect
      this.updateLocalState(state);
    });

    this.socket.on("disconnect", () => {
      // Show offline indicator
      this.setStatus("offline");
    });
  }
}
```

---

## Implementation Steps

### Phase 1: Basic Real-Time (Week 1)
1. Set up WebSocket server
2. Implement connection management
3. Create basic board CRUD (HTTP API)
4. Implement JOIN_BOARD and broadcast
5. Basic task creation with real-time updates
6. Test with 2-3 concurrent users

### Phase 2: CQRS & Event Sourcing (Week 2)
1. Set up event store (PostgreSQL)
2. Implement command handlers
3. Implement event handlers
4. Set up read model (MongoDB)
5. Event replay functionality
6. Optimistic locking

### Phase 3: Advanced Features (Week 3)
1. User presence and cursors
2. Task drag-and-drop with position updates
3. Conflict resolution
4. Offline support (queue commands)
5. Rate limiting per connection
6. Connection pooling

### Phase 4: Production Ready (Week 4)
1. Load balancing for WebSockets (sticky sessions)
2. Horizontal scaling (Redis Pub/Sub)
3. Comprehensive testing
4. Performance optimization
5. Monitoring and metrics
6. Graceful shutdown

---

## Performance Targets

| Metric | Target |
|--------|--------|
| WebSocket message latency | < 100ms |
| Initial board load | < 500ms |
| Concurrent users per board | 50+ |
| Messages per second per server | 10,000+ |
| Memory per connection | < 10KB |
| Reconnection time | < 2s |

---

## Testing Strategy

### Unit Tests
- Command validation
- Event generation
- State reconstruction from events
- Conflict detection

### Integration Tests
- WebSocket connection flow
- Event store persistence
- Read model updates
- Pub/Sub message delivery

### End-to-End Tests
```javascript
test("Two users can collaborate in real-time", async () => {
  // Connect two clients
  const client1 = new WebSocketClient();
  const client2 = new WebSocketClient();

  await client1.connect();
  await client2.connect();

  // Both join same board
  await client1.joinBoard("board-123");
  await client2.joinBoard("board-123");

  // Client 1 creates task
  client1.createTask({ title: "Test task" });

  // Client 2 should receive event
  const event = await client2.waitForEvent("TASK_CREATED");
  expect(event.payload.title).toBe("Test task");
});
```

### Load Tests
```bash
# Simulate 100 concurrent users
artillery run \
  --target wss://localhost:3000 \
  --count 100 \
  websocket-load-test.yml
```

---

## Technology Stack

### NestJS Implementation
- @nestjs/websockets (WebSocket)
- @nestjs/platform-socket.io (Socket.IO)
- @nestjs/event-emitter (Events)
- @nestjs/cqrs (CQRS support)
- TypeORM (Event store)
- Mongoose (Read model)

### Golang Implementation
- gorilla/websocket (WebSocket)
- NATS or RabbitMQ (Event bus)
- PostgreSQL (Event store)
- MongoDB (Read model)
- Redis (Pub/Sub)

### Python Implementation
- FastAPI + WebSockets
- python-socketio
- SQLAlchemy (Event store)
- Motor (Async MongoDB)
- Redis pub/sub

---

## Success Criteria

✅ Support 50+ concurrent users per board
✅ Messages delivered in < 100ms
✅ Zero message loss
✅ Graceful handling of disconnects
✅ Conflict resolution working
✅ Event sourcing with replay
✅ Comprehensive test coverage
✅ Monitoring and alerting set up

---

## Next Project

After completing this, move to **Project 03: Multi-Tenant SaaS API Gateway** to learn about rate limiting, authentication, and multi-tenancy at scale.

---

Happy Building! 🚀
