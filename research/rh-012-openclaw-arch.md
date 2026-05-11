# RH-012: OpenClaw 架构 — 控制面 + Channel + 进程隔离

## 来源
- https://github.com/openclaw/openclaw
- https://openclaw.ai · https://docs.openclaw.ai · [VISION.md](https://github.com/openclaw/openclaw/blob/main/VISION.md)

## 核心架构

```
openclaw gateway（控制面守护进程，单进程）
│
├── Agent A（ACP 会话）           ← 每个 agent 是独立 ACP 上下文
│   ├── session-alice-1
│   └── session-alice-2
│
├── Agent B（独立 ACP 会话）
│   └── session-bob-1
│
└── Channel Layer（消息路由层）
    ├── WhatsApp → routes to Agent A
    ├── Telegram → routes to Agent B
    └── Discord  → routes to Agent A (thread binding)
```

不是一个子进程一个 agent——agent 通过 ACP（Agent Communication Protocol）在网关进程内管理。
但 agent 可配置为 `acp` runtime（外部进程）或 `embedded` runtime（网关内联）。

## 三大核心创新

### 1. Gateway 控制面（统一管理多 Agent）

网关是一个持久的 Node.js 守护进程，拥有：
- `AcpSessionManager` 单例——管理所有 ACP 会话
- `SessionActorQueue`——按会话串行化操作
- `RuntimeCache`——缓存 agent 运行时

Agent 生命周期通过 `acp-spawn.ts` 管理：
```typescript
type SpawnAcpParams = {
  task: string          // 任务描述
  mode: "run" | "session"    // 一次性 or 持久会话
  sandboxMode: "inherit" | "require"  // 沙盒模式
  parentSession?: SessionEntry
  maxTurns?: number
  allowedTools?: string[]
}
```

**设计意图**：允许一个 Gateway 管理数百个 agent session，支持 spawn/fork/subagent。

### 2. Channel 概念

Channel 不是简单的「消息平台适配器」。它是 **用户交互表面 + Agent 绑定 + 会话解析**的三位一体抽象。

```typescript
type ChannelPlugin = {
  id: ChannelId; meta: ChannelMeta; capabilities: ChannelCapabilities;
  setupWizard, config, pairing, security, groups, mentions,
  outbound, status, gateway, auth, commands, lifecycle,
  streaming, threading, message, messaging, directory,
  resolver, actions, heartbeat, agentTools;
}
```

支持 80+ 平台/通道，每个 channel 可以：
- 绑定到特定 agent
- 有自己的 thread binding 策略（消息线程 → agent session）
- 独立的 idle timeout 和 max age
- 独立的 spawn policy

**Channel 的核心价值**：把「Agent 对外暴露为服务」产品化。用户通过 WhatsApp 发消息，Gateway 自动创建/复用 agent session，结果通过 WhatsApp 返回。用户不知道背后有几个 agent。

### 3. Agent 多实例与 Session 绑定

Agent 在配置中定义（`agents.list[]`），一个 Gateway 管理多个 agent：
```typescript
type AgentConfig = {
  id, default, name, workspace, agentDir,
  model, models, thinkingDefault, skills,
  sandbox, tools, runtime, subagents,
  contextLimits, identity, groupChat, ...
}
```

Session 通过 **线程绑定**（thread binding）路由到特定 agent：
- 入站消息 → Channel → 解析 Conversation Target → 查找或创建 Session
- Session 持久化在 SessionStore 中
- 支持 fork、resume、跨 channel 连续性

## 其他设计要点

### 沙盒系统
- `sandbox/workspace.ts` 控制工作空间在容器中的暴露
- `AgentSandboxConfig`: mode(off/non-main/all), workspaceAccess(none/ro/rw), scope(session/agent/shared)
- 支持 Docker 等后端

### 多模型 Provider
- 通过插件系统扩展（`extensions/openai/`, `anthropic/`, `google/`, `deepseek/`, `ollama/`）
- 提供者通过 SDK facade 注册，核心保持通用

### Plugin SDK
- `packages/plugin-sdk/` 导出 50+ 模块
- 覆盖：配置、通道、提供者、运行时、安全、测试
- 核心与插件边界严格分离

## 与我们视角的关联

| OpenClaw 概念 | 我们的等价 |
|-------------|----------|
| Gateway 控制面 | 我们的 Gateway（管理多 AgentHarness） |
| Channel 绑定 | 我们 Gateway 中的 channel 路由层 |
| ACP Session | 我们的 AgentHarness + Session |
| Agent Scope | 我们的 Agent 配置对象 |
| Sandbox | 我们的 ExecutionEnv（RemoteExecutionEnv/DockerExecutionEnv） |
| Plugin SDK | Pi 的 ExecutionEnv 接口 + SessionStorage 接口 |
