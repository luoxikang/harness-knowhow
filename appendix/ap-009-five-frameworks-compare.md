# AP-009: 五个框架的综合对比 — Harness 架构全景

## 关联
- [PB-001] Harness 边界画在哪？
- [RA-001] 以 Pi 为 Harness 的参考架构

## 五框架核心架构对比

```
Claude SDK    OpenCode      Pi           OpenClaw      Hermes
──────────    ────────      ──           ────────      ──────
子进程隔离     单进程Fiber   单进程接口     单进程+ACP     单进程缓存
stdin/stdout  Effect-TS    ExecutionEnv  Channel路由    Curator闭环
JSON文件      SQLite       Tree JSONL    SessionStore   MemoryManager
Hook系统      Plugin Hook  Harness Hook  Channel绑定    ContextEngine
仅Claude      多模型       多模型        多Provider     多模型+无服务器
```

## 详细对比

### 1. Harness 边界

| | Claude SDK | OpenCode | Pi | OpenClaw | Hermes |
|---|---|---|---|---|---|
| 引擎模型 | claude 子进程 | Effect Fiber | Agent 类 | ACP 会话 | AIAgent 实例 |
| 沙盒抽象 | 子进程即沙盒 | 无 | ExecutionEnv 接口 | Sandbox workspace | 7 种后端 |
| HTTP Server | 无 | 内建 | 无（RPC） | 无 | FastAPI |
| 边界清晰度 | ★★★★ | ★★ | ★★★★★ | ★★★ | ★★★ |

### 2. Channel / 多租户

| | Claude SDK | OpenCode | Pi | OpenClaw | Hermes |
|---|---|---|---|---|---|
| Channel 概念 | 无 | 无 | 无 | ✅ 核心创新 | ✅ 继承自OpenClaw |
| 平台支持 | 无 | 无 | 无 | 80+ | 20+ |
| 多租户隔离 | 进程 | Instance(dir) | ExecutionEnv | Agent Scope | Agent Cache |
| Session 路由 | 手动 | InstanceStore | 手动 | Channel 绑定 | SessionSource |

### 3. 学习/记忆

| | Claude SDK | OpenCode | Pi | OpenClaw | Hermes |
|---|---|---|---|---|---|
| 自学习闭环 | 无 | 无 | 无 | 无 | ✅ Curator |
| 技能创建 | 无 | Agent Skills | Skill 接口 | Skills (ClawHub) | Skill Manager |
| 跨 Session 记忆 | SessionStore | SQLite query | SessionStorage | Memory Plugin | MemoryProvider |
| 用户建模 | 无 | 无 | 无 | 无 | Honcho 方言模型 |
| 上下文压缩 | 子进程内置 | compaction | AgentHarness | 无 | ContextEngine 可插拔 |

### 4. 自动化

| | Claude SDK | OpenCode | Pi | OpenClaw | Hermes |
|---|---|---|---|---|---|
| Cron 调度 | 无 | 无 | 无 | 无 | ✅ 完整 |
| 脚本预处理 | 无 | 无 | 无 | 无 | ✅ Python |
| 多目标传递 | 无 | 无 | 无 | Channel 绑定 | ✅ delivery |
| Webhook 触发 | 无 | 无 | 无 | 无 | ✅ |

## 创新点总结

### OpenClaw 的三个突破性创新

1. **Channel 概念** — 把「Agent 对外暴露为服务」标准化。
   用户交互表面（WhatsApp/Telegram/...）自动映射到 Agent Session。
   不是消息适配器，是产品抽象。

2. **Gateway 控制面** — 统一管理多个 Agent、
   Session 绑定、Channel 路由、沙盒策略。
   一个守护进程管所有。

3. **ACP spawn + 沙盒隔离** — Agent 可以是嵌入式或外部进程。
   沙盒配置（mode/workspaceAccess/scope）声明式管理。

### Hermes 的五个突破性创新

1. **Curator 自学习闭环** — Agent 空闲时自动整理、合并、归档技能。
   将成功经验固化为可复用知识。目前无其他开源框架做到。

2. **三层记忆系统** — MemoryProvider（声明式） + Skill（过程式） + Honcho（方言用户模型）。
   跨 session 的持久记忆。

3. **可插拔 ContextEngine** — 压缩算法可替换。默认 ContextCompressor 实现智能修剪 + LLM 总结 + 迭代更新。

4. **Cron 自动化** — Agent 驱动的定时任务。脚本预处理 → Agent 执行 → 多目标传递。

5. **多环境执行后端** — 本地/Docker/SSH/Singularity/Modal/Daytona/Vercel Sandbox。
   无服务器休眠（Modal/Daytona）实现零空闲成本。

## 两个框架的关系

Hermes 和 OpenClaw 源自同一代码祖先（openclaw/clawdbot）。
Hermes 在 OpenClaw 基础上进行了大量增强：

```
OpenClaw 提供了基础：
  ✅ Gateway 守护进程
  ✅ Channel 抽象
  ✅ ACP agent spawn

Hermes 在此基础上增加了：
  ✅ Curator 自学习闭环
  ✅ MemoryProvider 三层记忆
  ✅ ContextEngine 可插拔压缩
  ✅ Cron 自动化
  ✅ 多环境执行后端
  ✅ FTS5 跨会话搜索
  ✅ 更多平台（QQ/WeChat/飞书/钉钉等）
```

## 对我们参考架构的启示

### 可以直接吸收的设计

| 来自 | 吸收什么 | 放在哪一层 |
|------|---------|----------|
| OpenClaw | Channel 路由模式 | Gateway 的 channel 路由层 |
| OpenClaw | Session 绑定机制 | Gateway 的 SessionManager |
| Hermes | Curator 自学习闭环 | 未来：技能管理系统 |
| Hermes | MemoryProvider 接口 | Gateway 的记忆管理层 |
| Hermes | ContextEngine 可插拔 | Session 的压缩策略 |
| Hermes | Cron 自动化 | Gateway 的调度模块 |

### 验证了我们已有的判断

- **ExecutionEnv 接口是正确的**：Hermes 的 7 种执行后端恰好在 ExecutionEnv 的覆盖范围内
- **Pi 的边界最清晰**：五个框架中 Pi 的接口隔离做得最好
- **进程隔离 vs 接口隔离**：OpenClaw 证明了两者可以共存（ACP spawn + sandbox config）
- **Channel 是对外暴露的终极形态**：我们之前讨论的「Gateway + 多租户」本质上就是 Channel 的雏形
