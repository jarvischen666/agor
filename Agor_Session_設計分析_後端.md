# Agor Session 設計分析 - 後端架構

> **專案**: Agor - 多人協作 AI 編碼代理編排平台
> **分析日期**: 2025-11-12
> **分析範圍**: 後端架構（FeathersJS Daemon + 核心工具層）

---

## 目錄

1. [Agentic 呼叫架構與流程](#1-agentic-呼叫架構與流程)
2. [Session API 設計](#2-session-api-設計)
3. [訊息中斷與接續機制](#3-訊息中斷與接續機制)
4. [權限系統設計](#4-權限系統設計)
5. [三種 Agent 的差異比較](#5-三種-agent-的差異比較)
6. [關鍵架構洞察](#6-關鍵架構洞察)
7. [重要檔案索引](#7-重要檔案索引)

---

## 1. Agentic 呼叫架構與流程

### 1.1 三層架構

Agor 採用三層式架構來編排 AI 代理：

```
UI 層 (React)
    ↓ WebSocket/REST
FeathersJS Daemon
    ↓ 工具層
Agent Tools (Claude/Codex/Gemini)
    ↓ SDK 呼叫
AI SDK (Claude Agent SDK / OpenAI SDK / Google Generative AI)
```

### 1.2 完整訊息流程（從使用者輸入到 AI 回應）

以下是一個完整的請求-回應週期：

```
1. 使用者輸入 Prompt
   → UI 呼叫 POST /tasks { session_id, prompt }

2. Daemon 建立 Task
   → 狀態: created
   → 存入資料庫
   → 透過 WebSocket 廣播 task:created 事件

3. Daemon 更新 Task 狀態
   → 狀態: created → running
   → 廣播 task:patched 事件

4. 工具選擇
   → 根據 session.agentic_tool 選擇工具
   → ClaudeTool / CodexTool / GeminiTool

5. 工具執行
   → executePromptWithStreaming(sessionId, prompt, taskId, permissionMode, callbacks)

6. 建立使用者訊息
   → messagesService.create({ role: 'user', content: prompt })
   → 廣播 message:created 事件

7. SDK 查詢
   → 配置 model、permissions、MCP servers、working directory
   → 呼叫 SDK API (串流模式)

8. 串流回應處理
   → SDK 產生事件流 (async generator)
   → 解析事件類型:
      - 文字片段 (partial text chunks)
      - 工具使用 (tool uses)
      - 思考區塊 (thinking blocks - Claude 限定)

9. 即時廣播
   → streamingCallbacks 發送 WebSocket 事件:
      • streaming:start → UI 顯示輸入指示器
      • streaming:chunk → UI 漸進式顯示文字 (打字機效果)
      • thinking:chunk → UI 顯示思考過程
      • tool:start → UI 顯示工具執行狀態
      • tool:complete → UI 顯示工具結果
      • streaming:end → 串流完成

10. 資料庫持久化
    → messagesService.create({ role: 'assistant', content: completeText })
    → 儲存完整的助理訊息

11. Task 完成
    → 狀態: running → completed
    → 更新使用量統計 (tokens, cost)
    → 廣播 task:patched 事件

12. Session 更新
    → 訊息計數 +1
    → ready_for_prompt 標記設為 true
    → 廣播 session:patched 事件
```

### 1.3 關鍵檔案

| 元件 | 檔案路徑 | 責任 |
|------|----------|------|
| 流程編排 | `apps/agor-daemon/src/services/tasks.ts` | TasksService - 任務生命週期管理 |
| Claude 執行 | `packages/core/src/tools/claude/claude-tool.ts` | ClaudeTool - Claude SDK 整合 |
| 訊息處理 | `packages/core/src/tools/claude/message-processor.ts` | SDKMessageProcessor - 事件解析 |
| 查詢建構 | `packages/core/src/tools/claude/query-builder.ts` | setupQuery - SDK 配置 |

### 1.4 Agent SDK 整合模式

#### Claude Code (透過 Claude Agent SDK)

```typescript
// 1. 設定查詢配置
const { query: result } = await setupQuery(sessionId, prompt, deps, options);

// 2. 建立訊息處理器
const processor = new SDKMessageProcessor({
  sessionId,
  enableTokenStreaming: true
});

// 3. 處理串流事件
for await (const msg of result) {
  const events = await processor.process(msg);

  for (const event of events) {
    if (event.type === 'partial') {
      // 發送即時文字片段
      streamingCallbacks.onStreamChunk(messageId, event.textChunk);
    }
    else if (event.type === 'thinking_partial') {
      // 單獨發送思考片段
      streamingCallbacks.onThinkingChunk(messageId, event.thinkingChunk);
    }
    else if (event.type === 'complete') {
      // 儲存完整訊息到資料庫
      await createAssistantMessage(...);
    }
  }
}
```

**Claude 特有功能：**

- ✅ **自動載入 CLAUDE.md** - 透過 `settingSources: ['project']`
- ✅ **預設系統提示** - 使用 `claude_code` preset
- ✅ **對話延續** - 透過 `sdk_session_id` (儲存在 DB，作為 `resume` 選項傳遞)
- ✅ **Fork/Spawn 支援** - 透過 `forkSession: true` (從父 session 建立新 SDK session)
- ✅ **擴展思考模式** - 自動偵測關鍵字 ("think", "think hard", "ultrathink")
- ✅ **Session 上下文注入** - Agor session 元資料附加到系統提示

#### Codex (透過 OpenAI SDK)

```typescript
// 使用 OpenAI 的自定義 CodexAgent 類別
const result = runStreamed({
  agent: codexAgent,
  input: prompt,
  config: { sandboxMode, approvalPolicy, networkAccess }
});

for await (const event of result) {
  if (event.type === 'item.updated') {
    // 部分文字片段 (罕見 - Codex 通常發送完整文字)
    streamingCallbacks?.onStreamChunk(messageId, event.delta);
  }
  else if (event.type === 'item.completed') {
    // 完整訊息
    await createAssistantMessage(...);
  }
  else if (event.type === 'tool.completed') {
    // 工具執行完成
    await createAssistantMessage(...); // 工具結果為獨立訊息
  }
}
```

**Codex 關鍵差異：**

- ❌ **無自動上下文載入** - 需要手動讀取檔案
- ✅ **Thread 為基礎的延續** - 透過 `thread_id` (儲存為 `sdk_session_id`)
- 🟡 **雙重權限模型** - `sandboxMode` (WHERE) + `approvalPolicy` (WHETHER)
- ❌ **無內建思考模式**

#### Gemini (透過 Google Generative AI SDK)

```typescript
// 使用 Google 的聊天 session API
const result = chat.sendMessageStream(prompt);

for await (const chunk of result.stream) {
  const text = chunk.text();
  streamingCallbacks?.onStreamChunk(messageId, text);
}

const finalResponse = await result.response;
await createAssistantMessage(...);
```

**Gemini 關鍵差異：**

- 🟡 **手動歷史管理** - 透過 `setHistory(messages)`
- ❌ **無 session ID** - Agor 管理對話狀態
- 🟡 **較簡單的串流模型** - 僅文字，無工具串流

---

## 2. Session API 設計

### 2.1 REST + WebSocket 統一 API

Agor 使用 **FeathersJS** 提供統一的 REST 和 WebSocket API：

#### RESTful 端點

```http
GET    /sessions              # 列出 sessions (支援篩選)
GET    /sessions/:id          # 取得單一 session
POST   /sessions              # 建立新 session
PATCH  /sessions/:id          # 更新 session
DELETE /sessions/:id          # 刪除 session (級聯刪除子 sessions)
POST   /sessions/:id/fork     # Fork session
POST   /sessions/:id/spawn    # Spawn 子 session
GET    /sessions/:id/genealogy # 取得 session 家譜樹
```

#### WebSocket 事件 (即時廣播)

所有 CRUD 操作自動廣播事件給已連接的客戶端：

```typescript
// FeathersJS 服務的自動事件
client.service('sessions').on('created', session => { ... });
client.service('sessions').on('patched', session => { ... });
client.service('sessions').on('removed', session => { ... });

// 串流的自定義事件
client.service('messages').on('streaming:start', data => { ... });
client.service('messages').on('streaming:chunk', data => { ... });
client.service('messages').on('thinking:chunk', data => { ... });
client.service('messages').on('streaming:end', data => { ... });

// 權限事件
client.service('tasks').on('patched', task => { ... }); // 狀態變更
```

### 2.2 服務層結構

#### SessionsService

**檔案**: `apps/agor-daemon/src/services/sessions.ts`

```typescript
class SessionsService extends DrizzleService<Session> {
  // 標準 CRUD
  async find(params): Promise<Paginated<Session>>
  async get(id): Promise<Session>
  async create(data): Promise<Session>
  async patch(id, data): Promise<Session>
  async remove(id): Promise<Session> // 級聯刪除子 sessions

  // 自定義方法
  async fork(id, { prompt, task_id }): Promise<Session>
  async spawn(id, { prompt, agentic_tool, task_id }): Promise<Session>
  async getGenealogy(id): Promise<{ session, ancestors, children }>
}
```

#### Fork vs Spawn 的差異

| 特性 | Fork | Spawn |
|------|------|-------|
| **關聯欄位** | `forked_from_session_id` | `parent_session_id` |
| **分岔點** | `fork_point_message_index` | `spawn_point_message_index` |
| **對話歷史** | ✅ 複製到分岔點 | ❌ 全新上下文視窗 |
| **SDK 行為** | 繼承父 `sdk_session_id` + `forkSession: true` | 建立新 SDK session |
| **設定繼承** | 完全繼承父設定 (權限、模型、上下文檔案) | 可使用不同 agentic_tool |
| **使用場景** | 探索不同對話分支 | 建立子任務 / 切換 agent |

```typescript
// Fork 範例
POST /sessions/:id/fork
{
  "prompt": "Let's try a different approach",
  "task_id": "task-123"
}
// 回傳: 新 session with forked_from_session_id = :id

// Spawn 範例
POST /sessions/:id/spawn
{
  "prompt": "Use Codex to refactor this module",
  "agentic_tool": "codex",
  "task_id": "task-456"
}
// 回傳: 新 session with parent_session_id = :id
```

#### TasksService

**檔案**: `apps/agor-daemon/src/services/tasks.ts`

```typescript
class TasksService extends DrizzleService<Task> {
  async find(params): Promise<Paginated<Task>>
  async get(id): Promise<Task>
  async create(data): Promise<Task>
  async patch(id, data): Promise<Task> // 完成時自動設定 session.ready_for_prompt
  async remove(id): Promise<Task>

  // 自定義方法
  async complete(id, { report }): Promise<Task>
  async fail(id, { error }): Promise<Task>
  async getRunning(): Promise<Task[]>
  async getOrphaned(): Promise<Task[]> // 用於清理
}
```

#### Task 狀態生命週期

```
created → running → [awaiting_permission] → completed/failed/stopped
```

| 狀態 | 說明 | 觸發條件 |
|------|------|----------|
| `created` | 任務已建立但未開始 | TasksService.create() |
| `running` | Agent 正在執行 | executePromptWithStreaming() |
| `awaiting_permission` | 等待使用者批准工具使用 | canUseTool callback |
| `completed` | 任務成功完成 | 所有訊息處理完畢 |
| `failed` | 任務失敗 (錯誤) | 捕捉到異常 |
| `stopped` | 使用者手動中斷 | stopTask() |

### 2.3 資料持久化策略

#### 混合具體化 (Materialized Columns + JSON)

**具體化欄位** (索引以加速查詢)：

```sql
-- 身份識別
session_id, task_id, message_id

-- 篩選欄位
status, agentic_tool, worktree_id, created_at

-- 家譜關聯
parent_session_id, forked_from_session_id

-- 時間戳記
created_at, updated_at
```

**JSON blob** (靈活的 schema)：

```typescript
// 動態配置，無需 migration
git_state: { ref, base_sha, current_sha }
genealogy: { fork_point_task_id, children[], ... }
permission_config: { mode, codex: {...} }
model_config: { mode, model, thinkingMode, ... }
custom_context: Record<string, unknown>
```

**優點：**

- ✅ 索引欄位的快速查詢
- ✅ Schema 變更無需 migration (JSON 欄位)
- ✅ 跨資料庫相容性 (LibSQL → PostgreSQL)

---

## 3. 訊息中斷與接續機制

### 3.1 中斷機制 (Interruption)

三種 agent 使用不同的中斷策略：

#### Claude Code - 原生 SDK `interrupt()` 方法

**檔案**: `packages/core/src/tools/claude/prompt-service.ts`

```typescript
// ClaudePromptService.stopTask()
async stopTask(sessionId: SessionID, taskId: TaskID): Promise<void> {
  const queryObj = this.activeQueries.get(sessionId);

  if (queryObj) {
    // 呼叫 SDK 的原生中斷方法 (相當於 CLI 按 Escape 鍵)
    await queryObj.interrupt();
  }

  // 設定標記以立即跳出事件迴圈
  this.stopRequested.set(sessionId, true);
}

// 事件迴圈檢查標記
for await (const msg of result) {
  if (this.stopRequested.get(sessionId)) {
    console.log('🛑 停止請求，跳出事件迴圈');
    this.stopRequested.delete(sessionId);
    break;
  }
  // 處理事件...
}
```

**流程：**

1. UI 發送 `PATCH /tasks/:id` with `{ status: 'stopping' }`
2. Daemon 呼叫 `claudeTool.stopTask(sessionId, taskId)`
3. Tool 呼叫 `promptService.stopTask(sessionId)`
4. Service 呼叫 `queryObj.interrupt()` (優雅的 SDK 停止)
5. Service 設定 `stopRequested` 標記
6. 事件迴圈在下次迭代檢查標記並跳出
7. Task 狀態更新為 `stopped`

#### Codex - 標記為基礎 (Flag-based)

**原因**: OpenAI SDK 缺乏原生中斷方法

**檔案**: `packages/core/src/tools/codex/prompt-service.ts`

```typescript
// CodexPromptService.stopTask()
async stopTask(sessionId: SessionID, taskId: TaskID): Promise<void> {
  this.stopRequested.set(sessionId, true);
}

// 事件迴圈檢查標記
for await (const event of result) {
  if (this.stopRequested.get(sessionId)) {
    console.log('🛑 停止請求，跳出事件迴圈');
    this.stopRequested.delete(sessionId);
    break;
  }
  // 處理事件...
}
```

#### Gemini - AbortController (原生 SDK 支援)

**檔案**: `packages/core/src/tools/gemini/prompt-service.ts`

```typescript
// GeminiPromptService - 儲存 AbortController
private activeControllers = new Map<SessionID, AbortController>();

// 執行時儲存 controller
const controller = new AbortController();
this.activeControllers.set(sessionId, controller);

const result = chat.sendMessageStream(prompt, {
  signal: controller.signal
});

// 中斷方法
async stopTask(sessionId: SessionID, taskId: TaskID): Promise<void> {
  const controller = this.activeControllers.get(sessionId);
  controller?.abort(); // 取消串流請求
}
```

### 3.2 接續機制 (Resumption / Conversation Continuity)

#### Claude Code - SDK Session ID 持久化

**檔案**: `packages/core/src/tools/claude/query-builder.ts`

```typescript
// 首次 prompt 後
const agentSessionId = event.session_id; // 從 SDK
await sessionsRepo.update(sessionId, { sdk_session_id: agentSessionId });

// 後續 prompts
queryOptions.resume = session.sdk_session_id; // 恢復現有對話
```

**Fork/Spawn 處理：**

```typescript
// Fork: 從父 session 的 SDK session 開始
if (parentSession?.sdk_session_id && !session.sdk_session_id) {
  queryOptions.resume = parentSession.sdk_session_id;
  queryOptions.forkSession = true; // SDK 建立新 session ID
}

// Spawn: 全新開始 (無 resume)
// SDK 會回傳新的 session ID
```

**過時 Session 偵測：**

```typescript
const hoursSinceUpdate = (now - lastUpdated) / (1000 * 60 * 60);
if (hoursSinceUpdate > 24 || !session.worktree_id) {
  // 清除過時的 session ID，重新開始
  await sessionsRepo.update(sessionId, { sdk_session_id: undefined });
}
```

#### Codex - Thread ID 持久化

**檔案**: `packages/core/src/tools/codex/prompt-service.ts`

```typescript
// Codex 使用 OpenAI thread IDs
const threadId = event.threadId; // 從 SDK
await sessionsRepo.update(sessionId, { sdk_session_id: threadId });

// 恢復: 傳遞 thread ID 給 SDK
const thread = await openai.beta.threads.retrieve(sdk_session_id);
```

#### Gemini - 手動歷史管理

**檔案**: `packages/core/src/tools/gemini/prompt-service.ts`

```typescript
// Gemini 需要明確傳遞歷史
const messages = await messagesRepo.findBySessionId(sessionId);
const history = messages.map(m => ({
  role: m.role === 'user' ? 'user' : 'model',
  parts: [{ text: m.content }]
}));

chat.setHistory(history); // SDK 管理對話狀態
```

**對比總結：**

| Agent | 接續方法 | 儲存欄位 | SDK 支援 |
|-------|----------|----------|----------|
| **Claude Code** | SDK session ID | `sdk_session_id` | ✅ 原生 `resume` 選項 |
| **Codex** | Thread ID | `sdk_session_id` (儲存 thread_id) | ✅ Thread 檢索 API |
| **Gemini** | 手動歷史 | (無需儲存) | 🟡 `setHistory()` 方法 |

---

## 4. 權限系統設計

### 4.1 三層架構

Agor 的權限系統尊重 **現有 Claude CLI 權限** 並新增 **UI 為基礎的批准** 作為後備：

```
第 1 層: 使用者層級 (~/.claude/settings.json)    ← SDK 先檢查
第 2 層: 專案層級 (.claude/settings.json)         ← SDK 次檢查
第 3 層: Agor UI 提示                              ← 僅當無規則匹配時顯示
```

### 4.2 SDK 整合透過 `canUseTool` Callback

**關鍵洞察**: Agor 使用 `canUseTool` callback，在 SDK 檢查 settings 檔案 **之後** 才觸發。

**檔案**: `packages/core/src/tools/claude/query-builder.ts`

```typescript
// SDK 配置
queryOptions.canUseTool = createCanUseToolCallback(sessionId, taskId, {
  permissionService,
  tasksService,
  messagesService,
  sessionsService
});
```

### 4.3 權限流程

**檔案**: `packages/core/src/tools/claude/permissions/permission-hooks.ts`

```typescript
// 1. SDK 檢查 settings.json 檔案
//    → 如果規則匹配，直接執行，不呼叫 Agor
//    → 如果無匹配，呼叫 canUseTool callback

// 2. Agor 的 canUseTool callback 觸發
async function canUseTool(toolName, toolInput, { signal }) {
  // 建立權限請求訊息
  const permissionMessage = await messagesService.create({
    type: 'permission_request',
    content: {
      tool_name: toolName,
      tool_input: toolInput,
      status: 'pending'
    }
  });

  // 更新 task 狀態 → 'awaiting_permission'
  await tasksService.patch(taskId, { status: 'awaiting_permission' });

  // 發送 WebSocket 事件 (廣播給所有使用者)
  permissionService.emitRequest(sessionId, { toolName, toolInput });

  // 等待使用者決定 (Promise 暫停 SDK 執行)
  const decision = await permissionService.waitForDecision(
    requestId,
    taskId,
    signal
  );

  if (!decision.allow) {
    return { behavior: 'deny', message: '權限被拒絕' };
  }

  // 建構回應與 SDK 的 updatedPermissions
  return {
    behavior: 'allow',
    updatedInput: toolInput,
    updatedPermissions: decision.remember ? [{
      type: 'addRules',
      rules: [{ toolName }],
      behavior: 'allow',
      destination: decision.scope === 'project' ? 'projectSettings' : 'session'
    }] : undefined
  };
}
```

### 4.4 權限範圍 (Scopes)

使用者可以選擇權限持續多久：

```typescript
enum PermissionScope {
  ONCE = 'once',        // 一次性批准 (無持久化)
  PROJECT = 'project',  // SDK 寫入 .claude/settings.json
  USER = 'user',        // SDK 寫入 ~/.claude/settings.json
  LOCAL = 'local'       // SDK 寫入 .claude/settings.json (gitignored)
}
```

**SDK 持久化映射：**

| Agor 範圍 | SDK destination | 檔案位置 | Git 追蹤 |
|-----------|-----------------|----------|----------|
| `once` | (無 updatedPermissions) | (無) | - |
| `project` | `projectSettings` | `.claude/settings.json` | ✅ 是 |
| `user` | `userSettings` | `~/.claude/settings.json` | - |
| `local` | `localSettings` | `.claude/settings.json` | ❌ 否 (gitignored) |

### 4.5 多使用者支援

權限請求具有 **多使用者感知**：

- ✅ 任何查看 session 的使用者都看到權限提示
- ✅ 第一個批准/拒絕的使用者解決所有人的請求
- ✅ 批准追蹤 `approved_by` 使用者 ID 以便稽核
- ✅ WebSocket 廣播確保所有觀看者立即看到決定

**UI 顯示：**

```
等待批准:  黃色卡片 + 工具詳情 + [批准/拒絕] 按鈕
已批准:    綠色卡片 + 時間戳記
已拒絕:    紅色卡片 + 時間戳記
```

### 4.6 Task 狀態在權限期間

```
running → awaiting_permission → [使用者決定] → running/failed
```

### 4.7 權限序列化 (防止重複提示)

為防止 **並發工具呼叫的重複權限提示**：

**檔案**: `packages/core/src/permissions/permission-service.ts`

```typescript
// 每個 session 的權限鎖
private permissionLocks = new Map<SessionID, Promise<void>>();

// 在 canUseTool callback 中:
const existingLock = permissionLocks.get(sessionId);
if (existingLock) {
  await existingLock; // 等待進行中的權限檢查
}

// 為此權限檢查建立鎖
const newLock = new Promise<void>(resolve => { releaseLock = resolve; });
permissionLocks.set(sessionId, newLock);

try {
  // ... 權限流程 ...
} finally {
  releaseLock(); // 總是釋放
  permissionLocks.delete(sessionId);
}
```

---

## 5. 三種 Agent 的差異比較

### 5.1 功能對比矩陣

| 功能 | Claude Code | Codex | Gemini |
|------|-------------|-------|--------|
| **SDK** | `@anthropic-ai/claude-agent-sdk` | OpenAI SDK | `@google/generative-ai` |
| **上下文載入** | ✅ 自動 (CLAUDE.md) | ❌ 手動 | ❌ 手動 |
| **系統提示** | ✅ 預設 (`claude_code`) | ❌ 手動 | ❌ 手動 |
| **對話延續** | ✅ `sdk_session_id` (自動) | ✅ `thread_id` | 🟡 手動 `setHistory()` |
| **Fork/Spawn** | ✅ `forkSession: true` | ❌ 不支援 | ❌ 不支援 |
| **串流** | ✅ Token 級 (async generator) | 🟡 有限 (OpenAI) | ✅ Token 級 |
| **思考模式** | ✅ 內建 (`maxThinkingTokens`) | ❌ 不支援 | ❌ 不支援 |
| **權限系統** | ✅ `canUseTool` callback | 🟡 自定義實作 | 🟡 自定義實作 |
| **權限範圍** | ✅ 4 層級 (once/session/project/user) | 🟡 自定義 DB 儲存 | 🟡 自定義 DB 儲存 |
| **工具執行** | ✅ 內建 | 🟡 Function calling | 🟡 Function calling |
| **中斷** | ✅ 原生 `interrupt()` | 🟡 標記為基礎 | ✅ `AbortController` |
| **MCP 支援** | ✅ 原生 | ❌ 不支援 | ❌ 不支援 |
| **工作目錄** | ✅ `cwd` 選項 | 🟡 手動設定 | 🟡 手動設定 |

### 5.2 Claude Code - 最完整整合

**優勢：**

- ✅ **生產就緒的 SDK**，與 Claude Code CLI 完全對等
- ✅ **自動上下文載入** (CLAUDE.md, settings.json)
- ✅ **原生權限系統**，SDK 持久化
- ✅ **對話延續**，session ID 管理
- ✅ **Fork/spawn 支援**，透過 `forkSession` 標記
- ✅ **擴展思考模式**，關鍵字自動偵測
- ✅ **即時串流**，分離思考/文字串流
- ✅ **優雅中斷**，透過原生 `interrupt()` 方法

**重要檔案：**

```
packages/core/src/tools/claude/
├── claude-tool.ts              # 主工具類別
├── prompt-service.ts           # SDK 查詢執行
├── query-builder.ts            # SDK 配置
├── message-processor.ts        # 事件處理
└── permissions/
    └── permission-hooks.ts     # 權限系統
```

### 5.3 Codex - 自定義整合

**實作差異：**

- ✅ **Thread 為基礎的延續** (儲存 `thread_id` 為 `sdk_session_id`)
- 🟡 **雙重權限模型**: `sandboxMode` + `approvalPolicy` + `networkAccess`
- 🟡 **自定義權限處理** (無原生 SDK 支援)
- 🟡 **手動工具執行** 透過 function calling
- 🟡 **有限串流** (文字通常完整到達)

**權限模型：**

```typescript
// Codex 特有的權限配置
{
  sandboxMode: 'isolated' | 'shared',     // WHERE: 工具可運行的位置
  approvalPolicy: 'auto' | 'manual',      // WHETHER: 需要批准
  networkAccess: boolean                   // 網路存取權限
}
```

**重要檔案：**

```
packages/core/src/tools/codex/
├── codex-tool.ts               # 主工具類別
└── prompt-service.ts           # SDK 查詢執行
```

### 5.4 Gemini - 最簡單整合

**實作差異：**

- 🟡 **手動歷史管理** 透過 `setHistory(messages)`
- ❌ **無 session ID** (Agor 完全管理狀態)
- 🟡 **簡單串流** (僅文字，無工具串流)
- ✅ **AbortController 為基礎的中斷**
- 🟡 **基本權限處理**

**歷史管理範例：**

```typescript
// 每次 prompt 前載入完整歷史
const messages = await messagesRepo.findBySessionId(sessionId);
const history = messages.map(m => ({
  role: m.role === 'user' ? 'user' : 'model',
  parts: [{ text: m.content }]
}));

chat.setHistory(history);
const result = await chat.sendMessageStream(prompt);
```

**重要檔案：**

```
packages/core/src/tools/gemini/
├── gemini-tool.ts              # 主工具類別
└── prompt-service.ts           # SDK 查詢執行
```

### 5.5 架構選擇建議

**選擇 Claude Code 當：**

- 需要完整的開發代理功能 (檔案編輯、git 操作等)
- 需要 Fork/Spawn 支援
- 需要擴展思考模式
- 需要原生 MCP server 整合
- 需要與 Claude CLI 設定檔相容

**選擇 Codex 當：**

- 需要 OpenAI 生態系整合
- 需要 Thread 為基礎的對話管理
- 已有 OpenAI 帳號/credits

**選擇 Gemini 當：**

- 需要 Google 生態系整合
- 需要簡單的聊天功能
- 預算有限 (Gemini 通常較便宜)

---

## 6. 關鍵架構洞察

### 6.1 Worktree 為中心的設計

每個 session **必須** 關聯一個 worktree：

```typescript
// Session model
interface Session {
  session_id: SessionID;
  worktree_id: WorktreeID;  // 必要外鍵
  // ...
}
```

**原因：**

- ✅ 提供隔離的檔案系統以進行平行開發
- ✅ 工作目錄 (`cwd`) 從 worktree 路徑衍生
- ✅ 啟用同時運行多個功能分支
- ✅ 防止 git 狀態衝突

### 6.2 即時優先 (Real-Time First)

FeathersJS 提供自動 WebSocket 廣播：

```typescript
// 所有 CRUD 操作自動發送事件
sessionsService.create(...) → client.on('sessions created')
tasksService.patch(...)      → client.on('tasks patched')
messagesService.create(...)  → client.on('messages created')

// 自定義串流事件
streamingCallbacks.onStreamChunk(...)    → 'streaming:chunk'
streamingCallbacks.onThinkingChunk(...)  → 'thinking:chunk'
```

**優點：**

- ✅ 多使用者協作，即時游標與在場狀態
- ✅ 漸進式 UI 更新 (打字機效果)
- ✅ 無需輪詢 (polling)

### 6.3 SDK 原生整合

Agor 利用 SDK 能力而非重新實作：

| 功能 | SDK 原生 | Agor 自定義 |
|------|----------|-------------|
| CLAUDE.md 載入 | ✅ Claude SDK | ❌ Codex/Gemini |
| 權限持久化 | ✅ Claude SDK | 🟡 Codex/Gemini (DB) |
| Fork/Spawn | ✅ Claude SDK | ❌ Codex/Gemini |
| 思考模式 | ✅ Claude SDK | ❌ Codex/Gemini |

### 6.4 優雅降級 (Graceful Degradation)

系統在每層處理失敗：

```typescript
// Session guard 模式 (處理執行中刪除的 sessions)
const session = await sessionsRepo.findById(sessionId);
if (!session) {
  throw new Error('Session 已被刪除');
}

// 過時 session 偵測 (清除 24 小時以上的 session IDs)
if (hoursSinceUpdate > 24 || !session.worktree_id) {
  await sessionsRepo.update(sessionId, { sdk_session_id: undefined });
}

// 孤立 task 清理 (daemon 重啟恢復)
const orphanedTasks = await tasksService.getOrphaned();
for (const task of orphanedTasks) {
  await tasksService.fail(task.task_id, { error: 'Daemon 重啟' });
}

// 權限逾時後備 (60 秒自動拒絕)
const decision = await Promise.race([
  permissionService.waitForDecision(requestId),
  timeout(60000).then(() => ({ allow: false }))
]);
```

### 6.5 靈活權限 (Flexible Permissions)

尊重現有 CLI 配置同時新增 UI 層：

```
1. SDK 先檢查 settings.json 檔案
   → 如果規則匹配: 直接執行 (不經過 Agor UI)
   → 如果無匹配: 呼叫 Agor 的 canUseTool callback

2. Agor UI 僅在無規則時顯示
   → 使用者批准後，可選擇持久化範圍
   → SDK 處理實際檔案寫入 (透過 updatedPermissions)
```

---

## 7. 重要檔案索引

### 7.1 型別定義

```
packages/core/src/types/
├── session.ts                  # Session model
├── task.ts                     # Task model
├── message.ts                  # Message model
├── worktree.ts                 # Worktree model
└── permission.ts               # Permission types
```

### 7.2 Claude 實作

```
packages/core/src/tools/claude/
├── claude-tool.ts              # 主工具類別
├── prompt-service.ts           # SDK 查詢執行
├── query-builder.ts            # SDK 配置
├── message-processor.ts        # 事件處理
├── thinking-detector.ts        # 思考模式關鍵字偵測
└── permissions/
    └── permission-hooks.ts     # 權限系統 (canUseTool callback)
```

### 7.3 Codex 實作

```
packages/core/src/tools/codex/
├── codex-tool.ts               # 主工具類別
└── prompt-service.ts           # SDK 查詢執行
```

### 7.4 Gemini 實作

```
packages/core/src/tools/gemini/
├── gemini-tool.ts              # 主工具類別
└── prompt-service.ts           # SDK 查詢執行
```

### 7.5 服務層

```
apps/agor-daemon/src/services/
├── sessions.ts                 # Session CRUD + fork/spawn
├── tasks.ts                    # Task 生命週期管理
├── messages.ts                 # Message CRUD + 串流廣播
└── worktrees.ts                # Worktree 管理
```

```
packages/core/src/permissions/
└── permission-service.ts       # 權限編排
```

### 7.6 資料庫

```
packages/core/src/db/
├── schema.ts                   # Drizzle schema 定義
├── repositories/
│   ├── sessions.ts             # Session 資料存取
│   ├── tasks.ts                # Task 資料存取
│   ├── messages.ts             # Message 資料存取
│   └── worktrees.ts            # Worktree 資料存取
└── index.ts                    # DB 連線設定
```

### 7.7 上下文文件

```
context/concepts/
├── agent-integration.md        # Agent 整合策略
├── permissions.md              # 權限系統設計
├── architecture.md             # 系統架構
├── core.md                     # 核心基元
├── models.md                   # 資料模型
└── worktrees.md                # Worktree 架構
```

---

## 總結

這份分析揭示了一個 **精緻、生產就緒的架構**，在以下方面取得平衡：

✅ **SDK 原生能力** - 最大化利用現有 SDK 功能
✅ **自定義 Agor 功能** - 多使用者協作、即時串流、空間看板
✅ **統一介面** - 為多個 AI 編碼代理提供一致的 API
✅ **即時協作** - WebSocket 廣播、在場狀態、共享權限
✅ **穩健錯誤處理** - Session guards、過時偵測、孤立清理

**適合 Python 移植的關鍵洞察：**

1. **採用相同的三層架構** (API 層 → 工具層 → SDK 層)
2. **使用 async/await 模式** 用於串流處理
3. **實作混合持久化** (SQL columns + JSON blobs)
4. **優先使用 WebSocket** 用於即時更新
5. **為每個 agent 建立抽象基類**，共享通用邏輯
6. **利用 SDK 原生功能** 而非重新實作 (特別是 Claude SDK)
7. **使用標記/AbortController** 用於中斷處理
8. **實作權限序列化** 以防止並發問題

---

**文件版本**: 1.0
**最後更新**: 2025-11-12
**分析來源**: Agor 專案 (TypeScript/Node.js)
