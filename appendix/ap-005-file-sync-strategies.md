# AP-005: 跨 Workspace 文件同步策略

## 关联问题
- [PB-007] 文件系统的统一视图

## 核心原则

**默认不自动同步。** 文件存在各自的 ExecutionEnv 中（用户本地或云容器），不在云端。

跨 workspace 需要文件时，三种方式：

## 方式一：Git（主力，零开发）

```
Alice 在笔记本上改完代码：
  git commit → git push origin

Alice 在云容器里：
  Agent: "git pull 然后跑测试"
  Bash 在云容器执行 git pull → 拿到最新代码
```

适用：90% 的场景。本来就要用 git。

## 方式二：Agent 显式 cp（通过 ExecutionEnv）

```
Agent 执行：
  cp /ws/alice/laptop-projects/foo/main.py
     /ws/alice/cloud-container/bar/main.py
```

但注意：cp 命令在哪执行？如果 Bash 在笔记本电脑上执行：
  - 源：env.readTextFile → 笔记本本地读（快）
  - 目标：env.writeFile → 云容器本地写（通过 Relay RPC）

实际数据流：笔记本内存 → Relay → 云容器磁盘。是一次性网络传输，不是实时同步。

适用：偶尔需要同步个别文件。

## 方式三：Watch + 按需提醒（可选增强）

```
mcpc(笔记本) 监听文件变更
  → 只推送变更摘要（文件名 + diff 大小，不推内容）
  → 云端记录 "pending sync" 标记

Agent 在目标 workspace 执行时
  → Hook 检测到 pending sync
  → Agent 被告知："workspace X 下的文件 Y 有未同步的变更"
  → Agent 自己决定要不要 git pull 或 cp
```

这不是自动同步，是自动提醒。实际同步还是由 Agent 通过 Git 或 cp 触发。

## 为什么不自动同步

- 同步粒度不确定：改了 `package.json` 要不要同步 `node_modules`？
- 冲突处理：两边同时改了同一个文件怎么合并？
- 大文件：项目 50GB，每次改动都同步？

原则：Agent 负责决策，架构只提供能力（Git、cp），不做自动行为。
