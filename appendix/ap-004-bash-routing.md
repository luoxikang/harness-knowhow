# AP-004: Bash 路径路由规则

## 关联问题
- [PB-006] Bash/工具在哪执行？

## 核心原则

**路径前缀 = 路由键。** 不需要额外路由表。

```
/ws/{user}/{workspace}/...
  │     │        │
  │     │        └── workspace 名 → mcpc 实例
  │     └── user         → 权限边界
  └── root              → FUSE 挂载点（可选）
```

## 路由表

```
Relay Server 维护：

routes["alice:laptop-projects"]     → ws(Alice 笔记本)
routes["alice:laptop-experiments"]  → ws(Alice 笔记本同一个连接)
routes["alice:cloud-container"]     → ws(Alice 云容器)
routes["bob:projects"]              → ws(Bob 笔记本)
```

## Bash 执行示例

```
Agent 执行：python /ws/alice/laptop-projects/foo/test.py

1. Plugin Hook 检测到 Bash tool
2. 解析路径 → user="alice", workspace="laptop-projects"
3. 查 relay routes["alice:laptop-projects"]
4. 发 RPC：bash.exec("python /home/alice/projects/foo/test.py", cwd="/home/alice/projects/foo")
5. mcpc(Alice笔记本) 执行 python
6. 结果返回

注意：路径映射
  /ws/alice/laptop-projects/foo/test.py
  → /home/alice/projects/foo/test.py (本地真实路径)
```

## 文件 I/O 示例

```
Agent 调 Read：/ws/alice/cloud-container/bar/main.py

1. ReadTool 调 this.env.readTextFile(path)
2. env 是 RemoteExecutionEnv(relay, "alice:cloud-container")
3. 发 RPC：file.read("/app/bar/main.py")
4. mcpc(Alice云容器) 读文件 → 返回内容
```

## 同一台机器的多个 workspace

```
Alice 笔记本上有两个 workspace：
  laptop-projects    → /home/alice/projects
  laptop-experiments → /home/alice/experiments

都走同一个 WebSocket 连接 → 同一个 mcpc 进程。
mcpc 内部根据 workspace 名映射到本地真实路径。

路由：
  routes["alice:laptop-projects"]     → ws
  routes["alice:laptop-experiments"]  → ws  (同一个)
```
