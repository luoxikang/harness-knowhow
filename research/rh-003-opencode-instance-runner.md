# RH-003: OpenCode Instance 与 Runner 详解

## 来源
- [SRC-03] https://github.com/anomalyco/opencode (157k⭐)
- 关键源文件：`src/project/instance-store.ts`, `src/project/instance-context.ts`, `src/session/run-state.ts`, `src/effect/runner.ts`

## InstanceStore 实现

```typescript
// instance-store.ts
const cache = new Map<string, Entry>()

const load = (input: LoadInput): Effect.Effect<InstanceContext> => {
    const directory = AppFileSystem.resolve(input.directory)
    const existing = cache.get(directory)  // 缓存命中 → 复用
    if (existing) return Deferred.await(existing.deferred)
    
    // 缓存未命中 → 创建新 Entry
    const entry = { deferred: Deferred.makeUnsafe<InstanceContext>() }
    cache.set(directory, entry)
    
    // fork 到 Effect Scope 异步初始化
    Effect.forkIn(scope, completeLoad(directory, input, entry))
    return Deferred.await(entry.deferred)
}
```

关键设计点：
- `Deferred` 模式：多个并发请求同一 directory 时，只有第一个触发创建，其余等待同一个 Promise
- `Effect.forkIn(scope)`：初始化在独立 fiber 中运行
- `dispose` 机制：通过 `registerDisposer` 注册清理函数，`markInstanceForDisposal` 标记实例在请求结束后销毁

## Runner 状态机

```typescript
// runner.ts
type State<A, E> =
  | { _tag: "Idle" }
  | { _tag: "Running"; run: RunHandle }
  | { _tag: "Shell"; shell: ShellHandle }
  | { _tag: "ShellThenRun"; shell: ShellHandle; run: PendingHandle }
```

状态转换：

```
Idle ──ensureRunning(work)──► Running ──work完成──► Idle
Idle ──startShell(work)──► Shell ──shell结束──► Idle
Shell ──ensureRunning──► ShellThenRun ──shell结束──► Running ──► Idle
任意态 ──cancel()──► Idle
```

关键设计：
- `SynchronizedRef` 做并发安全的状态转换
- `ensureRunning` 在 Running 态时不启动新 work，而是等待当前 work 的结果（`awaitDone`）
- `startShell` 用于长时间运行的 shell（PTY），与 Running 互斥
- Runner 绑定在 Session 上，`SessionRunState` 管理 `Map<SessionID, Runner>`

## 并发模型总结

```
单进程内：
  InstanceStore.Map<directory, Instance>  ← 按目录懒加载
    └── SessionRunState.Map<SessionID, Runner>  ← 按 Session 管理
          └── Runner.State  ← 每 Session 最多一个 active work
```

- 跨 Instance：N 个 Instance 各自独立
- 跨 Session：N 个 Runner 各自独立，通过 Fiber 并发
- 单 Session：最多一个 active work（Runner 互斥）
