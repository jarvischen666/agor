# Agor Session Design Analysis for Python Backend Conversion

**Date:** 2025-01-12
**Purpose:** Comprehensive backend architecture analysis for Python conversion
**Scope:** Agentic calling, Session API, Message interruption/resumption

---

## Table of Contents

1. [Agentic Calling Architecture](#1-agentic-calling-architecture)
2. [Session API Design](#2-session-api-design)
3. [Message Interruption and Resumption](#3-message-interruption-and-resumption)
4. [Python Conversion Guidelines](#4-python-conversion-guidelines)
5. [Key Files Reference](#5-key-files-reference)

---

## 1. Agentic Calling Architecture

### 1.1 Multi-Layer Agent Integration

Agor implements a **layered abstraction** over agent SDKs (Claude/Codex/Gemini):

```
┌─────────────────────────────────────────┐
│   Application Layer                     │
│   FeathersJS Services + WebSocket       │
│   - REST endpoints                      │
│   - WebSocket broadcasting              │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│   ClaudeTool (Agent Wrapper)            │
│   Location: core/tools/claude/index.ts  │
│   - executePromptWithStreaming()        │
│   - stopTask()                          │
│   - Permission enforcement              │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│   ClaudePromptService                   │
│   Location: core/tools/claude/          │
│            prompt-service.ts            │
│   - promptSessionStreaming()            │
│   - Active query tracking               │
│   - Lifecycle management                │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│   Query Builder                         │
│   Location: core/tools/claude/          │
│            query-builder.ts             │
│   - setupQuery()                        │
│   - MCP server injection                │
│   - Environment resolution              │
│   - Resume/Fork/Spawn logic             │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│   Claude Agent SDK                      │
│   - query() async generator             │
│   - Streaming events                    │
│   - Tool execution                      │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│   SDKMessageProcessor                   │
│   Location: core/tools/claude/          │
│            message-processor.ts         │
│   - process(msg) → ProcessedEvent[]     │
│   - Stream aggregation                  │
│   - Token tracking                      │
└─────────────────────────────────────────┘
```

### 1.2 Message Flow (Event-Driven Architecture)

**Core file:** `packages/core/src/tools/claude/prompt-service.ts`

The system uses **async generators** for streaming:

```typescript
async function* promptSessionStreaming(
  sessionId: SessionID,
  prompt: string,
  options?: PromptOptions
): AsyncGenerator<ProcessedEvent> {

  // 1. Setup query with session context
  const { query: result } = await setupQuery(
    sessionId,
    prompt,
    deps,
    options
  );

  // 2. Store query for interruption
  this.activeQueries.set(sessionId, result);

  // 3. Iterate through streaming events
  for await (const msg of result) {

    // Check for interruption
    if (this.stopRequested.get(sessionId)) {
      console.log('Stop requested, breaking event loop');
      this.stopRequested.delete(sessionId);
      break;
    }

    // 4. Process SDK message into structured events
    const events = await processor.process(msg);

    // 5. Yield each event
    for (const event of events) {
      yield event;
    }
  }

  // 6. Cleanup
  this.activeQueries.delete(sessionId);
}
```

### 1.3 Event Types

**Location:** `packages/core/src/tools/claude/message-processor.ts`

```typescript
export type ProcessedEvent =
  | PartialEvent              // Token-level streaming chunk
  | ThinkingPartialEvent      // Thinking block chunk
  | CompleteEvent             // Full message (persist to DB)
  | ToolStartEvent            // Tool execution started
  | ToolCompleteEvent         // Tool execution finished
  | SessionIdCapturedEvent    // SDK session ID (for resume)
  | ResultEvent               // Token usage & cost metadata
  | EndEvent;                 // Conversation complete

// Example: PartialEvent
interface PartialEvent {
  type: 'partial';
  textChunk: string;          // Incremental text token
  timestamp: number;
}

// Example: CompleteEvent (triggers DB save)
interface CompleteEvent {
  type: 'complete';
  role: 'assistant' | 'user' | 'system';
  content: MessageContent[];  // Text + thinking blocks
  metadata: {
    model?: string;
    stop_reason?: 'end_turn' | 'tool_use' | 'interrupted';
  };
}

// Example: ToolStartEvent
interface ToolStartEvent {
  type: 'tool_start';
  tool_use_id: string;
  tool_name: string;
  input: unknown;
}
```

### 1.4 Stream Processor State Machine

**Core class:** `SDKMessageProcessor`

```typescript
export class SDKMessageProcessor {
  private state: ProcessorState;

  constructor(sessionId: SessionID, options: ProcessorOptions) {
    this.state = {
      sessionId,
      messageCount: 0,
      lastActivityTime: Date.now(),

      // Streaming state
      currentTextBuffer: '',
      currentThinkingBuffer: '',
      textStreamId: null,
      thinkingStreamId: null,

      // Tool tracking
      toolsInProgress: new Map(),

      // SDK session tracking
      capturedSdkSessionId: null,
    };
  }

  async process(msg: SDKMessage): Promise<ProcessedEvent[]> {
    this.state.messageCount += 1;
    this.state.lastActivityTime = Date.now();

    // Route to handler based on message type
    switch (msg.type) {
      case 'assistant':
        return this.handleAssistant(msg);

      case 'user':
        return this.handleUser(msg);

      case 'stream_event':
        return this.handleStreamEvent(msg);

      case 'result':
        return this.handleResult(msg);

      case 'system':
        return this.handleSystem(msg);

      default:
        return [];
    }
  }

  private handleStreamEvent(msg: StreamEventMessage): ProcessedEvent[] {
    const events: ProcessedEvent[] = [];

    // Handle text streaming
    if (msg.content_block?.type === 'text') {
      if (msg.event === 'content_block_start') {
        this.state.textStreamId = generateId();
        this.state.currentTextBuffer = '';
      }
      else if (msg.event === 'content_block_delta') {
        const delta = msg.delta?.text || '';
        this.state.currentTextBuffer += delta;
        events.push({
          type: 'partial',
          textChunk: delta,
          timestamp: Date.now()
        });
      }
    }

    // Handle thinking streaming
    if (msg.content_block?.type === 'thinking') {
      if (msg.event === 'content_block_start') {
        this.state.thinkingStreamId = generateId();
        this.state.currentThinkingBuffer = '';
      }
      else if (msg.event === 'content_block_delta') {
        const delta = msg.delta?.thinking || '';
        this.state.currentThinkingBuffer += delta;
        events.push({
          type: 'thinking_partial',
          thinkingChunk: delta,
          timestamp: Date.now()
        });
      }
    }

    // Handle tool use
    if (msg.content_block?.type === 'tool_use') {
      if (msg.event === 'content_block_start') {
        events.push({
          type: 'tool_start',
          tool_use_id: msg.content_block.id,
          tool_name: msg.content_block.name,
          input: msg.content_block.input,
        });
      }
    }

    return events;
  }

  private handleAssistant(msg: AssistantMessage): ProcessedEvent[] {
    // Full assistant message = save to DB
    return [{
      type: 'complete',
      role: 'assistant',
      content: msg.content,
      metadata: {
        model: msg.model,
        stop_reason: msg.stop_reason,
      }
    }];
  }

  private handleResult(msg: ResultMessage): ProcessedEvent[] {
    // Token usage & cost tracking
    return [{
      type: 'result',
      usage: msg.usage,
      cost_usd: msg.cost_usd,
      duration_ms: msg.duration_ms,
    }];
  }
}
```

### 1.5 Dual-Stream Architecture (Text + Thinking)

Agor implements **parallel streaming** for text and thinking blocks:

```typescript
// Stream Separation Pattern
let currentTextMessageId: MessageID | null = null;
let currentThinkingMessageId: MessageID | null = null;

for await (const event of promptService.promptSessionStreaming(...)) {

  // Thinking stream (if extended thinking enabled)
  if (event.type === 'thinking_partial') {
    if (!currentThinkingMessageId) {
      currentThinkingMessageId = generateId();
      streamingCallbacks.onThinkingStart(currentThinkingMessageId, metadata);
    }
    streamingCallbacks.onThinkingChunk(currentThinkingMessageId, event.thinkingChunk);
  }

  // Text stream
  if (event.type === 'partial') {
    if (!currentTextMessageId) {
      currentTextMessageId = generateId();
      streamingCallbacks.onStreamStart(currentTextMessageId, metadata);
    }
    streamingCallbacks.onStreamChunk(currentTextMessageId, event.textChunk);
  }

  // Merge both streams into single DB message
  if (event.type === 'complete') {
    const messageId = currentTextMessageId || currentThinkingMessageId || generateId();
    await createAssistantMessage(sessionId, messageId, event.content, ...);

    // Reset stream IDs
    currentTextMessageId = null;
    currentThinkingMessageId = null;
  }
}
```

---

## 2. Session API Design

### 2.1 REST Endpoints (FeathersJS Pattern)

**Service file:** `apps/agor-daemon/src/services/sessions.ts`

FeathersJS auto-generates CRUD endpoints:

```typescript
// Standard REST Endpoints
GET    /sessions              // List sessions (paginated, filtered)
GET    /sessions/:id          // Get session by ID
POST   /sessions              // Create session
PATCH  /sessions/:id          // Update session (partial)
DELETE /sessions/:id          // Delete session (cascades to children)

// Custom Method Endpoints
POST   /sessions/:id/fork     // Fork session at decision point
POST   /sessions/:id/spawn    // Spawn child session
GET    /sessions/:id/genealogy // Get ancestor/descendant tree
POST   /sessions/:id/resume   // Resume interrupted session
GET    /sessions/:id/context  // Get session context (files, MCP, git)
```

**WebSocket Events** (auto-broadcast):
```typescript
// Client subscribes to 'sessions' service
socket.on('sessions created', (session) => { /* ... */ });
socket.on('sessions patched', (session) => { /* ... */ });
socket.on('sessions removed', (session) => { /* ... */ });
```

### 2.2 Database Schema (Hybrid Materialization)

**Schema file:** `packages/core/src/db/schema.ts`

**Strategy:** Materialized columns for queries + JSON blob for flexibility

```typescript
export const sessions = sqliteTable('sessions', {
  // Primary Key
  session_id: text('session_id').primaryKey(),  // UUIDv7

  // ===== MATERIALIZED COLUMNS (indexed for queries) =====
  status: text('status', {
    enum: ['idle', 'running', 'awaiting_permission', 'completed', 'failed']
  }).notNull(),

  agentic_tool: text('agentic_tool', {
    enum: ['claude-code', 'codex', 'gemini', 'opencode']
  }).notNull(),

  // Foreign Keys (materialized for joins)
  board_id: text('board_id').references(() => boards.board_id, {
    onDelete: 'set null'
  }),
  worktree_id: text('worktree_id').references(() => worktrees.worktree_id, {
    onDelete: 'set null'
  }),
  repo_id: text('repo_id').references(() => repos.repo_id, {
    onDelete: 'cascade'
  }),
  created_by: text('created_by').references(() => users.user_id, {
    onDelete: 'set null'
  }),

  // Genealogy (materialized for tree queries)
  parent_session_id: text('parent_session_id'),  // Spawn parent
  forked_from_session_id: text('forked_from_session_id'),  // Fork parent

  // Metadata
  title: text('title'),
  message_count: integer('message_count').default(0),
  created_at: integer('created_at', { mode: 'timestamp_ms' }).notNull(),
  last_updated: integer('last_updated', { mode: 'timestamp_ms' }).notNull(),

  // ===== JSON BLOB (flexible schema) =====
  data: json('data').$type<{
    // SDK session ID (for conversation continuity)
    sdk_session_id?: string;

    // MCP auth token
    mcp_token?: string;

    // Git state
    git_state: {
      ref: string;         // Current branch/ref
      base_sha: string;    // Base commit SHA
      current_sha: string; // Current commit SHA
    };

    // Genealogy metadata
    genealogy: {
      children: SessionID[];
      fork_point_task_id?: TaskID;
      fork_point_message_index?: number;
      spawn_point_task_id?: TaskID;
    };

    // Context
    context_files?: string[];           // Paths to concept files
    mcp_servers?: MCPServerConfig[];    // MCP server selection

    // Tasks
    tasks: TaskID[];
    active_task_id?: TaskID;

    // Configuration
    permission_config?: {
      mode: 'auto' | 'manual' | 'reject';
      codex?: { action_threshold: number };
    };

    model_config?: {
      mode: 'auto' | 'manual';
      model?: string;
      thinkingMode?: {
        mode: 'auto' | 'manual' | 'off';
        budget?: number;  // 0-31999
      };
    };

    // Custom context (user-provided)
    custom_context?: string;
  }>(),
});

// Indexes
export const sessionsByStatus = index('sessions_by_status').on(sessions.status);
export const sessionsByWorktree = index('sessions_by_worktree').on(sessions.worktree_id);
export const sessionsByParent = index('sessions_by_parent').on(sessions.parent_session_id);
export const sessionsByFork = index('sessions_by_fork').on(sessions.forked_from_session_id);
```

### 2.3 Key Session Operations

#### 2.3.1 Create Session

```typescript
// POST /sessions
const session = await sessionsService.create({
  agentic_tool: 'claude-code',
  status: 'idle',

  // REQUIRED: Session must belong to worktree
  worktree_id: worktreeId,

  // Optional
  board_id: boardId,
  repo_id: repoId,
  created_by: userId,
  title: 'New feature implementation',

  // JSON data
  git_state: {
    ref: 'main',
    base_sha: 'abc123',
    current_sha: 'abc123',
  },
  genealogy: {
    children: [],
  },
  tasks: [],
  context_files: ['context/concepts/core.md'],
  mcp_servers: [{ name: 'filesystem', enabled: true }],
  permission_config: { mode: 'auto' },
  model_config: {
    mode: 'auto',
    thinkingMode: { mode: 'auto' },
  },
});

// WebSocket broadcast: 'sessions created'
```

#### 2.3.2 Fork Session (Divergent Exploration)

**Use case:** Explore alternative approach from decision point

```typescript
// POST /sessions/:id/fork
const forkedSession = await sessionsService.fork(parentSessionId, {
  prompt: "Try using React Query instead of Redux",
  task_id: forkPointTaskId,  // Task where fork occurs
});

// Creates new session with:
{
  session_id: generateId(),  // New UUIDv7
  agentic_tool: parentSession.agentic_tool,
  status: 'idle',

  // Genealogy
  forked_from_session_id: parentSessionId,
  genealogy: {
    children: [],
    fork_point_task_id: forkPointTaskId,
    fork_point_message_index: parentSession.message_count,
  },

  // Inherit context
  worktree_id: parentSession.worktree_id,  // Same worktree
  git_state: parentSession.git_state,      // Same git state
  context_files: parentSession.context_files,
  mcp_servers: parentSession.mcp_servers,

  // Fresh conversation (SDK will fork from parent history)
  sdk_session_id: undefined,  // Will be set by SDK
  tasks: [],
  message_count: 0,
}

// Update parent session
await sessionsService.patch(parentSessionId, {
  genealogy: {
    ...parentSession.genealogy,
    children: [...parentSession.genealogy.children, forkedSession.session_id],
  },
});
```

**SDK Resume Logic:**
```typescript
// query-builder.ts
if (session.forked_from_session_id && !session.sdk_session_id) {
  const parentSession = await deps.sessionsRepo.findById(session.forked_from_session_id);
  if (parentSession?.sdk_session_id) {
    queryOptions.resume = parentSession.sdk_session_id;  // Resume from parent
    queryOptions.forkSession = true;  // SDK creates new session ID
  }
}
```

#### 2.3.3 Spawn Session (Delegate Subsession)

**Use case:** Delegate subtask to child session

```typescript
// POST /sessions/:id/spawn
const spawnedSession = await sessionsService.spawn(parentSessionId, {
  prompt: "Implement authentication module with JWT",
  title: "Auth subsession",
  agentic_tool: 'codex',  // Optional: use different agent
});

// Creates child session with:
{
  session_id: generateId(),
  agentic_tool: 'codex',  // Can differ from parent
  status: 'idle',

  // Genealogy
  parent_session_id: parentSessionId,
  genealogy: {
    children: [],
    spawn_point_task_id: activeTaskId,
  },

  // Inherit context
  worktree_id: parentSession.worktree_id,
  git_state: parentSession.git_state,
  context_files: parentSession.context_files,

  // Fresh conversation (no history)
  sdk_session_id: undefined,
  tasks: [],
  message_count: 0,
}

// Update parent session
await sessionsService.patch(parentSessionId, {
  genealogy: {
    ...parentSession.genealogy,
    children: [...parentSession.genealogy.children, spawnedSession.session_id],
  },
});
```

#### 2.3.4 Resume Session (Conversation Continuity)

**Use case:** Continue interrupted session

```typescript
// Handled automatically in query-builder.ts
export async function setupQuery(sessionId, prompt, deps, options) {
  const session = await deps.sessionsRepo.findById(sessionId);

  // CASE 1: Fork/Spawn on first prompt
  const parentSessionId =
    session.genealogy?.forked_from_session_id ||
    session.genealogy?.parent_session_id;

  if (parentSessionId && !session.sdk_session_id) {
    const parentSession = await deps.sessionsRepo.findById(parentSessionId);
    if (parentSession?.sdk_session_id) {
      queryOptions.resume = parentSession.sdk_session_id;
      queryOptions.forkSession = true;  // SDK creates new session ID
    }
  }

  // CASE 2: Normal resume
  else if (session.sdk_session_id) {
    // Check if session is stale (>24h old, no worktree)
    const hoursSinceUpdate = (Date.now() - new Date(session.last_updated).getTime()) / 3600000;
    const hasWorktree = !!session.worktree_id;

    if (hoursSinceUpdate > 24 || !hasWorktree) {
      // Clear stale session ID, start fresh
      console.log('Clearing stale SDK session ID');
      await deps.sessionsRepo.update(sessionId, {
        data: {
          ...session.data,
          sdk_session_id: undefined
        }
      });
    } else {
      // Resume from SDK session
      queryOptions.resume = session.sdk_session_id;
    }
  }

  // CASE 3: Fresh session (no sdk_session_id)
  // SDK will start fresh and return new session ID via SessionIdCapturedEvent

  return { query: result, processor };
}
```

### 2.4 Repository Pattern (Data Access Layer)

**File:** `packages/core/src/db/repositories/sessions.ts`

```typescript
export class SessionRepository {
  constructor(private db: Database) {}

  // CRUD Operations
  async findById(sessionId: SessionID): Promise<Session | null> {
    return this.db.query.sessions.findFirst({
      where: eq(schema.sessions.session_id, sessionId),
    });
  }

  async findAll(filters?: SessionFilters): Promise<Session[]> {
    let query = this.db.select().from(schema.sessions);

    if (filters?.status) {
      query = query.where(eq(schema.sessions.status, filters.status));
    }
    if (filters?.worktree_id) {
      query = query.where(eq(schema.sessions.worktree_id, filters.worktree_id));
    }

    return query;
  }

  async create(data: CreateSessionData): Promise<Session> {
    const sessionId = generateId();
    const now = new Date();

    await this.db.insert(schema.sessions).values({
      session_id: sessionId,
      ...data,
      created_at: now,
      last_updated: now,
    });

    return this.findById(sessionId);
  }

  async update(sessionId: SessionID, data: Partial<Session>): Promise<Session> {
    await this.db.update(schema.sessions)
      .set({ ...data, last_updated: new Date() })
      .where(eq(schema.sessions.session_id, sessionId));

    return this.findById(sessionId);
  }

  async delete(sessionId: SessionID): Promise<void> {
    // Cascade delete children
    const children = await this.findChildren(sessionId);
    for (const child of children) {
      await this.delete(child.session_id);
    }

    // Delete session
    await this.db.delete(schema.sessions)
      .where(eq(schema.sessions.session_id, sessionId));
  }

  // Genealogy Operations
  async findChildren(sessionId: SessionID): Promise<Session[]> {
    return this.db.select().from(schema.sessions).where(
      or(
        eq(schema.sessions.parent_session_id, sessionId),
        eq(schema.sessions.forked_from_session_id, sessionId)
      )
    );
  }

  async findAncestors(sessionId: SessionID): Promise<Session[]> {
    const ancestors: Session[] = [];
    let current = await this.findById(sessionId);

    while (current) {
      const parentId = current.parent_session_id || current.forked_from_session_id;
      if (!parentId) break;

      current = await this.findById(parentId);
      if (current) ancestors.push(current);
    }

    return ancestors;
  }

  async getGenealogy(sessionId: SessionID): Promise<SessionGenealogy> {
    const session = await this.findById(sessionId);
    const ancestors = await this.findAncestors(sessionId);
    const descendants = await this.findDescendantsRecursive(sessionId);

    return {
      session,
      ancestors,
      descendants,
      depth: ancestors.length,
      total_descendants: descendants.length,
    };
  }

  private async findDescendantsRecursive(sessionId: SessionID): Promise<Session[]> {
    const children = await this.findChildren(sessionId);
    const descendants = [...children];

    for (const child of children) {
      const childDescendants = await this.findDescendantsRecursive(child.session_id);
      descendants.push(...childDescendants);
    }

    return descendants;
  }
}
```

---

## 3. Message Interruption and Resumption

### 3.1 Interruption Mechanism

**Core file:** `packages/core/src/tools/claude/prompt-service.ts`

Uses **Claude SDK's native `interrupt()` method** (same as Escape key in CLI):

```typescript
export class ClaudePromptService {
  // Store active queries for interruption
  private activeQueries = new Map<SessionID, Query>();
  private stopRequested = new Map<SessionID, boolean>();

  async stopTask(sessionId: SessionID): Promise<StopResult> {
    const queryObj = this.activeQueries.get(sessionId);

    if (!queryObj) {
      return {
        success: false,
        reason: 'No active task for this session'
      };
    }

    try {
      console.log(`Stopping task for session ${sessionId}`);

      // 1. Set stop flag for immediate loop breaking
      this.stopRequested.set(sessionId, true);

      // 2. Call SDK's native interrupt() method
      //    This gracefully stops the agent (same as ESC key)
      await queryObj.interrupt();

      // 3. Clean up query reference
      this.activeQueries.delete(sessionId);

      console.log(`Task stopped successfully for session ${sessionId}`);
      return { success: true };

    } catch (error) {
      console.error('Error stopping task:', error);
      this.stopRequested.delete(sessionId);
      return {
        success: false,
        reason: error instanceof Error ? error.message : 'Unknown error'
      };
    }
  }
}
```

**Event loop checking:**
```typescript
async function* promptSessionStreaming(
  sessionId: SessionID,
  prompt: string,
  options?: PromptOptions
): AsyncGenerator<ProcessedEvent> {

  const { query: result, processor } = await setupQuery(...);

  // Store query for interruption
  this.activeQueries.set(sessionId, result);

  try {
    for await (const msg of result) {

      // ===== CHECK FOR INTERRUPTION =====
      if (this.stopRequested.get(sessionId)) {
        console.log('Stop requested, breaking event loop');
        this.stopRequested.delete(sessionId);
        break;  // Exit gracefully
      }

      // Process message
      const events = await processor.process(msg);
      for (const event of events) {
        yield event;
      }
    }
  } finally {
    // Always cleanup
    this.activeQueries.delete(sessionId);
    this.stopRequested.delete(sessionId);
  }
}
```

### 3.2 Session State Persistence

**Three-tier persistence strategy:**

#### Tier 1: Session Metadata (sessions table)

```typescript
{
  session_id: 'abc123',
  status: 'idle' | 'running' | 'awaiting_permission' | 'completed' | 'failed',

  // SDK session ID (for resume)
  data: {
    sdk_session_id: 'sdk_abc123',  // Claude SDK session ID

    // Git state (for resume)
    git_state: {
      ref: 'main',
      base_sha: 'abc123',
      current_sha: 'def456',
    },

    // Tasks
    tasks: ['task1', 'task2'],
    active_task_id: 'task2',
  },

  // Message count (for indexing)
  message_count: 42,

  last_updated: '2025-01-12T10:00:00Z',
}
```

#### Tier 2: Messages (messages table)

**Schema:** `packages/core/src/db/schema.ts`

```typescript
export const messages = sqliteTable('messages', {
  message_id: text('message_id').primaryKey(),
  session_id: text('session_id').references(() => sessions.session_id, {
    onDelete: 'cascade'
  }).notNull(),
  task_id: text('task_id').references(() => tasks.task_id, {
    onDelete: 'set null'
  }),

  // Ordering (within session)
  index: integer('index').notNull(),  // 0, 1, 2, ...

  // Content
  role: text('role', {
    enum: ['user', 'assistant', 'system']
  }).notNull(),

  content: json('content').$type<MessageContent[]>(),  // Text + thinking blocks + tool uses

  // Metadata
  created_at: integer('created_at', { mode: 'timestamp_ms' }).notNull(),

  // Tool usage
  tools_used: json('tools_used').$type<string[]>(),  // ['Read', 'Edit', 'Bash']

  // Token tracking
  token_info: json('token_info').$type<{
    input_tokens?: number;
    output_tokens?: number;
    thinking_tokens?: number;
  }>(),
});

// Composite index for fast session queries
export const messagesBySession = index('messages_by_session_index')
  .on(messages.session_id, messages.index);
```

**Message content structure:**
```typescript
type MessageContent =
  | { type: 'text', text: string }
  | { type: 'thinking', thinking: string }
  | { type: 'tool_use', id: string, name: string, input: unknown }
  | { type: 'tool_result', tool_use_id: string, content: unknown };

// Example assistant message
{
  message_id: 'msg123',
  session_id: 'ses123',
  task_id: 'task123',
  index: 5,
  role: 'assistant',
  content: [
    { type: 'thinking', thinking: 'Let me analyze the requirements...' },
    { type: 'text', text: 'I will implement the feature using...' },
    { type: 'tool_use', id: 'tool1', name: 'Read', input: { file_path: '...' } },
  ],
  tools_used: ['Read'],
  token_info: {
    input_tokens: 1000,
    output_tokens: 500,
    thinking_tokens: 300,
  },
  created_at: '2025-01-12T10:00:00Z',
}
```

#### Tier 3: Tasks (tasks table)

**Schema:** `packages/core/src/db/schema.ts`

```typescript
export const tasks = sqliteTable('tasks', {
  task_id: text('task_id').primaryKey(),
  session_id: text('session_id').references(() => sessions.session_id, {
    onDelete: 'cascade'
  }).notNull(),

  // Task content
  prompt: text('prompt').notNull(),
  title: text('title'),  // Auto-generated or user-provided

  // Status
  status: text('status', {
    enum: ['pending', 'running', 'completed', 'failed', 'stopped']
  }).notNull(),

  // Message range (for resume)
  message_range: json('message_range').$type<{
    start_index: number;
    end_index: number | null;  // null = still running
  }>(),

  // Git state (before/after)
  git_state: json('git_state').$type<{
    sha_at_start: string;
    sha_at_end: string | null;  // null = still running
    files_changed?: string[];
  }>(),

  // Token usage
  token_usage: json('token_usage').$type<{
    input_tokens: number;
    output_tokens: number;
    thinking_tokens?: number;
  }>(),

  // Cost
  cost_usd: real('cost_usd'),

  // Duration
  started_at: integer('started_at', { mode: 'timestamp_ms' }),
  completed_at: integer('completed_at', { mode: 'timestamp_ms' }),
  duration_ms: integer('duration_ms'),

  created_at: integer('created_at', { mode: 'timestamp_ms' }).notNull(),
});
```

### 3.3 Resumption Flow

**Core file:** `packages/core/src/tools/claude/query-builder.ts`

```typescript
export async function setupQuery(
  sessionId: SessionID,
  prompt: string,
  deps: Dependencies,
  options?: QueryOptions
): Promise<{ query: AsyncGenerator, processor: SDKMessageProcessor }> {

  const session = await deps.sessionsRepo.findById(sessionId);
  const queryOptions: SDKQueryOptions = {
    model: session.model_config?.model || 'claude-sonnet-4-5-20250929',
    workingDirectory: await resolveWorkingDirectory(session),
    environment: await resolveEnvironment(session),
    mcpServers: await resolveMCPServers(session),
    thinking: await resolveThinkingMode(session, prompt),
  };

  // ===== RESUME LOGIC =====

  // CASE 1: Fork/Spawn on first prompt
  const parentSessionId =
    session.genealogy?.forked_from_session_id ||
    session.genealogy?.parent_session_id;

  if (parentSessionId && !session.sdk_session_id) {
    const parentSession = await deps.sessionsRepo.findById(parentSessionId);

    if (parentSession?.sdk_session_id) {
      console.log(`Forking from parent session ${parentSessionId}`);
      queryOptions.resume = parentSession.sdk_session_id;
      queryOptions.forkSession = true;  // SDK creates new session ID
    }
  }

  // CASE 2: Normal resume (existing SDK session ID)
  else if (session.sdk_session_id) {
    // Check if session is stale
    const hoursSinceUpdate =
      (Date.now() - new Date(session.last_updated).getTime()) / 3600000;
    const hasWorktree = !!session.worktree_id;

    if (hoursSinceUpdate > 24 || !hasWorktree) {
      console.log('Session is stale, clearing SDK session ID');
      await deps.sessionsRepo.update(sessionId, {
        data: {
          ...session.data,
          sdk_session_id: undefined
        }
      });
    } else {
      console.log(`Resuming SDK session ${session.sdk_session_id}`);
      queryOptions.resume = session.sdk_session_id;
    }
  }

  // CASE 3: Fresh session (no sdk_session_id)
  // SDK will start fresh and return new session ID via SessionIdCapturedEvent

  // ===== CREATE QUERY =====
  const result = await query(prompt, queryOptions);

  // ===== CREATE PROCESSOR =====
  const processor = new SDKMessageProcessor(sessionId, {
    onSessionIdCaptured: async (sdkSessionId: string) => {
      console.log(`Captured SDK session ID: ${sdkSessionId}`);
      await deps.sessionsRepo.update(sessionId, {
        data: {
          ...session.data,
          sdk_session_id: sdkSessionId,
        },
      });
    },
  });

  return { query: result, processor };
}
```

### 3.4 Partial Message Handling

**Message creation is atomic:**

```typescript
// Task service handles streaming workflow
export async function executeTask(
  sessionId: SessionID,
  prompt: string,
  streamingCallbacks: StreamingCallbacks
): Promise<Task> {

  // 1. Create task
  const task = await tasksRepo.create({
    session_id: sessionId,
    prompt,
    status: 'running',
    message_range: { start_index: session.message_count, end_index: null },
    git_state: { sha_at_start: currentSha, sha_at_end: null },
  });

  // 2. Create user message (immediately persisted)
  const userMessageIndex = session.message_count;
  const userMessage = await messagesRepo.create({
    session_id: sessionId,
    task_id: task.task_id,
    index: userMessageIndex,
    role: 'user',
    content: [{ type: 'text', text: prompt }],
  });

  // Update session
  await sessionsRepo.update(sessionId, {
    status: 'running',
    message_count: userMessageIndex + 1,
    data: { ...session.data, active_task_id: task.task_id },
  });

  // 3. Stream assistant response
  let currentTextMessageId: MessageID | null = null;
  let currentThinkingMessageId: MessageID | null = null;
  let assistantContent: MessageContent[] = [];
  let toolsUsed: Set<string> = new Set();
  let tokenInfo = { input_tokens: 0, output_tokens: 0, thinking_tokens: 0 };

  try {
    for await (const event of promptService.promptSessionStreaming(sessionId, prompt)) {

      // Emit streaming chunks (ephemeral)
      if (event.type === 'partial') {
        if (!currentTextMessageId) {
          currentTextMessageId = generateId();
          streamingCallbacks.onStreamStart(currentTextMessageId, { taskId: task.task_id });
        }
        streamingCallbacks.onStreamChunk(currentTextMessageId, event.textChunk);
      }

      if (event.type === 'thinking_partial') {
        if (!currentThinkingMessageId) {
          currentThinkingMessageId = generateId();
          streamingCallbacks.onThinkingStart(currentThinkingMessageId, { taskId: task.task_id });
        }
        streamingCallbacks.onThinkingChunk(currentThinkingMessageId, event.thinkingChunk);
      }

      // Track tool uses
      if (event.type === 'tool_start') {
        toolsUsed.add(event.tool_name);
      }

      // Save complete message to DB (permanent)
      if (event.type === 'complete' && event.role === 'assistant') {
        const assistantMessageIndex = session.message_count + 1;
        const messageId = currentTextMessageId || currentThinkingMessageId || generateId();

        await messagesRepo.create({
          message_id: messageId,
          session_id: sessionId,
          task_id: task.task_id,
          index: assistantMessageIndex,
          role: 'assistant',
          content: event.content,
          tools_used: Array.from(toolsUsed),
          token_info: tokenInfo,
        });

        // Update session
        await sessionsRepo.update(sessionId, {
          message_count: assistantMessageIndex + 1,
        });

        // Reset stream IDs
        currentTextMessageId = null;
        currentThinkingMessageId = null;
        toolsUsed.clear();
      }

      // Track token usage
      if (event.type === 'result') {
        tokenInfo = event.usage;
      }

      // Handle end of conversation
      if (event.type === 'end') {
        break;
      }
    }

    // 4. Complete task
    await tasksRepo.update(task.task_id, {
      status: 'completed',
      message_range: { start_index: userMessageIndex, end_index: session.message_count },
      git_state: { ...task.git_state, sha_at_end: getCurrentSha() },
      token_usage: tokenInfo,
      completed_at: new Date(),
      duration_ms: Date.now() - task.started_at.getTime(),
    });

    await sessionsRepo.update(sessionId, {
      status: 'idle',
      data: { ...session.data, active_task_id: undefined },
    });

    return task;

  } catch (error) {
    // Handle interruption or error
    await tasksRepo.update(task.task_id, {
      status: 'stopped',
      message_range: { start_index: userMessageIndex, end_index: session.message_count },
    });

    await sessionsRepo.update(sessionId, {
      status: 'idle',
      data: { ...session.data, active_task_id: undefined },
    });

    throw error;
  }
}
```

**Interruption behavior:**
1. Partial streaming stops immediately
2. **Last complete message is saved to DB** (conversation continuity)
3. Task status set to `stopped`
4. **SDK session ID preserved** (for resume)
5. Next prompt continues from last saved message (SDK handles history)

---

## 4. Python Conversion Guidelines

### 4.1 Technology Stack Mapping

| TypeScript (Agor) | Python Equivalent | Notes |
|-------------------|-------------------|-------|
| **Backend Framework** |
| FeathersJS | **FastAPI** | REST + WebSocket support via `fastapi-socketio` |
| Express.js | Starlette (built into FastAPI) | ASGI framework |
| **Database** |
| Drizzle ORM | **SQLAlchemy 2.0** | Type-safe ORM with async support |
| LibSQL | SQLite3 / LibSQL Python | Compatible with SQLite |
| **Real-time** |
| Socket.IO (FeathersJS) | **python-socketio** | Same protocol, official Python impl |
| **Git Operations** |
| simple-git | **GitPython** | Native Python git library |
| **Type System** |
| TypeScript interfaces | **Pydantic v2** | Runtime validation + serialization |
| Branded types | `NewType` + validators | Type aliases with validation |
| **Async Patterns** |
| async/await + generators | **asyncio + async generators** | Direct mapping |
| **CLI Framework** |
| oclif | **Typer** or Click | Type-safe CLI with autocomplete |
| **UUID Generation** |
| uuidv7 package | **uuid-utils** or `uuid7` | UUIDv7 support |

### 4.2 Async Generator Pattern (Direct Mapping)

TypeScript:
```typescript
async function* promptSessionStreaming(
  sessionId: string,
  prompt: string
): AsyncGenerator<ProcessedEvent> {
  const result = await query(prompt, options);

  for await (const msg of result) {
    const events = await processor.process(msg);
    for (const event of events) {
      yield event;
    }
  }
}

// Usage
for await (const event of promptSessionStreaming(sessionId, prompt)) {
  console.log(event);
}
```

Python:
```python
from typing import AsyncGenerator

async def prompt_session_streaming(
    session_id: str,
    prompt: str
) -> AsyncGenerator[ProcessedEvent, None]:
    result = await query(prompt, options)

    async for msg in result:
        events = await processor.process(msg)
        for event in events:
            yield event

# Usage
async for event in prompt_session_streaming(session_id, prompt):
    print(event)
```

### 4.3 Pydantic Models (Type System)

TypeScript:
```typescript
// packages/core/src/types/session.ts
export type SessionID = string & { __brand: 'SessionID' };
export type SessionStatus = 'idle' | 'running' | 'awaiting_permission' | 'completed' | 'failed';

export interface Session {
  session_id: SessionID;
  status: SessionStatus;
  agentic_tool: 'claude-code' | 'codex' | 'gemini';
  worktree_id: WorktreeID | null;
  message_count: number;
  created_at: Date;
  last_updated: Date;

  // JSON data
  data: {
    sdk_session_id?: string;
    git_state: {
      ref: string;
      base_sha: string;
      current_sha: string;
    };
    genealogy: {
      children: SessionID[];
      fork_point_task_id?: TaskID;
    };
    tasks: TaskID[];
  };
}
```

Python:
```python
# core/types/session.py
from pydantic import BaseModel, Field, validator
from typing import Literal, Optional
from datetime import datetime
from uuid import UUID

# Branded type using NewType
SessionID = str  # Could use NewType for stricter typing
WorktreeID = str
TaskID = str

SessionStatus = Literal['idle', 'running', 'awaiting_permission', 'completed', 'failed']
AgenticTool = Literal['claude-code', 'codex', 'gemini']

class GitState(BaseModel):
    ref: str
    base_sha: str
    current_sha: str

class Genealogy(BaseModel):
    children: list[SessionID] = Field(default_factory=list)
    fork_point_task_id: Optional[TaskID] = None
    spawn_point_task_id: Optional[TaskID] = None

class SessionData(BaseModel):
    sdk_session_id: Optional[str] = None
    git_state: GitState
    genealogy: Genealogy
    tasks: list[TaskID] = Field(default_factory=list)
    active_task_id: Optional[TaskID] = None
    mcp_token: Optional[str] = None

class Session(BaseModel):
    session_id: SessionID
    status: SessionStatus
    agentic_tool: AgenticTool
    worktree_id: Optional[WorktreeID] = None
    message_count: int = 0
    created_at: datetime
    last_updated: datetime

    # JSON data
    data: SessionData

    class Config:
        json_encoders = {
            datetime: lambda v: v.isoformat()
        }
```

### 4.4 SQLAlchemy Schema

TypeScript (Drizzle):
```typescript
export const sessions = sqliteTable('sessions', {
  session_id: text('session_id').primaryKey(),
  status: text('status', { enum: ['idle', 'running', ...] }).notNull(),
  agentic_tool: text('agentic_tool').notNull(),
  worktree_id: text('worktree_id').references(() => worktrees.worktree_id),
  message_count: integer('message_count').default(0),
  created_at: integer('created_at', { mode: 'timestamp_ms' }).notNull(),
  data: json('data').$type<SessionData>(),
});
```

Python (SQLAlchemy 2.0):
```python
# core/db/schema.py
from sqlalchemy import Column, String, Integer, JSON, ForeignKey, Enum
from sqlalchemy.orm import DeclarativeBase, relationship
from datetime import datetime
import enum

class Base(DeclarativeBase):
    pass

class SessionStatusEnum(str, enum.Enum):
    IDLE = 'idle'
    RUNNING = 'running'
    AWAITING_PERMISSION = 'awaiting_permission'
    COMPLETED = 'completed'
    FAILED = 'failed'

class AgenticToolEnum(str, enum.Enum):
    CLAUDE_CODE = 'claude-code'
    CODEX = 'codex'
    GEMINI = 'gemini'

class SessionModel(Base):
    __tablename__ = 'sessions'

    session_id = Column(String, primary_key=True)
    status = Column(Enum(SessionStatusEnum), nullable=False)
    agentic_tool = Column(Enum(AgenticToolEnum), nullable=False)
    worktree_id = Column(String, ForeignKey('worktrees.worktree_id'))
    message_count = Column(Integer, default=0)
    created_at = Column(Integer, nullable=False)  # Unix timestamp ms
    last_updated = Column(Integer, nullable=False)
    data = Column(JSON, nullable=False)  # SessionData as JSON

    # Relationships
    worktree = relationship('WorktreeModel', back_populates='sessions')
    messages = relationship('MessageModel', back_populates='session', cascade='all, delete-orphan')
    tasks = relationship('TaskModel', back_populates='session', cascade='all, delete-orphan')
```

### 4.5 Repository Pattern

TypeScript:
```typescript
export class SessionRepository {
  constructor(private db: Database) {}

  async findById(sessionId: SessionID): Promise<Session | null> {
    return this.db.query.sessions.findFirst({
      where: eq(schema.sessions.session_id, sessionId),
    });
  }

  async create(data: CreateSessionData): Promise<Session> {
    const sessionId = generateId();
    await this.db.insert(schema.sessions).values({ session_id: sessionId, ...data });
    return this.findById(sessionId);
  }
}
```

Python:
```python
# core/db/repositories/sessions.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from typing import Optional
from core.db.schema import SessionModel
from core.types.session import Session, SessionID, CreateSessionData
from core.utils.id_generator import generate_id
import time

class SessionRepository:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def find_by_id(self, session_id: SessionID) -> Optional[Session]:
        stmt = select(SessionModel).where(SessionModel.session_id == session_id)
        result = await self.db.execute(stmt)
        model = result.scalar_one_or_none()

        if model is None:
            return None

        # Convert ORM model to Pydantic model
        return Session.model_validate(model)

    async def create(self, data: CreateSessionData) -> Session:
        session_id = generate_id()
        now = int(time.time() * 1000)  # Unix timestamp ms

        model = SessionModel(
            session_id=session_id,
            created_at=now,
            last_updated=now,
            **data.model_dump()
        )

        self.db.add(model)
        await self.db.commit()
        await self.db.refresh(model)

        return Session.model_validate(model)

    async def update(self, session_id: SessionID, data: dict) -> Session:
        stmt = select(SessionModel).where(SessionModel.session_id == session_id)
        result = await self.db.execute(stmt)
        model = result.scalar_one()

        for key, value in data.items():
            setattr(model, key, value)

        model.last_updated = int(time.time() * 1000)

        await self.db.commit()
        await self.db.refresh(model)

        return Session.model_validate(model)
```

### 4.6 FastAPI Service (REST + WebSocket)

TypeScript (FeathersJS):
```typescript
import { Service } from '@feathersjs/feathers';

export class SessionsService implements Service {
  async find(params) { /* ... */ }
  async get(id, params) { /* ... */ }
  async create(data, params) { /* ... */ }
  async patch(id, data, params) { /* ... */ }
  async remove(id, params) { /* ... */ }
}

// Register service
app.use('/sessions', new SessionsService());
```

Python (FastAPI):
```python
# daemon/services/sessions.py
from fastapi import APIRouter, Depends, HTTPException
from socketio import AsyncServer
from typing import List, Optional
from core.types.session import Session, CreateSessionData, SessionID
from core.db.repositories.sessions import SessionRepository
from daemon.dependencies import get_session_repo, get_socketio

router = APIRouter(prefix='/sessions', tags=['sessions'])

# REST Endpoints
@router.get('/', response_model=List[Session])
async def list_sessions(
    status: Optional[str] = None,
    repo: SessionRepository = Depends(get_session_repo)
):
    """List all sessions with optional filtering"""
    return await repo.find_all(filters={'status': status})

@router.get('/{session_id}', response_model=Session)
async def get_session(
    session_id: SessionID,
    repo: SessionRepository = Depends(get_session_repo)
):
    """Get session by ID"""
    session = await repo.find_by_id(session_id)
    if not session:
        raise HTTPException(status_code=404, detail='Session not found')
    return session

@router.post('/', response_model=Session, status_code=201)
async def create_session(
    data: CreateSessionData,
    repo: SessionRepository = Depends(get_session_repo),
    sio: AsyncServer = Depends(get_socketio)
):
    """Create new session"""
    session = await repo.create(data)

    # Broadcast WebSocket event
    await sio.emit('sessions.created', session.model_dump())

    return session

@router.patch('/{session_id}', response_model=Session)
async def update_session(
    session_id: SessionID,
    data: dict,
    repo: SessionRepository = Depends(get_session_repo),
    sio: AsyncServer = Depends(get_socketio)
):
    """Update session"""
    session = await repo.update(session_id, data)

    # Broadcast WebSocket event
    await sio.emit('sessions.patched', session.model_dump())

    return session

@router.delete('/{session_id}', status_code=204)
async def delete_session(
    session_id: SessionID,
    repo: SessionRepository = Depends(get_session_repo),
    sio: AsyncServer = Depends(get_socketio)
):
    """Delete session"""
    await repo.delete(session_id)

    # Broadcast WebSocket event
    await sio.emit('sessions.removed', {'session_id': session_id})

# Custom Methods
@router.post('/{session_id}/fork', response_model=Session)
async def fork_session(
    session_id: SessionID,
    data: dict,
    repo: SessionRepository = Depends(get_session_repo)
):
    """Fork session at decision point"""
    forked_session = await repo.fork(session_id, data)
    return forked_session

@router.post('/{session_id}/spawn', response_model=Session)
async def spawn_session(
    session_id: SessionID,
    data: dict,
    repo: SessionRepository = Depends(get_session_repo)
):
    """Spawn child session"""
    spawned_session = await repo.spawn(session_id, data)
    return spawned_session

@router.get('/{session_id}/genealogy')
async def get_genealogy(
    session_id: SessionID,
    repo: SessionRepository = Depends(get_session_repo)
):
    """Get session genealogy tree"""
    genealogy = await repo.get_genealogy(session_id)
    return genealogy
```

### 4.7 Prompt Service (Async Generator)

TypeScript:
```typescript
export class ClaudePromptService {
  private activeQueries = new Map<SessionID, Query>();
  private stopRequested = new Map<SessionID, boolean>();

  async *promptSessionStreaming(
    sessionId: SessionID,
    prompt: string,
    options?: PromptOptions
  ): AsyncGenerator<ProcessedEvent> {
    const { query: result, processor } = await setupQuery(...);
    this.activeQueries.set(sessionId, result);

    try {
      for await (const msg of result) {
        if (this.stopRequested.get(sessionId)) break;
        const events = await processor.process(msg);
        for (const event of events) yield event;
      }
    } finally {
      this.activeQueries.delete(sessionId);
    }
  }

  async stopTask(sessionId: SessionID): Promise<{ success: boolean }> {
    const query = this.activeQueries.get(sessionId);
    if (!query) return { success: false };

    this.stopRequested.set(sessionId, true);
    await query.interrupt();
    this.activeQueries.delete(sessionId);
    return { success: true };
  }
}
```

Python:
```python
# core/tools/claude/prompt_service.py
from typing import AsyncGenerator, Dict, Optional
from core.types.session import SessionID
from core.tools.claude.query_builder import setup_query
from core.tools.claude.message_processor import SDKMessageProcessor, ProcessedEvent

class ClaudePromptService:
    def __init__(self):
        self.active_queries: Dict[SessionID, 'Query'] = {}
        self.stop_requested: Dict[SessionID, bool] = {}

    async def prompt_session_streaming(
        self,
        session_id: SessionID,
        prompt: str,
        options: Optional[dict] = None
    ) -> AsyncGenerator[ProcessedEvent, None]:
        """Stream prompt response as events"""

        # Setup query
        query, processor = await setup_query(session_id, prompt, options)
        self.active_queries[session_id] = query

        try:
            # Iterate through streaming events
            async for msg in query:
                # Check for interruption
                if self.stop_requested.get(session_id):
                    print(f'Stop requested for session {session_id}')
                    del self.stop_requested[session_id]
                    break

                # Process message
                events = await processor.process(msg)
                for event in events:
                    yield event

        finally:
            # Cleanup
            if session_id in self.active_queries:
                del self.active_queries[session_id]
            if session_id in self.stop_requested:
                del self.stop_requested[session_id]

    async def stop_task(self, session_id: SessionID) -> dict:
        """Stop active task for session"""

        query = self.active_queries.get(session_id)
        if not query:
            return {'success': False, 'reason': 'No active task'}

        try:
            print(f'Stopping task for session {session_id}')

            # Set stop flag
            self.stop_requested[session_id] = True

            # Call SDK interrupt
            await query.interrupt()

            # Cleanup
            del self.active_queries[session_id]

            return {'success': True}

        except Exception as e:
            del self.stop_requested[session_id]
            return {'success': False, 'reason': str(e)}
```

### 4.8 Message Processor (State Machine)

Python:
```python
# core/tools/claude/message_processor.py
from dataclasses import dataclass, field
from typing import List, Dict, Optional, Callable, Awaitable
from core.types.message import MessageContent
from core.utils.id_generator import generate_id
import time

@dataclass
class ProcessorState:
    session_id: str
    message_count: int = 0
    last_activity_time: float = field(default_factory=time.time)

    # Streaming state
    current_text_buffer: str = ''
    current_thinking_buffer: str = ''
    text_stream_id: Optional[str] = None
    thinking_stream_id: Optional[str] = None

    # Tool tracking
    tools_in_progress: Dict[str, dict] = field(default_factory=dict)

    # SDK session tracking
    captured_sdk_session_id: Optional[str] = None

class SDKMessageProcessor:
    def __init__(
        self,
        session_id: str,
        on_session_id_captured: Optional[Callable[[str], Awaitable[None]]] = None
    ):
        self.state = ProcessorState(session_id=session_id)
        self.on_session_id_captured = on_session_id_captured

    async def process(self, msg: dict) -> List[dict]:
        """Process SDK message into typed events"""

        self.state.message_count += 1
        self.state.last_activity_time = time.time()

        # Route to handler
        match msg.get('type'):
            case 'assistant':
                return await self.handle_assistant(msg)
            case 'user':
                return await self.handle_user(msg)
            case 'stream_event':
                return await self.handle_stream_event(msg)
            case 'result':
                return await self.handle_result(msg)
            case 'system':
                return await self.handle_system(msg)
            case _:
                return []

    async def handle_stream_event(self, msg: dict) -> List[dict]:
        """Handle streaming delta events"""
        events = []

        content_block = msg.get('content_block', {})
        event_type = msg.get('event')

        # Text streaming
        if content_block.get('type') == 'text':
            if event_type == 'content_block_start':
                self.state.text_stream_id = generate_id()
                self.state.current_text_buffer = ''

            elif event_type == 'content_block_delta':
                delta = msg.get('delta', {}).get('text', '')
                self.state.current_text_buffer += delta
                events.append({
                    'type': 'partial',
                    'textChunk': delta,
                    'timestamp': time.time()
                })

        # Thinking streaming
        elif content_block.get('type') == 'thinking':
            if event_type == 'content_block_start':
                self.state.thinking_stream_id = generate_id()
                self.state.current_thinking_buffer = ''

            elif event_type == 'content_block_delta':
                delta = msg.get('delta', {}).get('thinking', '')
                self.state.current_thinking_buffer += delta
                events.append({
                    'type': 'thinking_partial',
                    'thinkingChunk': delta,
                    'timestamp': time.time()
                })

        # Tool use
        elif content_block.get('type') == 'tool_use':
            if event_type == 'content_block_start':
                events.append({
                    'type': 'tool_start',
                    'tool_use_id': content_block['id'],
                    'tool_name': content_block['name'],
                    'input': content_block.get('input', {})
                })

        return events

    async def handle_assistant(self, msg: dict) -> List[dict]:
        """Handle complete assistant message"""

        # Capture SDK session ID
        if sdk_session_id := msg.get('sdk_session_id'):
            if self.state.captured_sdk_session_id != sdk_session_id:
                self.state.captured_sdk_session_id = sdk_session_id
                if self.on_session_id_captured:
                    await self.on_session_id_captured(sdk_session_id)

        return [{
            'type': 'complete',
            'role': 'assistant',
            'content': msg.get('content', []),
            'metadata': {
                'model': msg.get('model'),
                'stop_reason': msg.get('stop_reason')
            }
        }]

    async def handle_result(self, msg: dict) -> List[dict]:
        """Handle token usage result"""
        return [{
            'type': 'result',
            'usage': msg.get('usage', {}),
            'cost_usd': msg.get('cost_usd'),
            'duration_ms': msg.get('duration_ms')
        }]

    async def handle_system(self, msg: dict) -> List[dict]:
        """Handle system messages"""
        return [{
            'type': 'complete',
            'role': 'system',
            'content': [{'type': 'text', 'text': msg.get('content', '')}],
            'metadata': {}
        }]
```

### 4.9 Context Managers for Query Lifecycle

Python:
```python
# core/tools/claude/prompt_service.py
from contextlib import asynccontextmanager

class ClaudePromptService:
    @asynccontextmanager
    async def query_context(self, session_id: SessionID, prompt: str):
        """Context manager for query lifecycle"""
        query, processor = await setup_query(session_id, prompt)
        self.active_queries[session_id] = query

        try:
            yield query, processor
        finally:
            # Cleanup
            if session_id in self.active_queries:
                del self.active_queries[session_id]
            if session_id in self.stop_requested:
                del self.stop_requested[session_id]

# Usage
async with prompt_service.query_context(session_id, prompt) as (query, processor):
    async for msg in query:
        events = await processor.process(msg)
        for event in events:
            print(event)
```

### 4.10 WebSocket Broadcasting (python-socketio)

Python:
```python
# daemon/main.py
from fastapi import FastAPI
from socketio import AsyncServer, ASGIApp

app = FastAPI()
sio = AsyncServer(async_mode='asgi', cors_allowed_origins='*')
socket_app = ASGIApp(sio, app)

# WebSocket events
@sio.event
async def connect(sid, environ):
    print(f'Client connected: {sid}')

@sio.event
async def disconnect(sid):
    print(f'Client disconnected: {sid}')

# Broadcast helper
async def broadcast_session_created(session: Session):
    await sio.emit('sessions.created', session.model_dump())

async def broadcast_session_patched(session: Session):
    await sio.emit('sessions.patched', session.model_dump())

# In service
@router.post('/', response_model=Session)
async def create_session(data: CreateSessionData):
    session = await repo.create(data)
    await broadcast_session_created(session)
    return session
```

---

## 5. Key Files Reference

### Core Type Definitions
- `packages/core/src/types/session.ts` → `core/types/session.py`
- `packages/core/src/types/task.ts` → `core/types/task.py`
- `packages/core/src/types/message.ts` → `core/types/message.py`

### Database Layer
- `packages/core/src/db/schema.ts` → `core/db/schema.py`
- `packages/core/src/db/repositories/sessions.ts` → `core/db/repositories/sessions.py`
- `packages/core/src/db/repositories/tasks.ts` → `core/db/repositories/tasks.py`
- `packages/core/src/db/repositories/messages.ts` → `core/db/repositories/messages.py`

### Agent Integration
- `packages/core/src/tools/claude/index.ts` → `core/tools/claude/__init__.py`
- `packages/core/src/tools/claude/prompt-service.ts` → `core/tools/claude/prompt_service.py`
- `packages/core/src/tools/claude/query-builder.ts` → `core/tools/claude/query_builder.py`
- `packages/core/src/tools/claude/message-processor.ts` → `core/tools/claude/message_processor.py`

### Services (API Layer)
- `apps/agor-daemon/src/services/sessions.ts` → `daemon/services/sessions.py`
- `apps/agor-daemon/src/services/tasks.ts` → `daemon/services/tasks.py`
- `apps/agor-daemon/src/services/messages.ts` → `daemon/services/messages.py`

### Configuration
- `packages/core/src/config/index.ts` → `core/config/__init__.py`
- `apps/agor-daemon/src/index.ts` → `daemon/main.py`

---

## Summary

### Key Architectural Patterns

1. **Event-Driven Streaming** - Async generators for real-time response streaming
2. **State Machine** - SDKMessageProcessor for message routing and aggregation
3. **Repository Pattern** - Clean data access layer with type safety
4. **Dual-Stream Architecture** - Parallel thinking + text streams
5. **Hybrid Persistence** - Materialized columns + JSON blob
6. **Native Interruption** - SDK interrupt() method for graceful stop
7. **Conversation Continuity** - SDK session ID for resume/fork/spawn

### Python Conversion Advantages

- **Direct async generator mapping** - Python's async/await is cleaner than TypeScript
- **Pydantic validation** - Runtime type checking + serialization
- **SQLAlchemy maturity** - Battle-tested ORM with async support
- **FastAPI performance** - One of the fastest Python frameworks
- **python-socketio** - Official Socket.IO implementation
- **Type hints** - Python 3.10+ type system is excellent

### Critical Design Decisions

1. **sdk_session_id is the continuity mechanism** - Store and reuse for resume
2. **Message index is append-only** - Never reorder or delete
3. **Complete messages are atomic** - Partial chunks are ephemeral
4. **Fork creates new conversation** - Parent history is forked by SDK
5. **Spawn creates fresh conversation** - No history inheritance
6. **Interruption preserves state** - Last complete message saved
7. **Worktree is required** - Sessions must belong to worktree

---

**Generated:** 2025-01-12
**For:** Python backend conversion of Agor session architecture
