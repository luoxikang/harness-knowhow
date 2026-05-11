# AP-003: MCP NAT 穿透方案

## 关联问题
- [PB-009] 多模型/多 MCP 如何路由？

## 问题本质

```
云端 OpenCode               用户的笔记本（NAT 后）
┌──────────────┐           ┌──────────────────────┐
│ 需要连 MCP   │────X────►│ 跑着 MCP Server       │
│ 但连不上     │           │ localhost:5100        │
└──────────────┘           └──────────────────────┘
```

## 四种方案

### 方案 A：ngrok

```bash
# 用户在笔记本上：
ngrok http 5100
→ https://alice-mcp.ngrok.io → localhost:5100
```

| 优点 | 缺点 |
|------|------|
| 一行命令，极简 | 免费版有速率限制 |
| 自动 HTTPS | 依赖第三方服务 |
| 不需自己维护基础设施 | URL 每次可能变化 |

### 方案 B：Cloudflare Tunnel

```bash
cloudflared tunnel --url http://localhost:5100
→ https://alice-mcp.trycloudflare.com → localhost:5100
```

| 优点 | 缺点 |
|------|------|
| 免费，无速率限制 | 需安装 cloudflared |
| Cloudflare 基础设施稳定 | URL 每次可能变化 |

### 方案 C：自建 WebSocket Relay

```
用户的 mcpc 进程 ──WS──► 你的 Relay Server ──TCP──► OpenCode MCP 配置
```

| 优点 | 缺点 |
|------|------|
| 完全可控 | 需要自己写 Relay Server |
| 用户只需装 mcpc（不用装第三方） | 运维成本 |
| 可做权限/限流/审计 | |

实现：见 [AP-002] Relay + mcpc 协议设计。

### 方案 D：Tailscale Funnel

```bash
tailscale funnel 5100
→ https://alice-machine.tailnet-name.ts.net:443 → localhost:5100
```

| 优点 | 缺点 |
|------|------|
| 一行命令 | 用户必须装 Tailscale |
| 稳定，不走第三方公网 | 依赖 Tailscale 平台 |
| 免费 | |

## 方案对比

| | ngrok | CF Tunnel | 自建 Relay | Tailscale |
|---|---|---|---|---|
| 用户负担 | 注册账号 | 装 cloudflared | 装 mcpc | 装 Tailscale |
| 你的负担 | 零 | 零 | 写 Relay | 零 |
| 稳定性 | 免费有限制 | 稳定 | 完全可控 | 稳定 |
| 可管控性 | 差 | 差 | 最好 | 差 |
| 成本 | 免费 | 免费 | 服务器 | 免费 |
| 适合规模 | 开发/小 | 中 | 大 | 中 |
