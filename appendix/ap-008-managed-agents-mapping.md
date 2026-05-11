# AP-008: Managed Agents API 契约映射

## 关联
- [RA-001] 以 Pi 为 Harness 的参考架构
- [RH-011] Managed Agents API 全貌

## 资源映射

```
Anthropic API                    参考实现（Pi 方案）
─────────────                    ──────────────────
Agent（模型+提示词+工具+MCP）      Gateway agent_configs 表 + AgentVersionManager
Environment（包+网络+容器）        Gateway env_configs 表 → 选择 ExecutionEnv 类型
Session（引用 Agent+Env）          Pi AgentHarness + Session + ExecutionEnv
Events（不可变事件流）              Pi SessionTreeEntry → 格式转换
Vault（OAuth 凭据）                Gateway user_tokens 表 + getApiKeyAndHeaders

Stream（SSE）                      Gateway SSEController（订阅 Pi events）
```

## 逐端点映射

### Agents
```
POST   /v1/agents         → Gateway → agent_configs INSERT → 返回 agent_id + version
GET    /v1/agents         → Gateway → agent_configs SELECT
GET    /v1/agents/{id}    → Gateway → agent_configs WHERE id
PUT    /v1/agents/{id}    → Gateway → 新行 INSERT（新 version）→ 对比检测 no-op
POST   /v1/agents/{id}/archive → Gateway → SET archived_at
GET    /v1/agents/{id}/versions → Gateway → SELECT WHERE agent_id ORDER BY version
```

### Environments
```
POST   /v1/environments        → Gateway → env_configs INSERT
GET    /v1/environments        → Gateway → env_configs SELECT
GET    /v1/environments/{id}   → Gateway → env_configs WHERE id
POST   /v1/environments/{id}/archive → Gateway → SET archived_at
DELETE /v1/environments/{id}   → Gateway → DELETE（无 session 引用时）
```

### Sessions
```
POST   /v1/sessions        → Gateway:
                               1. resolve agent config → resolve model/tools
                               2. resolve env config → 选择 ExecutionEnv 类型
                               3. new AgentHarness(env, session, tools)
                               4. 存 session 元数据
                               5. 返回 session_id

GET    /v1/sessions        → Gateway → session 表 SELECT（支持分页）
GET    /v1/sessions/{id}   → Gateway → 查 session 元数据 + harness 状态
                               status 映射：Agent.Running ↔ status.running
POST   /v1/sessions/{id}/archive → Gateway → SET archived_at
DELETE /v1/sessions/{id}   → Gateway → dispose harness + env.cleanup() → DELETE
```

### Events
```
POST   /v1/sessions/{id}/events → Gateway:
   1. 解析 events 数组
   2. user.message       → harness.prompt(message)
   3. user.interrupt     → harness.agent.abort()
   4. user.tool_confirmation → 等待挂起的 permission ask → resolve
   5. user.custom_tool_result → 传递给挂起的 custom tool handler

GET    /v1/sessions/{id}/events/stream → Gateway:
   1. 创建 SSE stream
   2. 订阅 harness 所有事件（通过 AgentEvent + AgentHarnessOwnEvent）
   3. 格式转换 → Managed Agents 格式 → enqueue

GET    /v1/sessions/{id}/events → Gateway:
   1. session.getEntries() → SessionTreeEntry[]
   2. 转成 Managed Agents Event 格式返回
```

## 事件格式转换

```
Pi Event                            →  Managed Agents Event
────────────────────────────────────    ──────────────────────────
AgentEvent.message_start (user)     →  user.message (processed_at=null)
AgentEvent.message_start (asst)     →  agent.message
AgentEvent.message_update (thinking)→  agent.thinking
AgentEvent.tool_execution_start     →  agent.tool_use
AgentEvent.tool_execution_end       →  agent.tool_result
HarnessEvent.tool_call              →  (如果需要 confirmation: 发 tool_confirmation 事件)
HarnessEvent.before_provider_request→ span.model_request_start
HarnessEvent.after_provider_response→ span.model_request_end (含 token 用量)
HarnessEvent.session_before_compact → agent.thread_context_compacted
AgentEvent.agent_end                → session.status_idle (agent 停止)
```

## 无法直接映射的部分

| Anthropic API | 实现策略 |
|---------------|---------|
| `agent.mcp_tool_use/result` | MCPToolFactory 创建的工具在 Pi 中就是普通 AgentTool。事件可通过 tool_call/tool_result hook 区分 |
| `agent.custom_tool_use/result` | 同样的 event 格式，通过 tool name 前缀区分 |
| MultiAgent Thread 事件 | 当前 Pi 无此概念。可通过多个 AgentHarness + 协调逻辑实现 |
| `user.define_outcome` | 当前无等价。可作为 prompt 中的特殊指令实现 |
| `span.outcome_evaluation_*` | 当前无等价。需自己实现评估逻辑 |
