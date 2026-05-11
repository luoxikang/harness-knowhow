# AP-001: FUSE 统一沙盒方案

## 关联问题
- [PB-002] 沙盒/执行环境如何抽象？
- [PB-007] 文件系统的统一视图

## 方案描述

使用 FUSE（Filesystem in Userspace）将分布在多台机器上的文件系统挂载到一棵统一树下。

```
/workspace/
├── alice/
│   ├── laptop-projects/   → Relay → Alice 笔记本
│   └── cloud-container/   → 本地容器
└── bob/
    └── projects/           → Relay → Bob 笔记本
```

## 工作原理

```
Agent: cat /workspace/alice/laptop-projects/main.py

1. Linux VFS → FUSE Daemon（用户态进程）
2. FUSE Daemon 解析路径 → /workspace/alice/laptop-projects
3. 查路由表 → RemoteBackend(relay, "alice:laptop-projects")
4. 发 WebSocket RPC：{ op: "read", path: "/projects/main.py" }
5. Relay → mcpc(alice 笔记本) → fs.readFile("main.py")
6. 内容原路返回 → FUSE Daemon → VFS → bash stdout
```

## 实现要点

```go
type UnifiedFS struct {
    fuse.RawFileSystem
    routes map[string]Backend  // path prefix → backend
}

type Backend interface {
    Open(path string) ([]byte, error)
    ReadDir(path string) ([]fuse.DirEntry, error)
    Write(path string, data []byte) error
    Create(path string) error
    Remove(path string) error
}

// 本地后端（容器内文件）
type LocalBackend struct { root string }

// 远程后端（通过 Relay 到用户笔记本）
type RemoteBackend struct {
    userId string
    relay  *RelayClient
}
```

## 优点

- **Bash 完全透明**：`cat`、`ls`、`python test.py` 直接可用
- **Agent 零改动**：Agent 看到就是普通文件树
- **权限在路径层面**：`/ws/alice` 只属于 Alice

## 缺点

- **需要 FUSE 内核支持**（Linux 专有，macOS 用 macFUSE）
- **网络延迟影响每次文件 I/O**
- **大文件传输效率低**（每次读都走 RPC）
- **并发写入复杂度高**（多个 Agent 写同一文件）

## 重要洞察

**如果使用 Pi 的 ExecutionEnv 接口，FUSE 可能不需要。**

Pi 的 Agent 不直接操作文件——它调 `env.readTextFile()` 和 `env.exec()`。
`env` 是 `RemoteExecutionEnv` 时自动走 Relay RPC。
Bash 通过 `env.exec()` 也在正确的机器上执行（`cat /home/alice/foo/main.py` 在 Alice 的本地跑）。
不需要统一挂载点——接口本身就是统一的。
