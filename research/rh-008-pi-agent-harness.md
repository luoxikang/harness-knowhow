# RH-008: Pi 的 AgentHarness 编排层

## 来源
- [SRC-04] https://github.com/earendil-works/pi (47k⭐)
- 关键源文件：`packages/agent/src/harness/agent-harness.ts`, `packages/agent/src/harness/types.ts`

## AgentHarness 职责

```
AgentHarness 是 Agent 的「外壳」，负责：
  1. Session 持久化（flushPendingSessionWrites → SessionStorage）
  2. 消息队列管理（steer / followUp / nextTurn）
  3. 压缩管理（compaction / branch_summary）
  4. Hook 分发（before/after 工具调用、上下文变换等）
  5. Skill + PromptTemplate 管理
  6. 模型切换 / Thinking Level 管理
```

## AgentHarness 与 Agent 的关系

```typescript
class AgentHarness<TSkill, TPromptTemplate, TTool> {
  readonly agent: Agent         // 核心引擎
  readonly env: ExecutionEnv    // 沙盒
  private session: Session       // 持久化
  private tools: Map<string, TTool>  // 工具注册表
}
```

创建时，AgentHarness 配置 Agent 的回调链路：

```
Agent.beforeToolCall ← AgentHarness 的 hook 转发
Agent.afterToolCall  ← AgentHarness 的 hook 转发
Agent.prepareNextTurn ← AgentHarness.flushPendingSessionWrites
Agent.onPayload       ← AgentHarness 的 before_provider_request hook
Agent.onResponse      ← AgentHarness 的 after_provider_response hook
Agent.transformContext ← AgentHarness 的 context hook
```

## Hook 系统（AgentHarnessEventResultMap）

15 个事件类型，分为四组：

### Agent 生命周期
- `before_agent_start` — prompt 发送前，可修改 messages 和 systemPrompt
- `context` — 每次 LLM 调用前，可修改完整消息链

### Provider 请求
- `before_provider_request` — 可修改 LLM 请求 payload
- `after_provider_response` — 获取 status + headers

### 工具执行
- `tool_call` — 工具调用前，可 block
- `tool_result` — 工具执行后，可修改 content/details/isError/terminate

### Session 持久化
- `session_before_compact` — 压缩前，可取消或注入 prompt
- `session_compact` — 压缩完成后通知
- `session_before_tree` — 分支摘要前，可取消或注入指令
- `session_tree` — 分支完成通知

### 其他
- `model_select` / `thinking_level_select` — 模型/思维等级变更通知
- `resources_update` — 资源变更通知
- `queue_update` / `abort` / `settled` — 队列状态变更通知

## 消息队列模型

```
steer 队列：
  用户消息 → Agent 正在跑 → 排入 steer 队列
  → 当前 assistant turn 结束后 → drain steer 队列 → 注入到 LLM 上下文

followUp 队列：
  Agent 停止后（无更多 tool call）→ drain followUp → 继续执行
```

## Session 持久化机制

```typescript
// 所有变更先缓存到 pendingSessionWrites
private pendingSessionWrites: PendingSessionWrite[] = []

// prepareNextTurn 时 flush
async flushPendingSessionWrites() {
    for (const write of writes) {
        if (write.type === "message") 
            await this.session.appendMessage(write.message)
        if (write.type === "compaction")
            await this.session.appendCompaction(...)
        // ...
    }
}
```

所有写入都是 append 操作。Session 是**不可变树**。
