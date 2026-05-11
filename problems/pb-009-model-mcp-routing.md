# PB-009: 多模型/多 MCP 如何路由？

## 问题

不同用户用不同的 LLM provider（Anthropic/OpenAI/Google），不同用户连不同的 MCP server（自己的 GitHub/Notion/Linear token）。怎么动态路由？

## 关键维度

- LLM token 的动态注入
- MCP server URL 的动态选择
- 模型选择（同一用户的 Agent 用哪个 model）
- Provider 切换（用户 A 用 Anthropic，用户 B 用 OpenAI）

## 参考方案

### 方案 A：静态配置（Claude Code / OpenCode 默认）

```
创建 Agent 时写好 model 和 mcp_servers
所有 session 用同一个配置
```

- 优点：简单
- 缺点：每个用户需要不同的 Agent 配置，管理复杂

### 方案 B：Gateway 动态注入

```
请求到达 Gateway
  → 验证用户 JWT → 查出 user 的 LLM token / MCP URL / model preference
  → 通过 HTTP header 注入（X-LLM-Token, X-MCP-URL, X-Model）
  → OpenCode Plugin 的 chat.headers hook 将 token 注入 LLM 请求
```

- 优点：Agent 配置通用，Gateway 做个性化
- 缺点：Gateway 成为单点

### 方案 C：Tool Factory（MCP 专用）

```
MCP Tool 不作为静态配置，而是由 ToolFactory 动态创建
创建 Session 时：
  1. 查用户 MCP 配置（URL + token）
  2. ToolFactory.createTools(mcpUrl, mcpToken) → 工具列表
  3. 合并到 Agent 的 tools 中
```

- 优点：每个 session 有独立的 MCP 工具列表
- 缺点：需要 ToolFactory 实现

### 方案 D：Pi 的 getApiKeyAndHeaders

```
Pi 的 AgentHarness 配置中有 getApiKeyAndHeaders 回调：
  getApiKeyAndHeaders: async (model) => {
    // 可以从外部上下文获取当前用户的 token
    return { apiKey: currentUser.llmToken, headers: {...} }
  }
```

- 优点：token 完全由外部控制，引擎无感
- 缺点：需要外部提供 currentUser 上下文

## 关键取舍

| | 静态配置 | Gateway header | Tool Factory | getApiKey |
|---|---|---|---|---|
| 灵活性 | 低 | 高 | 高 | 高 |
| Agent 感知 | 是 | 否（透明） | 否 | 否 |
| Gateway 依赖 | 无 | 有 | 有 | 有 |
| 适合场景 | 单用户 | 多租户 SaaS | MCP 路由 | LLM 路由 |

## 相关研究
- [RH-004] OpenCode 的 chat.headers / chat.params hook
- [RH-008] Pi 的 getApiKeyAndHeaders 机制
- [PB-005] 凭据安全管理
