# RH-002: OpenCode 五层核心概念

## 来源
- [SRC-03] https://github.com/anomalyco/opencode (157k⭐)

## 五层架构

```
Layer 1: Process — 一个 Bun 进程 + Effect-TS Fiber Runtime
Layer 2: Instance — 按 directory 隔离的服务上下文
Layer 3: Session / Runner — 会话数据 + 执行状态机
Layer 4: Agent / Tool — 配置 + 可执行工具
Layer 5: Plugin / Hooks — 扩展与生命周期拦截
```

## Layer 1: Process（进程）

- 单 OS 进程（Bun）
- 内嵌 Effect-TS ManagedRuntime
- 并发通过 Fiber（类似 goroutine），不是 OS 线程
- 无子进程隔离

## Layer 2: Instance（实例）

```typescript
// instance-context.ts
interface InstanceContext {
    directory: string    // 根路径，如 /workspaces/alice
    worktree: string     // git worktree 路径
    project: ProjectInfo // 项目配置
}
```

- InstanceStore 用 `Map<directory, InstanceContext>` 管理
- 懒加载：第一次访问时创建
- 缓存：后续访问复用
- Instance 是**内存对象**，不是进程/容器/线程
- 提供该 directory 下的全部 Effect Service（Config/Session/File/MCP 等 30+）

## Layer 3: Session 与 Runner

### Session
- SQLite `session` 表一行
- 字段：id, parent_id, project_id, title, agent, model, permission
- 通过 parent_id 形成树（SubAgent 关系）

### Runner
- 每 Session 一个 Runner
- 状态机：`Idle | Running | Shell | ShellThenRun`
- 用 `SynchronizedRef` 实现并发安全
- 保证：同一 Session 同一时刻只有一个 active work
- 不同 Session 的 Runner 通过 Fiber 并发

## Layer 4: Agent 与 Tool

### Agent
- 配置对象：name, model, systemPrompt, permission, tools
- 不存状态（状态在 Session 中）
- mode: "primary" | "subagent" | "all"

### Tool
- 统一接口：`execute(args, ctx) → result`
- 内置工具：Bash, Read, Write, Edit, Grep, Glob, List, Task, WebSearch, WebFetch
- **注意**：工具实现直接调用 OS API（child_process, fs），没有沙盒抽象

## Layer 5: Plugin 与 Hooks

- 通过 npm 包分发
- Hooks 接口在 `packages/plugin/src/index.ts` 定义
- 15 个 hook 覆盖 agent 生命周期
- chat.headers / chat.params / tool.execute.before / tool.execute.after 等
