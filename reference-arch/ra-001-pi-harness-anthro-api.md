# RA-001: 以 Pi 为 Harness 引擎的参考架构

## 概述

以 Pi 的 AgentHarness + Agent + ExecutionEnv 为核心引擎，实现 Anthropic Managed Agents API 契约，吸收 OpenCode 的成熟设计模式。

**这不是一个决定。这是一个参考设想，展示各组件如何组合。**

## 整体架构

```
┌──────────────────────────────────────────────────────────────┐
│                    Gateway（你写的）                           │
│                                                              │
│  职责：HTTP Server / JWT 认证 / 计费 / 多租户路由              │
│       Managed Agents API 契约                                │
│                                                              │
│  吸收的设计：                                                  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ SessionManager      ← OpenCode InstanceStore         │   │
│  │ WorkspaceRouter     ← OpenCode WorkspaceRouting      │   │
│  │ PermissionEngine    ← OpenCode Permission Ruleset   │   │
│  │ AgentVersionManager ← Managed Agents Agent Version  │   │
│  │ SSEController       ← OpenCode /event SSE           │   │
│  │ BillingHook         ← Managed Agents span event     │   │
│  │ MCPToolFactory      ← OpenCode MCP integration      │   │
│  │ SQLiteSessionStore  ← OpenCode SQLite schema        │   │
│  └──────────────────────────────────────────────────────┘   │
└────────────────────────────┬─────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────┐
│                 Pi AgentHarness（不改）                        │
│                                                              │
│  职责：消息队列 / Session 持久化 / 压缩 / Hook 分发            │
│        Skill + PromptTemplate 管理                           │
└────────────────────────────┬─────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────┐
│                 Pi Agent + AgentLoop（不改）                   │
│                                                              │
│  职责：LLM 循环 / 工具执行 / 事件发射                          │
│                                                              │
│  引擎不知道：HTTP / 用户 / 计费 / 文件在哪 / Bash 在哪         │
└────────────────────────────┬─────────────────────────────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
              ▼                             ▼
┌─────────────────────────┐   ┌─────────────────────────┐
│  Tool 实现（你定义的）     │   │  ExecutionEnv（你实现的）  │
│                         │   │                         │
│  BashTool  → env.exec   │   │  RemoteExecutionEnv     │
│  ReadTool  → env.read   │   │   → Relay RPC → mcpc   │
│  WriteTool → env.write  │   │                         │
│  MCPTool   → MCP client │   │  DockerExecutionEnv    │
│                         │   │   → 云容器              │
│                         │   │                         │
│                         │   │  NodeExecutionEnv      │
│                         │   │   → 本地 OS（内置）      │
└─────────────────────────┘   └─────────────────────────┘
```

## 关键设计要点

### 1. 不使用 FUSE

Pi 的 ExecutionEnv 接口让 FUSE 变得不必要：

- Agent 不直接调 `fs.readFile()`，它调 `this.env.readTextFile(path)`
- `env` 是 `RemoteExecutionEnv(relay, user, workspace)` → 自动 RPC 到用户机器
- Bash 通过 `env.exec()` 同样走 Relay RPC
- 文件 I/O 和 Bash 执行都在正确的机器上，不需要统一文件树

### 2. 多租户 = 多个 AgentHarness 实例

```
Gateway 维护：
  SessionManager.Map<"userId:workspaceId", {
    harness: AgentHarness,
    env: ExecutionEnv,
    session: Session
  }>
```

每个用户+workspace 组合得到一个独立的 AgentHarness + ExecutionEnv。
AgentHarness 之间通过 Gateway 路由隔离。

### 3. 凭据分两层

```
LLM API Key → Gateway 的 X-LLM-Token header → Pi 的 getApiKeyAndHeaders → LLM 请求头
MCP Token   → mcpc 本地进程保管 → Relay RPC 时注入 → 不进云端 Agent
Git Token   → mcpc 本地 git config → provision 时注入
```

### 4. 版本化 Agent

Gateway 层实现版本管理（不受 Pi 限制）：
- `agent_versions` 表：agent_id, version, config(JSON), created_at
- PUT /v1/agents/{id} → 生成新版本
- Session 创建时 resolve 到特定版本

### 5. Session 存储：SQLite 实现 Pi 的 SessionStorage 接口

```
Pi 的 Session 接口
  → 实现 SQLiteSessionStorage
  → session_events 表（session_id, seq, type, data, time_created）
  → 不可变的 append-only 表
  → 索引：session_idx, session_type_idx, time_created_idx
  → 继承 Pi 的 SessionTreeEntry 类型系统
```

## 与非 Pi 方案的对比

| 组件 | Pi 方案 | 无 Pi（从头写） | OpenCode 改造 |
|------|---------|----------------|--------------|
| Agent 引擎 | Pi Agent + AgentLoop | 自己写 LLM 循环 | OpenCode Runner |
| 沙盒接口 | ExecutionEnv（已有） | 自己设计接口 | Plugin Hook 模拟 |
| Session | Pi Session + SQLite | 自己设计 | SQLite（需改 schema） |
| Hook | AgentHarness event | 自己设计 | Plugin Hooks |
| HTTP | 自己写 Gateway | 自己写 | 已有 |
| MCP | 自己写 ToolFactory | 自己写 | 已有 |

## 相关
- [PB-001] ~ [PB-009] 底层问题
- [AP-007] 可吸收的 OpenCode 设计清单
- [AP-008] Managed Agents API 契约映射
