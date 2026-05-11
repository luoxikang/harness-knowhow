# RH-006: Pi 架构全览

## 来源
- [SRC-04] https://github.com/earendil-works/pi (47k⭐)

## 五层架构

```
Layer 5: Gateway（你要写的）          ← HTTP / 认证 / 计费 / 多租户
Layer 4: AgentHarness（编排层）        ← Session 持久化 / 压缩 / 队列 / Hook
Layer 3: Agent + AgentLoop（引擎层）   ← LLM 循环 / 工具执行 / 事件发射
Layer 2: Tool 实现（你定义的）         ← BashTool, ReadTool, MCPTool
Layer 1: ExecutionEnv（沙盒接口）      ← 文件 + Shell 的抽象 interface
```

## 关键边界

Pi 在设计上画了两道线：

### 边界 A：ExecutionEnv 接口

引擎与沙盒的分离。AgentLoop 不知道 Bash 在哪执行——它只调 `tool.execute()`。Tool 的实现调 `env.exec()`。`env` 是 `NodeExecutionEnv`（本地）还是 `RemoteExecutionEnv`（远程），对引擎完全透明。

### 边界 B：AgentHarness 的 Event Stream

引擎与客户端的分离。所有状态变更通过 `AgentEvent` + `AgentHarnessOwnEvent` 暴露。HTTP/SSE 在 Gateway 层接入，不在引擎内。

## 核心组件关系

```
AgentHarness
  └── Agent
        └── AgentLoop
              ├── streamAssistantResponse()  ← LLM 调用
              └── executeToolCalls()          ← 工具执行
                    └── tool.execute()        ← 通过 ExecutionEnv
```

- AgentHarness 拥有 Session、管理 Hook、持久化
- Agent 拥有状态（messages/tools/model）、管理队列（steer/followUp）
- AgentLoop 是纯函数，驱动一轮 LLM → Tool → LLM 循环

## 与其他框架的关键区别

| | Pi | OpenCode | Claude SDK |
|---|---|---|---|
| Agent 引擎有工具 | 无（由外部注入） | 有（Bash/Read/Write 内建） | 有（通过 CLI 二进制） |
| 沙盒抽象 | ExecutionEnv 接口 | 无 | 无（子进程即沙盒） |
| HTTP Server | 无（RPC 模式） | 内建 | 无 |
| Session 模型 | Tree + append-only | SQL + mutable | JSON 文件 |
