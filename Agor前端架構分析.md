# Agor 前端架構分析文檔

> **分析範圍**: Agor 前端如何處理訊息、工具、權限的顯示和訊息回覆
> **版本**: v0.1
> **日期**: 2025-01-12

---

## 目錄

1. [Agentic 呼叫架構與流程](#1-agentic-呼叫架構與流程)
2. [Session API 設計](#2-session-api-設計)
3. [訊息中斷與接續機制](#3-訊息中斷與接續機制)
4. [Permission 系統設計](#4-permission-系統設計)
5. [三種 SDK 的前端處理差異](#5-三種-sdk-的前端處理差異)
6. [關鍵文件路徑索引](#6-關鍵文件路徑索引)

---

## 1. Agentic 呼叫架構與流程

### 1.1 整體架構

Agor 採用三層架構設計,將前端、後端和 AI SDK 清楚分離:

```
┌─────────────────────────────────────────────────────────┐
│                   前端 (React UI)                        │
│  - WebSocket 訂閱                                        │
│  - 訊息顯示與串流                                         │
│  - 工具視覺化                                            │
│  - Permission 交互                                       │
└──────────────────────┬──────────────────────────────────┘
                       │ HTTP REST + WebSocket (FeathersJS)
┌──────────────────────▼──────────────────────────────────┐
│              後端 Daemon (FeathersJS)                    │
│  - Session/Task/Message 服務                            │
│  - WebSocket 事件廣播                                   │
│  - Permission 決策協調                                   │
│  - MCP 端點 (自我存取)                                   │
└──────────────────────┬──────────────────────────────────┘
                       │ SDK API 呼叫
┌──────────────────────▼──────────────────────────────────┐
│           Agent SDK 層 (三選一)                          │
│  - @anthropic-ai/claude-agent-sdk (Claude Code)         │
│  - @openai/codex-sdk (Codex)                            │
│  - @google/gemini-cli-core (Gemini)                     │
└─────────────────────────────────────────────────────────┘
```

### 1.2 前端訊息處理流程

#### **階段 1: 用戶發送提示 (Prompt)**

```typescript
// apps/agor-ui/src/components/App/App.tsx:493
await client.service(`sessions/${sessionId}/prompt`).create({
  prompt: promptText,
  permissionMode: 'acceptEdits', // 或 'auto', 'ask'
});
```

**流程**:
1. 用戶在 `SessionDrawer` 輸入提示文字
2. 前端透過 FeathersJS 客戶端發送 POST 請求到 `/sessions/:id/prompt`
3. 後端立即返回 `{ taskId }`,不等待 AI 完成
4. 後端使用 `setImmediate()` 在背景執行 AI 請求

#### **階段 2: 後端建立 Task**

```typescript
// 後端創建 Task 記錄
const task = await tasksService.create({
  session_id: sessionId,
  status: 'running',
  description: promptText.substring(0, 120),
  full_prompt: promptText,
  message_range: {
    start_index: currentIndex + 1,
    start_timestamp: new Date().toISOString(),
  },
});
```

**WebSocket 事件**: `tasks.created` → 前端立即顯示 Task 卡片 (帶 spinner)

#### **階段 3: Agent SDK 串流執行**

後端呼叫對應的 Agent SDK:

**Claude Code**:
```typescript
const result = query({
  prompt,
  options: {
    cwd: session.worktree.absolute_path,
    systemPrompt: { type: 'preset', preset: 'claude_code' },
    settingSources: ['project'], // 自動載入 CLAUDE.md
    model: 'claude-sonnet-4-5-20250929',
    resume: session.sdk_session_id, // Session 接續
  },
});

// 串流處理
for await (const chunk of result) {
  if (chunk.type === 'text') {
    // 發送 streaming:chunk 事件到前端
  }
}
```

**Codex**:
```typescript
const stream = client.run({
  workdir: session.worktree.absolute_path,
  threadId: session.sdk_session_id, // Session 接續
});

// 事件驅動串流
for await (const event of stream) {
  if (event.type === 'item.updated') {
    // 發送 streaming:chunk 事件到前端
  }
}
```

**Gemini**:
```typescript
const stream = await client.sendMessageStream(prompt);

// 13 種事件類型
for await (const event of stream) {
  if (event.type === 'content') {
    // 發送 streaming:chunk 事件到前端
  }
}
```

#### **階段 4: 前端接收串流訊息**

**Hooks 訂閱**:
```typescript
// apps/agor-ui/src/hooks/useStreamingMessages.ts
const handleStreamingChunk = (data: { message_id, chunk }) => {
  setStreamingMessages((prev) => {
    const existing = prev.get(data.message_id);
    return new Map(prev).set(data.message_id, {
      message_id: data.message_id,
      content: (existing?.content || '') + data.chunk, // 累加文字
      isStreaming: true,
    });
  });
};

client.service('messages').on('streaming:chunk', handleStreamingChunk);
```

**渲染打字機效果**:
```typescript
// apps/agor-ui/src/components/MessageBlock/MessageBlock.tsx
<Bubble
  typing={isStreaming ? { step: 5, interval: 20 } : false}
  content={streamingContent}
/>
```

#### **階段 5: 訊息持久化**

串流結束後,後端將完整訊息寫入資料庫:

```typescript
const assistantMessage = await messagesService.create({
  session_id: sessionId,
  role: 'assistant',
  content: fullContent,
  task_id: taskId,
  tool_uses: [...], // 工具呼叫記錄
});
```

**WebSocket 事件**: `messages.created` → 前端將串流訊息替換為 DB 版本 (無閃爍)

#### **階段 6: Task 完成**

```typescript
await tasksService.patch(taskId, {
  status: 'completed',
  message_range: {
    ...existing,
    end_index: lastMessageIndex,
    end_timestamp: new Date().toISOString(),
  },
});
```

**WebSocket 事件**: `tasks.patched` → 前端移除 spinner,顯示完成狀態

---

### 1.3 前端核心 Hooks

#### **useMessages** - 歷史訊息獲取
```typescript
// apps/agor-ui/src/hooks/useMessages.ts
const { messages, loading } = useMessages(sessionId);
```

**功能**:
- 獲取 session 的所有歷史訊息 (最多 1000 筆)
- 按 `index` 欄位升序排序
- 訂閱 `created/patched/updated/removed` 事件
- 自動去重 (檢查 `message_id`)

**關鍵邏輯**:
```typescript
const handleMessageCreated = (message: Message) => {
  if (message.session_id === sessionId) {
    setMessages((prev) => {
      // 避免重複
      if (prev.some((m) => m.message_id === message.message_id)) {
        return prev;
      }
      // 插入並重新排序
      const newMessages = [...prev, message];
      return [...newMessages].sort((a, b) => a.index - b.index);
    });
  }
};
```

#### **useStreamingMessages** - 即時串流追蹤
```typescript
// apps/agor-ui/src/hooks/useStreamingMessages.ts
const streamingMessages = useStreamingMessages(sessionId);
```

**事件處理**:
- `streaming:start` - 初始化空內容
- `streaming:chunk` - 累加文字片段
- `streaming:end` - 標記完成,等待 DB 確認
- `streaming:error` - 錯誤處理
- `thinking:start/chunk/end` - Extended Thinking 模式

**平滑過渡機制**:
```typescript
// 串流結束後不立即移除,等待 DB 持久化
const handleStreamingEnd = (data: { message_id }) => {
  setStreamingMessages((prev) => {
    const updated = new Map(prev);
    const msg = updated.get(data.message_id);
    if (msg) {
      updated.set(data.message_id, { ...msg, isStreaming: false });
    }
    return updated;
  });
};

// messages.created 事件觸發時才移除串流訊息
const handleMessageCreated = (message: Message) => {
  setStreamingMessages((prev) => {
    const updated = new Map(prev);
    updated.delete(message.message_id); // 移除串流版本
    return updated;
  });
};
```

#### **useTaskMessages** - 按需載入優化
```typescript
// apps/agor-ui/src/hooks/useTaskMessages.ts
const { messages, loadMessages } = useTaskMessages(sessionId, taskId);
```

**效能優化**:
- 只在 Task 展開時才載入訊息
- 使用 `flushSync()` 強制同步渲染
- 適用於長對話 session (100+ tasks)

#### **useTaskEvents** - 工具執行狀態
```typescript
// apps/agor-ui/src/hooks/useTaskEvents.ts
const { runningTools } = useTaskEvents(sessionId);
```

**事件**:
- `tool:start` - 顯示工具執行中 spinner
- `tool:complete` - 2 秒後移除指示器

---

## 2. Session API 設計

### 2.1 Session 資料模型

```typescript
interface Session {
  session_id: SessionID;                    // UUIDv7 主鍵
  agentic_tool: 'claude' | 'codex' | 'gemini'; // SDK 選擇
  worktree_id: WorktreeID;                  // 必需:工作樹 FK
  status: 'idle' | 'running' | 'completed' | 'failed';
  sdk_session_id?: string;                  // SDK 內部 session ID (用於接續)
  title?: string;                           // Session 標題
  permission_config?: {
    mode: PermissionMode;                   // 權限模式
    codex?: {                               // Codex 特有設定
      sandboxMode: 'read-only' | 'workspace-write' | 'danger-full-access';
      approvalPolicy: 'untrusted' | 'on-request' | 'on-failure' | 'never';
      networkAccess: boolean;
    };
    updated_at: string;
  };
  model_config?: {
    mode: 'exact' | 'latest';               // 模型選擇模式
    model?: string;                         // 具體模型名稱
    updated_at?: string;
  };
  mcp_server_ids?: string[];                // MCP 伺服器 UUID 列表
  created_at: Date;
  updated_at: Date;
}
```

### 2.2 前端 API 呼叫方式

#### **建立 Session**
```typescript
// apps/agor-ui/src/hooks/useSessionActions.ts
const newSession = await client.service('sessions').create({
  agentic_tool: 'claude', // 或 'codex', 'gemini'
  worktree_id: selectedWorktreeId,
  title: 'My Session',
  permission_config: {
    mode: 'acceptEdits',
    codex: {
      sandboxMode: 'workspace-write',
      approvalPolicy: 'on-request',
      networkAccess: false,
    },
  },
  model_config: {
    mode: 'latest',
    model: 'claude-sonnet-4-5-20250929',
  },
  mcp_server_ids: ['mcp-server-uuid-1', 'mcp-server-uuid-2'],
});
```

**WebSocket 廣播**: `sessions.created` → 所有客戶端收到新 session

#### **發送 Prompt**
```typescript
// apps/agor-ui/src/components/App/App.tsx
await client.service(`sessions/${sessionId}/prompt`).create({
  prompt: 'Implement user authentication',
  permissionMode: 'acceptEdits',
});
```

**返回值**:
```typescript
{
  taskId: 'task-uuid',
  message: 'Task created and executing in background'
}
```

#### **Fork Session (分支)**
```typescript
// 建立分支 session (共享歷史到分叉點)
const forkedSession = await client
  .service(`sessions/${sessionId}/fork`)
  .create({ prompt: 'Try alternative approach' });

// 發送 prompt 執行
await client
  .service(`sessions/${forkedSession.session_id}/prompt`)
  .create({ prompt: 'Try alternative approach' });
```

**特性**:
- 新 session 繼承 worktree、permission、model 設定
- `forked_from_session_id` 指向原 session
- 前端顯示 fork 圖標和父子關係

#### **Spawn Session (子 session)**
```typescript
// 建立子 session (父子關係)
const spawnedSession = await client
  .service(`sessions/${sessionId}/spawn`)
  .create({ prompt: 'Research best authentication libraries' });

// 發送 prompt 執行
await client
  .service(`sessions/${spawnedSession.session_id}/prompt`)
  .create({ prompt: 'Research best authentication libraries' });
```

**特性**:
- 子 session 有獨立的對話歷史
- `parent_session_id` 指向父 session
- 父 session 可透過 MCP 工具獲取子 session 結果

#### **更新 Session 設定**
```typescript
// 更新 permission mode
await client.service('sessions').patch(sessionId, {
  permission_config: {
    mode: 'bypassPermissions',
    updated_at: new Date().toISOString(),
  },
});
```

**WebSocket 廣播**: `sessions.patched` → 所有客戶端同步更新

#### **訊息排隊 (Session Running)**
```typescript
// Session 運行中時,訊息加入佇列
const response = await client
  .service(`/sessions/${sessionId}/messages/queue`)
  .create({ prompt: 'Additional request' });

// 返回值
{
  message: {
    queue_position: 2,
    message_id: 'msg-uuid',
  }
}
```

**WebSocket 事件**: `messages.queued` → 前端顯示佇列位置提示

---

### 2.3 WebSocket 事件訂閱模式

```typescript
// apps/agor-ui/src/hooks/useSessions.ts
useEffect(() => {
  if (!client) return;

  const sessionsService = client.service('sessions');

  // 訂閱事件
  sessionsService.on('created', handleSessionCreated);
  sessionsService.on('patched', handleSessionPatched);
  sessionsService.on('updated', handleSessionUpdated);
  sessionsService.on('removed', handleSessionRemoved);

  // 清理函數
  return () => {
    sessionsService.removeListener('created', handleSessionCreated);
    sessionsService.removeListener('patched', handleSessionPatched);
    sessionsService.removeListener('updated', handleSessionUpdated);
    sessionsService.removeListener('removed', handleSessionRemoved);
  };
}, [client]);
```

**事件類型總覽**:

| 服務 | 事件 | 觸發時機 | 前端處理 |
|------|------|---------|---------|
| `sessions` | `created` | 新 session 建立 | 加入列表 |
| `sessions` | `patched` | Session 更新 | 更新狀態 |
| `tasks` | `created` | Task 建立 | 顯示 Task 卡片 |
| `tasks` | `patched` | Task 更新 (狀態/權限) | 更新 UI |
| `messages` | `created` | 訊息持久化到 DB | 替換串流版本 |
| `messages` | `streaming:start` | 開始串流 | 初始化空內容 |
| `messages` | `streaming:chunk` | 串流片段 | 累加文字 |
| `messages` | `streaming:end` | 串流結束 | 標記完成 |
| `messages` | `thinking:start` | Extended Thinking 開始 | 顯示 thinking 區塊 |
| `messages` | `thinking:chunk` | Thinking 片段 | 累加推理內容 |
| `tasks` | `tool:start` | 工具開始執行 | 顯示 spinner |
| `tasks` | `tool:complete` | 工具完成 | 移除 spinner |

---

## 3. 訊息中斷與接續機制

### 3.1 Session 接續 (Resumption)

Agor 透過 `sdk_session_id` 欄位實現對話接續,不同 SDK 有不同的實現方式:

#### **Claude Code**
```typescript
// 後端儲存 SDK session ID
session.sdk_session_id = result.sessionId; // SDK 返回的 UUID

// 接續對話時傳遞
const result = query({
  prompt,
  options: {
    resume: session.sdk_session_id, // 傳遞已儲存的 ID
    // ... 其他選項
  },
});
```

**特性**:
- SDK 自動管理對話歷史
- 支援跨 Agor session 的對話延續
- Session ID 由 SDK 生成並返回

#### **Codex**
```typescript
// 後端儲存 thread ID
session.sdk_session_id = threadId;

// 接續對話
const stream = client.run({
  workdir: session.worktree.absolute_path,
  threadId: session.sdk_session_id, // 傳遞 thread ID
});
```

**特性**:
- Thread ID 映射到 Codex 的對話線程
- 透過 `resumeThread(threadId)` API 恢復
- Thread 儲存在 Codex 後端

#### **Gemini**
```typescript
// SDK 自動持久化到檔案系統
// ~/.gemini/tmp/<project_hash>/chats/session-*.json

// ChatRecordingService 自動處理 session 接續
const client = new GeminiClient({
  cwd: session.worktree.absolute_path,
  // SDK 內部讀取 session 歷史
});

// 手動載入歷史 (可選)
client.setHistory(previousMessages);
```

**特性**:
- SDK 自動將對話儲存到本地檔案
- 透過 `ChatRecordingService` 管理 session 持久化
- Agor 不需要手動管理歷史記錄
- SDK 生成自己的 session ID (與 Agor 的 sessionId 不同)

### 3.2 訊息歷史管理

前端透過兩種方式獲取訊息:

#### **完整歷史載入**
```typescript
// apps/agor-ui/src/hooks/useMessages.ts
const messages = await client.service('messages').find({
  query: {
    session_id: sessionId,
    $limit: 1000,
    $sort: { index: 1 }, // 按順序排序
  },
});
```

#### **增量訂閱**
```typescript
// 訂閱新訊息
client.service('messages').on('created', (message) => {
  if (message.session_id === sessionId) {
    setMessages((prev) => [...prev, message].sort((a, b) => a.index - b.index));
  }
});
```

### 3.3 中斷處理機制

#### **用戶主動中斷**
```typescript
// apps/agor-ui/src/components/SessionDrawer/SessionDrawer.tsx
const handleStop = async () => {
  await client.service('sessions').patch(sessionId, {
    status: 'idle',
  });

  // 後端會中斷 Agent SDK 執行
  // 前端接收 sessions.patched 事件更新 UI
};
```

**前端顯示**:
- 停止按鈕 (`StopOutlined`) 在 session running 時顯示
- 點擊後變更狀態為 `idle`
- 清除所有 running tasks

#### **網路斷線處理**
```typescript
// apps/agor-ui/src/contexts/ConnectionContext.tsx
const [connected, setConnected] = useState(false);

useEffect(() => {
  client.io.on('connect', () => setConnected(true));
  client.io.on('disconnect', () => setConnected(false));
}, [client]);
```

**斷線時行為**:
- 顯示連線狀態指示器 (`ConnectionStatus`)
- 禁用所有輸入框和按鈕
- 重連後自動重新訂閱事件
- 串流訊息可能丟失 (後端會持久化完整訊息)

#### **錯誤恢復**
```typescript
// streaming:error 事件處理
const handleStreamingError = (data: { message_id, error }) => {
  setStreamingMessages((prev) => {
    const updated = new Map(prev);
    const msg = updated.get(data.message_id);
    if (msg) {
      updated.set(data.message_id, {
        ...msg,
        content: msg.content + '\n\n❌ Error: ' + data.error,
        isStreaming: false,
      });
    }
    return updated;
  });
};
```

### 3.4 Context Window 管理

前端顯示 session 的 context window 使用狀況:

```typescript
// apps/agor-ui/src/utils/contextWindow.ts
export function getContextWindowGradient(
  inputTokens: number,
  maxTokens: number
): string {
  const usage = inputTokens / maxTokens;

  if (usage < 0.5) return 'linear-gradient(90deg, #52c41a, #73d13d)'; // 綠色
  if (usage < 0.75) return 'linear-gradient(90deg, #faad14, #ffc53d)'; // 橘色
  return 'linear-gradient(90deg, #f5222d, #ff4d4f)'; // 紅色 (警告)
}
```

**前端元件**:
```typescript
// apps/agor-ui/src/components/Pill/Pill.tsx
<ContextWindowPill
  inputTokens={session.total_input_tokens}
  maxTokens={200000}
/>
```

**視覺提示**:
- 50% 以下: 綠色 (安全)
- 50-75%: 橘色 (注意)
- 75% 以上: 紅色 (接近上限,考慮 fork)

---

## 4. Permission 系統設計

### 4.1 三層權限架構

Agor 的權限系統遵循三層檢查順序:

```
1. ~/.claude/settings.json (用戶級) - SDK 優先檢查
        ↓ (未匹配)
2. .claude/settings.json (專案級) - SDK 次要檢查
        ↓ (未匹配)
3. Session 記憶體規則 - SDK 暫存
        ↓ (未匹配)
4. Agor UI 提示 - canUseTool callback
```

**關鍵設計**: Agor 使用 `canUseTool` callback,只在 SDK 所有檢查都未匹配時才觸發,確保尊重用戶既有設定。

### 4.2 前端 Permission 流程

#### **階段 1: SDK 觸發 canUseTool**

後端 (Claude Code 為例):
```typescript
// packages/core/src/tools/claude/permissions/permission-hooks.ts
export function createCanUseToolCallback(sessionId, taskId, deps) {
  return async (toolName, toolInput, options) => {
    // 建立 permission request 訊息
    const permissionMessage = await createPermissionRequestMessage(
      toolName,
      toolInput
    );

    // 更新 task 狀態為 awaiting_permission
    await tasksService.patch(taskId, {
      status: 'awaiting_permission',
      permission_request: {
        request_id: requestId,
        tool_name: toolName,
        tool_input: toolInput,
        requested_at: new Date().toISOString(),
      },
    });

    // 等待用戶決策 (Promise 暫停 SDK 執行)
    const decision = await permissionService.waitForDecision(
      requestId,
      taskId,
      signal
    );

    // 返回給 SDK
    return {
      behavior: decision.allow ? 'allow' : 'deny',
      updatedInput: toolInput,
      updatedPermissions: decision.remember ? [{
        kind: 'addRules',
        rules: [toolName],
        behavior: 'allow',
        destination: decision.scope === 'project' ? 'projectSettings' : 'session',
      }] : undefined,
    };
  };
}
```

#### **階段 2: 前端接收 WebSocket 事件**

```typescript
// apps/agor-ui/src/hooks/useTasks.ts
const handleTaskPatched = (task: Task) => {
  setTasks((prev) => {
    const updated = { ...prev };
    const sessionTasks = updated[task.session_id] || [];
    const index = sessionTasks.findIndex((t) => t.task_id === task.task_id);

    if (index >= 0) {
      sessionTasks[index] = task;
    }

    updated[task.session_id] = sessionTasks;
    return updated;
  });
};

client.service('tasks').on('patched', handleTaskPatched);
```

**Task 狀態變更**: `running` → `awaiting_permission`

#### **階段 3: 渲染 Permission Request UI**

```typescript
// apps/agor-ui/src/components/PermissionRequestBlock/PermissionRequestBlock.tsx
export const PermissionRequestBlock: React.FC<Props> = ({
  task,
  isActive, // 是否為第一個 pending permission
  onApprove,
  onDeny,
}) => {
  const [scope, setScope] = useState<PermissionScope>(PermissionScope.ONCE);

  // 只有 isActive 的 permission 可以互動
  if (!isActive && task.status === 'awaiting_permission') {
    return <WaitingPermissionCard task={task} />;
  }

  return (
    <Card
      style={{
        borderColor: task.approved_at ? '#52c41a' :
                     task.denied_at ? '#f5222d' : '#faad14',
      }}
    >
      <Space direction="vertical" style={{ width: '100%' }}>
        <Typography.Title level={5}>
          🔒 Permission Required
        </Typography.Title>

        <Typography.Text>
          Tool: <Tag>{task.permission_request.tool_name}</Tag>
        </Typography.Text>

        {/* 工具參數展示 */}
        <Descriptions
          items={Object.entries(task.permission_request.tool_input).map(
            ([key, value]) => ({
              label: key,
              children: <pre>{JSON.stringify(value, null, 2)}</pre>,
            })
          )}
        />

        {/* Scope 選擇 */}
        <Radio.Group value={scope} onChange={(e) => setScope(e.target.value)}>
          <Radio value={PermissionScope.ONCE}>Allow once</Radio>
          <Radio value={PermissionScope.PROJECT}>
            Remember for this project
          </Radio>
          <Radio value={PermissionScope.USER}>
            Remember for all projects
          </Radio>
        </Radio.Group>

        {/* 批准/拒絕按鈕 */}
        <Space>
          <Button
            type="primary"
            onClick={() => onApprove(task.task_id, scope)}
          >
            Approve
          </Button>
          <Button danger onClick={() => onDeny(task.task_id)}>
            Deny
          </Button>
        </Space>
      </Space>
    </Card>
  );
};
```

**三種 Scope**:
- `ONCE` - 僅一次 (無持久化)
- `PROJECT` - 專案級 (寫入 `.claude/settings.json`)
- `USER` - 用戶級 (寫入 `~/.claude/settings.json`)

#### **階段 4: 發送決策到後端**

```typescript
// apps/agor-ui/src/components/App/App.tsx:314
const handlePermissionDecision = async (
  sessionId: string,
  requestId: string,
  taskId: string,
  allow: boolean,
  scope: PermissionScope
) => {
  await client.service(`sessions/${sessionId}/permission-decision`).create({
    requestId,
    taskId,
    allow,
    reason: allow ? 'Approved by user' : 'Denied by user',
    scope: allow ? scope : PermissionScope.ONCE, // 拒絕時強制 ONCE
  });
};
```

#### **階段 5: 後端處理決策**

```typescript
// apps/agor-daemon/src/index.ts
app.use(`sessions/:sessionId/permission-decision`, {
  async create(data: PermissionDecision) {
    // 更新 task 狀態
    await tasksService.patch(data.taskId, {
      status: data.allow ? 'running' : 'failed',
      approved_at: data.allow ? new Date().toISOString() : undefined,
      denied_at: !data.allow ? new Date().toISOString() : undefined,
      approved_by: data.decidedBy,
    });

    // 解析 waitForDecision Promise
    permissionService.resolvePermission(data);

    // canUseTool callback 返回,SDK 繼續執行
  },
});
```

**SDK 自動持久化**:
- `scope: 'project'` → SDK 寫入 `.claude/settings.json`
- `scope: 'session'` → SDK 儲存在記憶體
- `scope: 'once'` → 無持久化

#### **階段 6: 前端顯示決策結果**

```typescript
// WebSocket 事件 tasks.patched 觸發 UI 更新
<PermissionRequestBlock
  task={task} // status = 'running', approved_at = timestamp
  isActive={false} // 不再是 pending
/>
```

**視覺狀態**:
- **Pending (Active)**: 黃色卡片,顯示按鈕
- **Pending (Waiting)**: 灰色卡片,顯示等待訊息
- **Approved**: 綠色卡片,顯示批准時間 (收合)
- **Denied**: 紅色卡片,顯示拒絕時間 (收合)

### 4.3 多用戶協作

所有連線的客戶端都會即時看到 permission requests:

```
用戶 A 觸發工具使用
    ↓ tasks.patched (status: awaiting_permission)
所有客戶端顯示黃色 permission 卡片
    ↓
用戶 B 點擊 "Approve"
    ↓ permission-decision.create
後端更新 task.approved_by = user-b-id
    ↓ tasks.patched (status: running, approved_at)
所有客戶端看到綠色 "Permission Approved" 卡片
    ↓
SDK 恢復執行
```

**審計追蹤**: 所有 permission 決策都保留在 task 歷史中,可追溯誰批准/拒絕。

### 4.4 Permission Mode 選擇器

前端提供下拉選單切換權限模式:

```typescript
// apps/agor-ui/src/components/PermissionModeSelector/PermissionModeSelector.tsx
<PermissionModeSelector
  value={session.permission_config?.mode}
  onChange={handlePermissionModeChange}
  agentic_tool={session.agentic_tool}
  compact={true} // 收合模式 (下拉選單)
/>
```

**模式選項** (依 SDK 不同):

**Claude Code**:
- `default` - 每次工具都詢問
- `acceptEdits` - 自動批准檔案編輯
- `bypassPermissions` - 全部自動批准
- `plan` - 僅產生計畫,不執行

**Codex** (雙下拉選單):
- Sandbox: `read-only` / `workspace-write` / `danger-full-access`
- Approval: `untrusted` / `on-request` / `on-failure` / `never`

**Gemini**:
- `default` - 每次工具都詢問
- `acceptEdits` (顯示為 autoEdit) - 自動批准檔案編輯
- `bypassPermissions` (顯示為 yolo) - 全部自動批准

**即時更新**:
```typescript
const handlePermissionModeChange = async (mode: PermissionMode) => {
  await client.service('sessions').patch(sessionId, {
    permission_config: {
      mode,
      updated_at: new Date().toISOString(),
    },
  });

  // WebSocket 廣播 → 所有客戶端同步
};
```

---

## 5. 三種 SDK 的前端處理差異

### 5.1 Claude Code (claude-agent-sdk)

#### **SDK 特性**
- ✅ 官方 SDK,功能最完整
- ✅ CLAUDE.md 自動載入
- ✅ Preset 系統提示 (claude_code)
- ✅ PreToolUse hook (互動式權限)
- ✅ Session 接續 (resume 參數)
- ✅ Token 追蹤 (input/output/cache)
- ✅ Extended Thinking 支援

#### **前端特有元件**

**1. Token 使用顯示**
```typescript
// apps/agor-ui/src/components/Pill/Pill.tsx
<TokenCountPill
  inputTokens={task.usage?.input_tokens}
  outputTokens={task.usage?.output_tokens}
  cacheReadTokens={task.usage?.cache_read_tokens}
  cacheCreationTokens={task.usage?.cache_creation_tokens}
/>
```

**視覺化**:
- 金色圖標 (`ThunderboltOutlined`)
- Tooltip 顯示成本明細
- Session 總計 token 顯示

**2. Thinking Block**
```typescript
// apps/agor-ui/src/components/ThinkingBlock/ThinkingBlock.tsx
<ThinkingBlock
  content={message.thinkingContent}
  isStreaming={message.isThinking}
/>
```

**特性**:
- 可折疊區塊
- 串流支援 (`thinking:chunk` 事件)
- 灰色背景區別於一般回覆

**3. Permission Request (完整)**
```typescript
<PermissionRequestBlock
  task={task}
  isActive={isFirstPendingPermission}
  onApprove={handleApprove}
  onDeny={handleDeny}
/>
```

**互動流程**:
- 即時 UI 提示
- 三種 scope 選擇
- SDK 自動持久化

#### **前端訊息處理**

**串流事件**:
```typescript
// streaming:start → 初始化
// streaming:chunk → 累加文字
// thinking:start → 初始化 thinking
// thinking:chunk → 累加推理內容
// streaming:end → 完成
```

**特殊 Content Blocks**:
```typescript
// 處理 Extended Thinking
if (block.type === 'thinking') {
  return <ThinkingBlock content={block.thinking} />;
}

// 處理工具使用
if (block.type === 'tool_use') {
  return <ToolUseRenderer tool={block} />;
}
```

---

### 5.2 Codex (openai/codex-sdk)

#### **SDK 特性**
- ✅ 官方 SDK
- ⚠️ 雙設定系統 (sandboxMode + approvalPolicy)
- ✅ Thread ID 接續
- ❌ 無互動式權限 (僅 config)
- ❌ Token 使用未暴露
- ⚠️ MCP 僅支援 STDIO

#### **前端特有元件**

**1. Dual Permission Selector**
```typescript
// apps/agor-ui/src/components/PermissionModeSelector/PermissionModeSelector.tsx
{agentic_tool === 'codex' && (
  <Space size={8}>
    <Select
      value={codexSandboxMode}
      onChange={(val) => onCodexChange(val, codexApprovalPolicy)}
      options={CODEX_SANDBOX_MODES} // read-only, workspace-write, full-access
    />
    <Select
      value={codexApprovalPolicy}
      onChange={(val) => onCodexChange(codexSandboxMode, val)}
      options={CODEX_APPROVAL_POLICIES} // untrusted, on-request, on-failure, never
    />
  </Space>
)}
```

**前端顯示**:
- 兩個下拉選單並排
- Sandbox 控制檔案系統存取
- Approval 控制工具執行策略

**2. Network Access Toggle**
```typescript
// apps/agor-ui/src/components/CodexNetworkAccessToggle/CodexNetworkAccessToggle.tsx
<Switch
  checked={session.permission_config?.codex?.networkAccess}
  onChange={handleToggle}
  checkedChildren="Network ON"
  unCheckedChildren="Network OFF"
/>
```

**功能**:
- 控制 Codex 是否可存取網路
- 更新 `~/.codex/config.toml`
- WebSocket 同步到所有客戶端

**3. 簡化的 Permission UI**
- ❌ 無互動式 permission requests
- ✅ 在 SessionDrawer 顯示當前設定
- 用戶需在 Codex 內使用 `/approvals` 指令

#### **前端訊息處理**

**串流事件**:
```typescript
// item.updated → 累加文字
// item.started → 工具開始
// item.completed → 工具完成
// turn.completed → 回合完成
```

**工具渲染** (基礎):
```typescript
// Codex 工具事件較簡單
if (event.type === 'command_execution') {
  return <BashRenderer tool={event.data} />;
}

// 檔案變更
if (event.type === 'file_change') {
  return <FileEditBlock file={event.data} />;
}
```

**限制**:
- 工具視覺化較陽春 (無 Claude Code 的語義分組)
- 無 Extended Thinking
- 無 token 追蹤

#### **前端 API 呼叫差異**

```typescript
// 建立 Codex session 需要額外參數
await client.service('sessions').create({
  agentic_tool: 'codex',
  worktree_id: worktreeId,
  permission_config: {
    mode: 'auto', // 映射到內部設定
    codex: {
      sandboxMode: 'workspace-write',
      approvalPolicy: 'on-request',
      networkAccess: false,
    },
  },
});
```

---

### 5.3 Gemini (google/gemini-cli-core)

#### **SDK 特性**
- ✅ 官方 SDK
- ✅ 13 種事件類型 (最豐富)
- ✅ ChatRecordingService (自動持久化)
- ✅ 模型選擇豐富 (Pro/Flash/Flash-Lite)
- ⚠️ 工具視覺化未實作
- ⚠️ 中途切換權限未測試
- ✅ MCP 完整支援 (HTTP/STDIO/TCP)

#### **SDK 特性 (已實作)**

**1. Session 接續**
```typescript
// SDK 自動持久化到檔案系統
// ~/.gemini/tmp/<project_hash>/chats/session-*.json

// 前端不需要手動管理
// SDK 的 ChatRecordingService 自動載入歷史
```

**特性**:
- SDK 生成自己的 session ID
- 自動儲存到本地檔案
- 跨 Agor session 的對話延續

**2. MCP 伺服器整合**
```typescript
// 前端 MCP 選擇器
<MCPServerSelect
  value={session.mcp_server_ids}
  onChange={handleMCPChange}
  mcpServers={mcpServers}
  sessionId={session.session_id}
/>
```

**階層式 Scoping**:
- Global → Repo → Session
- 包含 Agor MCP (daemon 自我存取) + 用戶設定的 MCP
- 支援工具過濾 (`includeTools` / `excludeTools`)

#### **前端特有元件**

**1. 模型選擇器**
```typescript
// apps/agor-ui/src/components/ModelSelector/ModelSelector.tsx
<ModelSelector
  value={session.model_config?.model}
  onChange={handleModelChange}
  agentic_tool="gemini"
/>
```

**選項**:
- `gemini-2.5-pro` - 最強推理能力
- `gemini-2.5-flash` - 平衡 (便宜 10 倍)
- `gemini-2.5-flash-lite` - 高吞吐量 (最便宜)

**前端顯示**:
- 下拉選單顯示價格比較
- Tooltip 提示使用場景

**2. Permission Selector (基礎)**
```typescript
// 3 種模式 (與 Claude Code 類似命名)
const GEMINI_MODES = [
  { mode: 'default', label: 'default' },
  { mode: 'acceptEdits', label: 'autoEdit' }, // Gemini SDK 名稱
  { mode: 'bypassPermissions', label: 'yolo' }, // Gemini SDK 名稱
];
```

#### **前端訊息處理**

**13 種事件類型**:
```typescript
// content → 文字內容
// tool_call_request → 工具請求
// tool_call_response → 工具結果
// thought → Agent 推理 (類似 Claude Thinking)
// error → 錯誤
// chat_compressed → Context 壓縮
// citation → 引用來源
// retry → 重試
// loop_detected → 迴圈偵測
// tool_call_confirmation → 工具確認
// tool_filtering_warning → 工具過濾警告
// session_file_not_found_warning → Session 檔案未找到
// insufficient_files_read → 檔案讀取不足
```

**串流處理**:
```typescript
// 前端目前僅處理基礎事件
if (event.type === 'content') {
  // 累加文字
  streamingMessages.set(messageId, {
    content: existing.content + event.data.text,
  });
}

if (event.type === 'thought') {
  // 顯示推理過程 (類似 Claude Thinking)
  // ❌ 未實作 ThoughtBlock 組件
}
```

#### **前端待實作功能**

**1. 工具視覺化**
```typescript
// ❌ 尚未實作 Gemini 特有的工具渲染器
// 目前使用通用 ToolUseRenderer

// 應實作:
// - LoopDetectionBlock - 顯示迴圈警告
// - ThoughtBlock - 顯示 Agent 推理
// - CitationBlock - 顯示引用來源
// - CompressionIndicator - 顯示 context 壓縮
```

**2. 互動式 Permission**
```typescript
// ⚠️ 未測試 tool_call_confirmation 事件
// 可能支援類似 Claude PreToolUse 的互動

// 需要研究:
// - 如何 hook 到 approval flow
// - 是否支援 updatedPermissions 持久化
```

**3. 中途切換權限**
```typescript
// ⚠️ 未測試是否需要重建 GeminiClient

// 目前實作:
await client.service('sessions').patch(sessionId, {
  permission_config: { mode: 'bypassPermissions' },
});

// 可能需要:
// - 重建 SDK client instance
// - 或 SDK 支援動態更新 ApprovalMode
```

---

### 5.4 三種 SDK 前端差異總表

| 功能 | Claude Code | Codex | Gemini |
|------|------------|-------|--------|
| **Permission UI** | ✅ 完整 (3 scope) | ❌ Config-only (2 dropdown) | ⚠️ 基礎 (3 mode) |
| **Token 顯示** | ✅ 完整成本明細 | ❌ SDK 未暴露 | ⚠️ 基礎設施已備 |
| **Thinking 視覺化** | ✅ ThinkingBlock | ❌ 無 | ⚠️ 事件存在,未實作 |
| **工具渲染** | ✅ 語義分組 | ⚠️ 基礎 | ❌ 未實作 |
| **模型選擇** | ✅ Dropdown | ⚠️ UI-only | ✅ Dropdown (3 選項) |
| **MCP 整合** | ✅ HTTP/STDIO/SSE | ⚠️ STDIO only | ✅ HTTP/STDIO/TCP |
| **Session 接續** | ✅ resume 參數 | ✅ threadId | ✅ 自動持久化 |
| **串流平滑度** | ✅ Token-level | ✅ Event-based | ✅ Token-level |
| **中途切換權限** | ✅ 已測試 | ✅ 已測試 | ⚠️ 未測試 |
| **網路存取控制** | ❌ 無 | ✅ Toggle | ❌ 無 |
| **Import Session** | ✅ JSONL 解析 | ❌ 格式未知 | ❌ 未實作 |

### 5.5 前端統一處理策略

儘管 SDK 不同,前端使用統一的資料模型和組件:

#### **統一 Message 模型**
```typescript
interface Message {
  message_id: MessageID;
  role: 'user' | 'assistant' | 'system';
  content: string | ContentBlock[];
  tool_uses?: ToolUse[]; // 統一工具格式
  task_id?: TaskID;
  // ... SDK-agnostic 欄位
}
```

#### **統一 Task 模型**
```typescript
interface Task {
  task_id: TaskID;
  status: 'pending' | 'running' | 'completed' | 'failed' | 'awaiting_permission';
  permission_request?: PermissionRequest; // 所有 SDK 共用
  usage?: TokenUsage; // Claude Code 有值,其他為空
}
```

#### **條件渲染**
```typescript
// apps/agor-ui/src/components/TaskBlock/TaskBlock.tsx
{task.usage && session.agentic_tool === 'claude' && (
  <TokenCountPill {...task.usage} />
)}

{task.permission_request && (
  session.agentic_tool === 'claude' ? (
    <PermissionRequestBlock {...} /> // 完整版
  ) : (
    <SimplePermissionIndicator {...} /> // Codex/Gemini 簡化版
  )
)}
```

#### **Permission Mode 映射**
```typescript
// apps/agor-ui/src/components/PermissionModeSelector/PermissionModeSelector.tsx
const modes =
  agentic_tool === 'codex'
    ? CODEX_MODES
    : agentic_tool === 'gemini'
      ? GEMINI_MODES
      : CLAUDE_CODE_MODES;
```

**統一介面,差異化實作** - 前端保持一致的 UX,同時支援各 SDK 特性。

---

## 6. 關鍵文件路徑索引

### 6.1 Hooks (`apps/agor-ui/src/hooks/`)

| 檔案 | 功能 | 關鍵邏輯 |
|------|------|---------|
| `useMessages.ts` | 獲取 session 歷史訊息 | WebSocket 訂閱 `created/patched` |
| `useStreamingMessages.ts` | 追蹤即時串流 | `streaming:*` 和 `thinking:*` 事件 |
| `useTaskMessages.ts` | 按 task 懶加載訊息 | `flushSync()` 強制同步渲染 |
| `useTaskEvents.ts` | 工具執行狀態 | `tool:start/complete` 事件 |
| `useSessions.ts` | Session 列表和訂閱 | `sessions.created/patched` 事件 |
| `useTasks.ts` | Task 列表和訂閱 | `tasks.created/patched` 事件 |
| `useSessionActions.ts` | Session CRUD 操作 | `create/fork/spawn/prompt` API |

### 6.2 組件 (`apps/agor-ui/src/components/`)

#### **訊息顯示**
| 檔案 | 功能 |
|------|------|
| `ConversationView/ConversationView.tsx` | 對話視圖主組件 |
| `TaskBlock/TaskBlock.tsx` | Task 折疊卡片 (訊息分組) |
| `MessageBlock/MessageBlock.tsx` | 單條訊息渲染 |
| `ThinkingBlock/ThinkingBlock.tsx` | Extended Thinking 顯示 |

#### **工具視覺化**
| 檔案 | 功能 |
|------|------|
| `ToolUseRenderer/ToolUseRenderer.tsx` | 工具渲染器主入口 |
| `ToolUseRenderer/renderers/BashRenderer.tsx` | Bash 工具 (ANSI 顏色) |
| `ToolUseRenderer/renderers/TodoListRenderer.tsx` | TodoWrite 工具 |
| `ToolUseRenderer/renderers/index.ts` | 渲染器註冊表 |

#### **Permission 系統**
| 檔案 | 功能 |
|------|------|
| `PermissionRequestBlock/PermissionRequestBlock.tsx` | Permission UI (3 scope) |
| `PermissionModeSelector/PermissionModeSelector.tsx` | Permission mode 下拉選單 |
| `CodexNetworkAccessToggle/CodexNetworkAccessToggle.tsx` | Codex 網路存取開關 |

#### **Session 管理**
| 檔案 | 功能 |
|------|------|
| `SessionDrawer/SessionDrawer.tsx` | Session 詳情抽屜 (含輸入框) |
| `SessionCard/SessionCard.tsx` | Session 卡片 (Canvas 上) |
| `SessionSettingsModal/SessionSettingsModal.tsx` | Session 設定彈窗 |
| `NewSessionModal/NewSessionModal.tsx` | 建立 session 彈窗 |
| `ForkSpawnModal/ForkSpawnModal.tsx` | Fork/Spawn 彈窗 |

#### **輔助組件**
| 檔案 | 功能 |
|------|------|
| `Pill/Pill.tsx` | 各種狀態 pill (Token/Timer/Message Count) |
| `ModelSelector/ModelSelector.tsx` | 模型選擇下拉選單 |
| `MCPServerSelect/MCPServerSelect.tsx` | MCP 伺服器多選 |
| `AgentSelectionGrid/AgentSelectionGrid.tsx` | Agent 選擇網格 |

### 6.3 Context 文檔 (`context/concepts/`)

| 檔案 | 主題 |
|------|------|
| `architecture.md` | 系統架構、資料庫、API 設計 |
| `agent-integration.md` | Claude Agent SDK 整合 |
| `permissions.md` | 三層權限系統設計 |
| `conversation-ui.md` | Task-centric UI 設計 |
| `websockets.md` | 即時通訊實現 |
| `agentic-coding-tool-integrations.md` | 三種 SDK 功能比較 |

---

## 總結

### 核心設計原則

1. **Task-Centric Architecture** - 以 Task 為對話組織單位,清晰的訊息分組
2. **Progressive Enhancement** - 串流 → DB 持久化的平滑過渡,無閃爍體驗
3. **SDK-Agnostic Frontend** - 統一資料模型,支援多種 Agent SDK
4. **Real-Time Collaboration** - WebSocket 驅動的即時多用戶協作
5. **Respectful Permissions** - 尊重用戶既有設定,僅在必要時提示

### 前端技術亮點

- **React 18 flushSync** - 強制同步渲染,確保串流訊息即時顯示
- **Map-based Streaming State** - 高效管理多個並行串流訊息
- **註冊表模式** - 可擴展的工具渲染器架構
- **階層式 MCP Scoping** - Global → Repo → Session 的 MCP 伺服器管理
- **Dual Permission Control** - Codex 的雙下拉選單處理複雜權限策略

### 未來改進方向

1. **Gemini 工具視覺化** - 實作 Thought/Loop/Citation 組件
2. **統一 Token 追蹤** - 擴展到 Codex 和 Gemini (待 SDK 支援)
3. **Permission 審計面板** - 顯示所有歷史決策和自動規則
4. **Session Migration** - 在不同 SDK 間轉換 session 格式
5. **Board-Based Channels** - 按 Board 分隔 WebSocket 事件,提升效能

---

**文檔版本**: v1.0
**最後更新**: 2025-01-12
**維護者**: Agor 開發團隊
