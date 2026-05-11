# AP-006: Pi vs OpenCode 详细对比

## 关联问题
- [PB-001] Harness 边界画在哪？

## 核心差异

| | Pi | OpenCode |
|---|---|---|
| **沙盒抽象** | ExecutionEnv 接口 — 12 个方法覆盖文件和进程 | 无 — Bash/Read/Write 直接调 OS |
| **引擎与沙盒关系** | 完全解耦，引擎只调接口 | 耦合，工具实现在引擎内 |
| **HTTP Server** | 无（RPC 模式） | 内建 `opencode serve` |
| **多租户路由** | 无 | Instance + Workspace 按目录隔离 + remote proxy |
| **MCP** | 不支持（设计哲学如此） | 原生支持 |
| **Session 模型** | Tree + append-only（不可变） | SQLite + mutable（可变） |
| **跨 session 查询** | 遍历 JSONL（O(n)） | SQL 查询（O(log n)） |
| **Hook 系统** | 15 个事件，类型安全的 hook map | 11 个 hook，npm 包分发 |
| **并发模型** | 单 Agent 互斥（prompt 不能并发） | 单 Session 互斥（Runner 状态机） |
| **代码复杂度** | 简洁（核心 ~500 行） | 高（Effect-TS 全栈） |
| **运行时** | Node.js / Bun | Bun + Effect-TS |
| **扩展方式** | TypeScript 函数 | npm Plugin |

## 各项评分

```
                     Pi     OpenCode
沙盒抽象接口          ★★★★★   ★☆☆☆☆
Hook 覆盖度           ★★★★★   ★★★★☆
Session 不可变模型     ★★★★★   ★★☆☆☆
HTTP Server 现成度    ★☆☆☆☆   ★★★★★
多租户路由现成度       ★☆☆☆☆   ★★★★★
MCP 现成度            ★☆☆☆☆   ★★★★★
SDK 现成度            ★★★☆☆   ★★★★★
代码简洁/可读性        ★★★★☆   ★★★☆☆
TypeScript 类型安全    ★★★★★   ★★★★☆
```

## 适合场景

### Pi 适合
- 需要引擎和沙盒边界清晰的长期产品
- 需要灵活替换沙盒实现（本地/远程/容器/Docker）
- 团队有能力写 HTTP 层和存储层
- 追求「改外部不改引擎」的架构

### OpenCode 适合
- 需要快速原型/开箱即用的 HTTP API
- 单实例/单用户部署
- 不需要自定义沙盒实现
- 需要原生 MCP 支持
