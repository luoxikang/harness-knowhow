# PB-004: 多租户隔离怎么做？

## 问题

多个用户的 Agent 共享同一套基础设施时，如何保证不互相影响？

## 关键维度

- 隔离强度（OS 进程？内存对象？文件系统？）
- 资源效率（一个用户消耗多少内存？）
- 数据隔离（用户 A 能不能读到用户 B 的 session？）
- 故障隔离（用户 A 的 Agent 崩溃是否影响用户 B？）

## 参考方案

### 方案 A：进程隔离（Claude SDK 模式）

```
每 session 一个 claude 子进程。
进程边界 = 安全边界。
```

- 优点：最强隔离，一个崩了不影响其他
- 缺点：100 个 session = 100 个 Node 进程 ≈ 10GB 内存

### 方案 B：Instance 隔离（OpenCode 模式）

```
Map<directory, InstanceContext>
按目录隔离服务上下文（Config/Session/MCP 等）
所有 Instance 在同一进程内。
```

- 优点：内存高效（一个进程）
- 缺点：同一进程内的 bug 影响所有用户

### 方案 C：ExecutionEnv 隔离（Pi 模式）

```
每个用户/workspace 独立的 ExecutionEnv 实例
AgentHarness 和 Agent 引擎共享（无用户状态）
```

- 优点：引擎无状态 + 沙盒实例化，可在同一进程中隔离
- 缺点：ExecutionEnv 实现需要自己保证隔离

### 方案 D：SQLite-per-user 数据隔离

```
/workspaces/alice/opencode.db
/workspaces/bob/opencode.db
```

- 优点：备份/迁移简单，一个用户一个文件
- 缺点：不能跨用户 JOIN

## 关键取舍

| | 进程隔离 | Instance | ExecutionEnv | SQLite-per-user |
|---|---|---|---|---|
| 安全隔离 | ★★★★★ | ★★ | ★★★ | ★★★★ |
| 资源效率 | ★ | ★★★★★ | ★★★★★ | ★★★★ |
| 实现复杂度 | 低 | 中 | 中 | 低 |
| 跨用户查询 | 不可能 | 同 DB | 取决于存储 | 不能 |

## 相关研究
- [RH-001] Claude SDK 的进程模型
- [RH-003] OpenCode 的 Instance 与 Runner
- [RH-007] Pi 的 ExecutionEnv
