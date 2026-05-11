# PB-008: Agent 之间如何协作？

## 问题

一个复杂任务需要多个 Agent 协作时，它们怎么通信？

## 关键维度

- 协作模式（树形委派？平等协作？中心协调？）
- 权限继承（子 Agent 能做什么？）
- 上下文传递（子 Agent 看到多少父 Agent 的上下文？）
- 取消信号级联（父被中断时子怎么办？）

## 参考方案

### 方案 A：SubAgent（OpenCode task tool）

```
父 Agent 调 task tool → 创建子 Session（parent_id）
子 Session 有自己的 Runner + Agent + 权限
父等待子完成 → 结果回传
```

- 树形结构（父→子→孙）
- 权限继承：父 session deny 规则 + 父 agent deny 规则 → 子
- 取消级联：父 abort → 子 abort
- 子看不到父的上下文（只看到 task description）

### 方案 B：MultiAgent Thread（Anthropic 文章）

```
协调者（Coordinator）分发任务给多个 Agent
每个 Agent 在自己的 Thread 中执行
协调者收集结果、发 follow-up、最终汇总
```

- 星形结构（协调者→多个 Agent）
- API 事件：thread_message_sent / thread_message_received / thread_status_*
- 每个 Agent 独立上下文，协调者管理上下文

### 方案 C：共享 Event Log

```
多个 Agent 读写同一个 Session 的 event log
每个 Agent 看到完整历史
Agent 间通过 event 通信
```

- 去中心化，最灵活
- 并发控制难（两个 Agent 同时写）
- 上下文膨胀（所有 Agent 的 event 都在同一个 log）

## 关键取舍

| | SubAgent | MultiAgent Thread | Shared Event Log |
|---|---|---|---|
| 拓扑 | 树 | 星 | 去中心化 |
| 权限控制 | 继承 | 各自配置 | 共享 |
| 上下文隔离 | 强 | 强 | 弱 |
| 取消级联 | 自动 | 需手动 | 需手动 |
| 复杂度 | 低 | 中 | 高 |

## 相关研究
- [RH-003] OpenCode 的 SubAgent 机制（task tool + subagent-permissions）
- [RH-011] Managed Agents API 的 MultiAgent 事件
