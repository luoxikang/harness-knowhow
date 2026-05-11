# AP-007: 可吸收的 OpenCode 设计清单

## 关联
- [RA-001] 以 Pi 为 Harness 的参考架构

## 核心原则

**吸收 OpenCode 的设计模式，不吸收 OpenCode 的代码。** 在 Pi 的 Gateway 层实现。

## 清单

### 1. InstanceStore 懒加载模式

**来源**：`src/project/instance-store.ts`

**设计**：`Map<key, Entry>` — 首次访问时创建，后续访问复用，Deferred 去重。

**吸收方式**：Gateway 的 SessionManager 中用同样的模式管理 `Map<userId:workspaceId, {harness, env, session}>`。

### 2. Workspace Target 代理模式

**来源**：`src/control-plane/types.ts`, `src/server/shared/workspace-routing.ts`

**设计**：Target { type: "local" | "remote", url } — 请求自动转发。

**吸收方式**：Gateway 的 `resolveEnvironment()` 方法。根据 workspace 配置创建对应的 ExecutionEnv（local → NodeExecutionEnv，remote → RemoteExecutionEnv）。

### 3. Permission Ruleset

**来源**：`src/config/permission.ts`

**设计**：Action: "allow" | "deny" | "ask"；Rule: { permission, pattern, action }；支持 pattern 匹配 + 继承。

**吸收方式**：Pi 的 `before_tool_call` hook 中实现权限检查。参数完全对应。

### 4. SubAgent 权限继承

**来源**：`src/agent/subagent-permissions.ts`

**设计**：`deriveSubagentSessionPermission(parent, subagent)` — 父 session deny + 父 agent deny → 子。

**吸收方式**：创建子 Session（AgentHarness 副本）时，调用相同的权限合并逻辑。

### 5. SSE Event Stream

**来源**：`src/server/routes/instance/httpapi/event.ts`

**设计**：`GET /event` → SSE stream，10 秒 heartbeat，`text/event-stream` 格式。

**吸收方式**：Gateway 的 SSEController。订阅 Pi AgentHarness 的所有事件 → 转 Managed Agents 格式 → SSE 输出。

### 6. Agent 版本化

**来源**：[DOC-03] Managed Agents Agent Setup

**设计**：PUT 生成新 version；Session 可 pin 到版本或跟随 latest；版本历史 API。

**吸收方式**：Gateway 的 `agent_versions` 表 + AgentVersionManager。Pi 不需要知道版本存在。

### 7. Span 事件（计费钩子）

**来源**：[DOC-06] Managed Agents Events

**设计**：`span.model_request_start/end` — 每次 LLM 调用的 token 用量。

**吸收方式**：Pi 的 `after_provider_response` hook 中包含 token 用量信息。在 Gateway 层扣除余额。

### 8. SQLite Session Schema

**来源**：`src/session/session.sql.ts`

**设计**：session / message / part / todo / permission / event 表，Drizzle ORM。

**吸收方式**：实现 Pi 的 SessionStorage 接口，底层用同样的 SQLite schema。继承索引设计（session_idx, session_type_idx, time_created_idx）。

### 9. MCP 集成模式

**来源**：`src/mcp/`, `src/server/routes/instance/httpapi/groups/mcp.ts`

**设计**：listTools → wrap → inject tools 到 agent；OAuth token 管理。

**吸收方式**：Gateway 的 MCPToolFactory。连接 MCP Server → 获取 tool list → 包装成 Pi AgentTool → 合并到 AgentHarness 的 tools。
