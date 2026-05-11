# PB-002: 沙盒/执行环境如何抽象？

## 问题

Bash、文件读写这些操作底层可能是本地进程、远程容器、用户笔记本。引擎应该看到什么样的接口？

## 关键维度

- 接口应该多薄？（execute(name,input)→string 还是完整文件系统接口？）
- 接口应该覆盖哪些操作？（Bash？Read？Write？Glob？网络？）
- 错误如何统一表示？

## 参考方案

### 方案 A：无抽象（直接调 OS）

```
Claude Code：子进程中直接 child_process.spawn / fs.readFile
OpenCode：BashTool 直接调 spawn，ReadTool 直接调 fs
```

- 优点：实现简单，性能最高
- 缺点：换沙盒实现 = 改所有工具代码

### 方案 B：最简接口

```
interface Sandbox {
  execute(name: string, input: string): Promise<string>
}
```

- 优点：极致简单，任何后端都能实现
- 缺点：文件操作也变成 execute("read", "{path}")——字符串序列化开销大
  流式输出难处理

### 方案 C：ExecutionEnv 接口（Pi 的方案）

```
interface ExecutionEnv {
  exec(command, opts) → {stdout, stderr, exitCode}
  readTextFile(path) → string
  readBinaryFile(path) → Uint8Array
  writeFile(path, content)
  listDir(path) → FileInfo[]
  fileInfo(path) → FileInfo
  exists(path) → boolean
  createDir / remove / realPath / cleanup
}
```

- 优点：12 个方法覆盖 agent 所有文件/进程需求，接口语义清晰
- 缺点：比最简接口多 11 个方法

### 方案 D：无沙盒（Process-per-Session）

```
每个 session 一个独立 OS 进程（或容器）。
Agent 的所有操作在该进程内执行。
进程边界 = 沙盒边界。
```

- 优点：安全边界 = OS 边界，最强隔离
- 缺点：每 session 一个进程/容器，资源开销大

## 关键取舍

| | 无抽象 | 最简接口 | ExecutionEnv | 进程隔离 |
|---|---|---|---|---|
| 实现复杂度 | 最低 | 低 | 中 | 高 |
| 可替换性 | 无 | 高 | 高 | 中 |
| 性能 | 最高 | 低（序列化） | 高 | 中 |
| 表达能力 | 无限制（但不可替换） | 弱 | 强 | 无限制 |
| 安全性 | 低 | 取决于实现 | 取决于实现 | 最高 |

## 相关研究
- [RH-007] Pi 的 ExecutionEnv 接口
- [RH-002] OpenCode 的 Tool 实现（直接调 OS）
