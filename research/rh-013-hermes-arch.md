# RH-013: Hermes Agent 架构 — 自学习闭环 + 可插拔引擎 + Cron 自动化

## 来源
- https://github.com/NousResearch/hermes-agent
- https://hermes-agent.nousresearch.com/docs/

## 核心架构

```
CLI / Gateway（网关守护进程，~16,000 行 Python）
│
├── run_agent.py（主代理循环，~788KB）
│   ├── MemoryManager ←→ MemoryProvider（内置 + 1 外部）
│   ├── ContextEngine（ContextCompressor，可替换）
│   ├── handle_function_call → tools/
│   └── Curator（后台线程，每 7 天运行）
│
├── GatewayRunner → 20+ 平台适配器
├── Cron Scheduler（每 60 秒 tick）
└── Environments（Atropos RL 训练）
```

Hermes 和 OpenClaw 都源自同一个代码祖先（openclaw/clawdbot）。
Hermes 在 OpenClaw 的基础上进行了大量重构和增强，特别是在学习闭环、上下文引擎和自动化方面。

## 六大核心创新

### 1. 自学习闭环（Curator）

**这是 Hermes 最独特的创新——目前没有其他开源 agent 框架有这个功能。**

Curator 是一个背景运行的辅助 agent，定期执行技能维护：
```
用户会话 → Agent 完成任务 → 创建 Skill（通过 skill_manager_tool）
→ 空闲时 curator 运行 →
  1. 自动状态转换（stale/archive/pinned，纯规则，无 LLM）
  2. Spawn 辅助 AIAgent → 评审技能库
  3. 合并（consolidate）、归档（archive）、修补（patch）
  4. 持久化 curator 状态 + 备份
```

```python
# curator.py — 核心参数
DEFAULT_INTERVAL_HOURS = 24 * 7     # 每 7 天
DEFAULT_MIN_IDLE_HOURS = 2          # 空闲 2 小时后才触发
DEFAULT_STALE_AFTER_DAYS = 30       # 30 天无用 = stale
DEFAULT_ARCHIVE_AFTER_DAYS = 90     # 90 天 = archive
```

关键设计原则：
- 只处理 agent-created skills（skill_usage.is_agent_created）
- 只用辅助模型（不污染主 session 的 prompt cache）
- 不删除——只归档（可恢复）
- 固定技能完全跳过自动转换

### 2. 三层记忆系统

| 层 | 实现 | 职责 |
|----|------|------|
| MemoryManager | `agent/memory_manager.py` | 统一入口，强制只有一个外部 provider |
| MemoryProvider | `agent/memory_provider.py`（ABC） | 抽象接口：initialize → prefetch → sync → tool_schemas |
| Honcho | 外部库 | 方言用户建模（跨 session 的用户理解） |
| FTS5 | SQLite FTS5 | 跨 session 全文搜索历史对话 |
| Skill | skill_manager_tool.py | 窄而可操作的「怎么做」过程知识 |

MemoryProvider 生命周期 hook：
```
initialize → system_prompt_block → on_turn_start → prefetch（背景召回）
→ sync_turn（异步写入）→ on_pre_compress → on_session_end
→ on_delegation（子代理观察）→ on_memory_write（镜像写入）
```

StreamingContextScrubber：在流式传输时从 LLM 输出中去除 `<memory-context>` 标签，防止记忆上下文泄漏到用户可见输出。

### 3. 可插拔上下文引擎

```python
class ContextEngine(ABC):
    def update_from_response(usage)          # 追踪 token 用量
    def should_compress(prompt_tokens)       # 判断是否压缩
    def compress(messages, tokens, focus)    # 执行压缩（可替换算法）
    def should_compress_preflight(msg)       # 预检（廉价估算）
```

默认 `ContextCompressor`（~1800 行）：
1. **免费修剪**：将旧工具输出替换为单行摘要（如 `[terminal] ran npm test => exit 0, 47 lines`），去重相同结果
2. **保护头尾**：protect_first_n=3, protect_last_n=6 + tail_token_budget
3. **LLM 总结**：结构化模板（已解决/待处理问题跟踪、「剩余工作」而非「后续步骤」）
4. **迭代更新**：后续压缩更新先前摘要，不重新生成

FTS5 跨会话搜索（`tools/session_search_tool.py`）：
- FTS5 在 SQLite 中找匹配消息 → 按会话分组 → 截断到 ~100K 字符 → 辅助模型总结

### 4. Cron 自动化系统

```python
# cron/scheduler.py — 1819 行完整实现
# 每 60 秒 tick() 检查到期作业 → 运行 AIAgent → 传递结果
hermes cron create "0 2 * * *" \
  "Pull the top bug, attempt a fix, open a PR" \
  --script ~/.hermes/scripts/fetch-bug.py \  # Python 预处理脚本
  --skills "github,linear" \                  # 技能链
  --deliver telegram                           # 多目标传递
```

完整特性：
- Python 脚本预处理：脚本 stdout → agent 上下文（分离机械工作和推理工作）
- `[SILENT]` 标记：空结果抑制传递
- 多目标传递：telegram/discord/slack/email/sms/webhook/local file
- 技能注入：运行时加载技能内容，在注入前扫描
- 单进程作业锁（`~/.hermes/cron/.tick.lock`）

### 5. Gateway（20+ 平台，16,046 行）

```
GatewayRunner
├── Telegram    ├── Discord    ├── WhatsApp   ├── Signal
├── Slack       ├── Matrix     ├── Mattermost ├── Email
├── SMS         ├── DingTalk   ├── Feishu     ├── WeChat
├── BlueBubbles ├── QQ Bot     ├── YuanBao    └── ...
```

核心特点：
- 全在同一进程中（agent 缓存 `_AGENT_CACHE_MAX_SIZE=128`，LRU+idle TTL 淘汰）
- 渐进编辑消息（Telegram/Discord/Slack 通用「先发初始消息后编辑」模式）
- PII 审查（平台 ID 哈希化）
- 推理标签剥离（`<think>` `<reasoning>` 不泄露到预览消息）

### 6. 子代理系统

`delegate_task` 工具：
- 创建具有隔离上下文、受限工具集和自身终端会话的子 `AIAgent` 实例
- 阻止递归委托、用户交互、内存写入、消息发送
- 两种角色：`leaf`（不能委托）和 `orchestrator`（可以委托）
- 审批自动拒绝子代理中的危险命令（`subagent_auto_approve: false`）
- `_active_subagents` 全局跟踪

## 与我们视角的关联

| Hermes 概念 | 我们的等价 |
|------------|----------|
| Curator（自学习） | 未来设计：Skill 自动积累 |
| MemoryManager | 我们的上下文持久化层 |
| ContextEngine（可插拔） | 我们的 Session 压缩策略 |
| 7 种执行后端 | 我们的 ExecutionEnv 不同实现 |
| Cron | 未来 Gateway 的 cron 调度 |
| Gateway 平台适配器 | 我们的 Gateway channel 路由 |
| Skill（过程记忆） | 我们的 Skill 系统 |
