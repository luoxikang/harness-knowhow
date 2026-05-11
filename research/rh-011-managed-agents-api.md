# RH-011: Managed Agents API 全貌

## 来源
- [ART-01] https://www.anthropic.com/engineering/managed-agents
- [DOC-02] https://platform.claude.com/docs/en/managed-agents/overview
- [DOC-03] https://platform.claude.com/docs/en/managed-agents/agent-setup
- [DOC-04] https://platform.claude.com/docs/en/managed-agents/sessions
- [DOC-05] https://platform.claude.com/docs/en/managed-agents/environments
- [DOC-06] https://platform.claude.com/docs/en/managed-agents/events-and-streaming
- [DOC-07] https://platform.claude.com/docs/en/managed-agents/tools

## 四层资源模型

```
Agent（定义）           Environment（模板）
┌────────────────┐    ┌────────────────────┐
│ model           │    │ packages           │
│ system prompt   │    │ networking         │
│ tools           │    │ (unrestricted/     │
│ mcp_servers     │    │  limited)          │
│ skills          │    │                    │
│ versioned ✓     │    │ not versioned      │
└────────┬───────┘    └─────────┬──────────┘
         │                      │
         └──────────┬───────────┘
                    ▼
Session（实例）       ← 引用 Agent + Environment
┌────────────────┐
│ status 状态机    │
│ vault_ids 凭据  │
│ 不可变事件日志   │
└────────┬───────┘
         │
         ▼
Events（通信）
┌───────────────────────────────┐
│ User:   message/interrupt/    │
│         tool_confirmation     │
│ Agent:  message/thinking/     │
│         tool_use/tool_result  │
│ Session: status_running/idle/ │
│         terminated/error      │
│ Span:   model_request_start/end│
└───────────────────────────────┘
```

## 完整 API 端点

```
Agents:
  POST   /v1/agents                     创建（版本化）
  GET    /v1/agents                     列表
  GET    /v1/agents/{id}                详情
  PUT    /v1/agents/{id}                更新（生成新版本）
  POST   /v1/agents/{id}/archive        归档
  GET    /v1/agents/{id}/versions       版本历史

Environments:
  POST   /v1/environments               创建
  GET    /v1/environments               列表
  GET    /v1/environments/{id}          详情
  POST   /v1/environments/{id}/archive  归档
  DELETE /v1/environments/{id}          删除

Sessions:
  POST   /v1/sessions                   创建（agent + environment_id + vault_ids）
  GET    /v1/sessions                   列表
  GET    /v1/sessions/{id}              详情（含 status）
  POST   /v1/sessions/{id}/archive      归档（只读）
  DELETE /v1/sessions/{id}              删除（清理容器+记录）

Events:
  POST   /v1/sessions/{id}/events       发送事件（user.message, user.interrupt...）
  GET    /v1/sessions/{id}/events/stream SSE 流（实时接收）
```

## Session 状态机

```
idle → running → idle → running → ...
   ↓              ↓
rescheduling  terminated
(自动重试)    (不可恢复错误)
```

## 事件类型全景

### User Events（客户端→服务端）
- `user.message` — 用户消息
- `user.interrupt` — 中断执行
- `user.custom_tool_result` — 自定义工具结果
- `user.tool_confirmation` — 权限确认
- `user.define_outcome` — 定义输出目标

### Agent Events（服务端→客户端）
- `agent.message` — Agent 响应（文本块）
- `agent.thinking` — 思考内容
- `agent.tool_use` — 调用内置工具
- `agent.tool_result` — 内置工具结果
- `agent.mcp_tool_use` — 调用 MCP 工具
- `agent.mcp_tool_result` — MCP 工具结果
- `agent.custom_tool_use` — 调用自定义工具
- `agent.thread_context_compacted` — 上下文压缩
- 多 Agent 消息（thread_message_sent/received）

### Session Events
- `session.status_running` — 执行中
- `session.status_idle` — 等待输入（含 stop_reason）
- `session.status_rescheduled` — 重试中
- `session.status_terminated` — 终止
- `session.error` — 错误（含 retry_status）
- 多 Agent Thread 状态事件

### Span Events（可观测性）
- `span.model_request_start` — LLM 调用开始
- `span.model_request_end` — LLM 调用结束（含 token 计数）
- `span.outcome_evaluation_*` — 输出评估

## Streaming 模型

```
1. 客户端开 SSE stream: GET /v1/sessions/{id}/events/stream
2. 客户端发事件: POST /v1/sessions/{id}/events (user.message)
3. 服务端通过 SSE 推送 agent/session/span 事件
4. session.status_idle 表示完成
```

每个事件有 `processed_at` 时间戳（null = 排队中尚未处理）。

## 设计原则（从文章和 API 中提取）

| # | 原则 | 体现 |
|---|------|------|
| P1 | 大脑和手分离 | Harness 调 LLM（大脑），Sandbox 执行工具（手） |
| P2 | Session 是不可变事件日志 | 所有通信走 Event，语义是追加 |
| P3 | 凭据不进沙盒 | vault_ids 在 Session 创建时注入，Agent 定义中无 token |
| P4 | Cattle not Pet | Harness/Sandbox 故障 → 重 provision |
| P5 | 懒加载 | 容器在需要时才 provision |
| P6 | 版本化 Agent | Agent 有 version + 版本历史 API |
| P7 | 稳定接口，可替换实现 | API 契约不变，后端实现可换 |
