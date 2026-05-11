# RH-007: Pi 的 ExecutionEnv 接口

## 来源
- [SRC-04] https://github.com/earendil-works/pi (47k⭐)
- 关键源文件：`packages/agent/src/harness/types.ts`, `packages/agent/src/harness/env/nodejs.ts`

## 接口定义

```typescript
interface ExecutionEnv {
  cwd: string

  // Shell 执行
  exec(command: string, options?: {
    cwd?: string
    env?: Record<string, string>
    timeout?: number        // 秒
    signal?: AbortSignal
    onStdout?: (chunk: string) => void
    onStderr?: (chunk: string) => void
  }): Promise<{ stdout: string; stderr: string; exitCode: number }>

  // 文件操作
  readTextFile(path: string): Promise<string>
  readBinaryFile(path: string): Promise<Uint8Array>
  writeFile(path: string, content: string | Uint8Array): Promise<void>
  fileInfo(path: string): Promise<FileInfo>
  listDir(path: string): Promise<FileInfo[]>
  exists(path: string): Promise<boolean>
  createDir(path: string, options?: { recursive?: boolean }): Promise<void>
  remove(path: string, options?: { recursive?: boolean; force?: boolean }): Promise<void>
  realPath(path: string): Promise<string>
  createTempDir(prefix?: string): Promise<string>
  createTempFile(options?: { prefix?: string; suffix?: string }): Promise<string>

  // 生命周期
  cleanup(): Promise<void>
}
```

## 内置实现：NodeExecutionEnv

```typescript
// env/nodejs.ts
class NodeExecutionEnv implements ExecutionEnv {
  cwd: string

  async exec(command, options) {
    // child_process.spawn(shell, ['-c', command], { cwd, env })
    // stdout/stderr 通过 stream 事件收集
    // 支持 timeout + AbortSignal
  }

  async readTextFile(path) {
    // fs.readFile(resolve(this.cwd, path), 'utf8')
  }
  async writeFile(path, content) {
    // fs.mkdir + fs.writeFile
  }
  // ... 其他方法直接映射到 Node.js fs API
}
```

## 接口设计原则

1. **路径语义**：相对路径以 `cwd` 为基准，绝对路径直接使用
2. **错误语义**：统一抛出 `FileError`（code: not_found/permission_denied/not_directory/is_directory/invalid/unknown）
3. **路径不跟随 symlink**：除 `realPath()` 外，所有方法不解析 symlink
4. **`cleanup()` 释放资源**：对于本地实现是 no-op，对于远程实现可以关闭连接、清理临时文件

## 为什么这个接口是 Pi 最精华的设计

1. **12 个方法覆盖了 agent 需要的全部文件/进程操作**
2. **不包含任何特定实现细节**（没有 `child_process`、没有 `fs`）
3. **能实现为任何后端**：
   - `NodeExecutionEnv` — 本地 OS
   - `DockerExecutionEnv` — Docker 容器
   - `RemoteExecutionEnv` — WebSocket RPC 到用户机器
   - `FuseExecutionEnv` — FUSE 挂载点
   - `MockExecutionEnv` — 测试用
4. **引擎完全不用改**：AgentLoop 和 AgentHarness 只依赖接口，不依赖实现
