# PB-007: 文件系统的统一视图

## 问题

文件可能在不同机器上（用户的笔记本、云容器、NFS、S3）。Agent 应该看到怎样的文件树？

## 关键维度

- Agent 看到的文件树是否统一？
- Bash 工具能否直接操作（cat、ls、python test.py）？
- 文件操作是否有延迟？
- 是否支持流式读写大文件？

## 参考方案

### 方案 A：不做统一

```
Agent 只知道 cwd。
文件在哪 = 路径前缀决定。
Agent 不用关心——它正常调 Read/Write/Bash。
底层 ExecutionEnv 负责路由到正确的后端。
```

- 优点：最简单，不改 Agent
- 局限：跨 workspace 的 `cp` 需要两次 RPC（读 A → 写 B）
  但 Pi 的 ExecutionEnv 模式下这是透明的——Agent 看到的 `env.readTextFile` 和 `env.writeFile` 自动路由

**注意**：使用 Pi 的 ExecutionEnv 时，「文件在哪」这个问题被接口消解了。Agent 只调 `env.readTextFile(path)`，`env` 是 `RemoteExecutionEnv` 就会走 Relay RPC。没有「文件统一视图」问题——因为 Agent 根本不知道文件不在本地。

### 方案 B：FUSE 虚拟文件系统

```
所有文件挂到 /workspace 下
/workspace/alice/laptop-projects  → Relay → 用户笔记本
/workspace/alice/cloud-container  → 本地容器
/workspace/bob/projects           → Relay → Bob 笔记本
```

- 优点：Bash 透明（cat/ls/python 直接可用），文件 I/O 零代码改动
- 缺点：需要 FUSE 内核模块（Linux），延迟取决于网络，大文件不适合

### 方案 C：文件 RPC

```
Read/Write/Edit 工具通过 Relay RPC 代理
每次文件操作是一次 RPC 往返
不在本地执行 Bash（Bash 在用户机器上跑）
```

- 优点：跨平台（不需要 FUSE），天然支持远程
- 局限：不能直接用 cat/ls（只能通过工具）
  — 但 Bash 本来就在用户本地跑，所以 cat/ls 也在本地，不影响

## 关键取舍

| | 不统一 | FUSE | 文件 RPC |
|---|---|---|---|
| Agent 改动 | 零 | 零 | 零 |
| Bash 兼容 | 取决于 execute 位置 | 完美 | 取决于 execute 位置 |
| 延迟 | 取决于后端 | 网络往返 | 网络往返 |
| 平台要求 | 无 | Linux（FUSE） | 无 |

## 核心洞察

**使用 Pi 的 ExecutionEnv 后，「文件统一视图」问题自动消失**：
- Agent 看到 `env.readTextFile(path)` — 它不知道文件在哪
- `env` 是 `RemoteExecutionEnv(relay, user, workspace)` — 自动路由
- 不同 workspace 的文件通过不同的 ExecutionEnv 实例访问
- 没有「统一视图」问题，因为接口本身就是统一的

FUSE 方案只在需要「用标准 Linux 命令操作远程文件」时才有价值。
如果 Bash 已经在正确的机器上执行（通过 ExecutionEnv 路由），
那么 `cat /ws/alice/foo/main.py` 等价于 `cat /home/alice/foo/main.py`（在本地）。
不需要 FUSE。

## 相关研究
- [RH-007] Pi 的 ExecutionEnv 接口
- [PB-002] 沙盒抽象问题
- [PB-006] Bash 在哪执行
