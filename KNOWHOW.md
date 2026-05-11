# KnowHow 总索引

## 底层问题（problems/）

| 编号 | 问题 | 说明 |
|------|------|------|
| [PB-001](problems/pb-001-harness-boundary.md) | Harness 的边界应该画在哪？ | 引擎与基础设施的分界线 |
| [PB-002](problems/pb-002-sandbox-abstraction.md) | 沙盒/执行环境如何抽象？ | Bash、文件读写在引擎中的接口 |
| [PB-003](problems/pb-003-session-model.md) | Session 的数据模型？ | 可变 vs 不可变，文件 vs 数据库 |
| [PB-004](problems/pb-004-multi-tenant.md) | 多租户隔离怎么做？ | 多用户共享基础设施时的隔离 |
| [PB-005](problems/pb-005-credential-isolation.md) | 凭据安全管理 | LLM Token、MCP Token 存放策略 |
| [PB-006](problems/pb-006-tool-execution.md) | Bash/工具在哪执行？ | 云端容器 vs 用户本地 vs 混合 |
| [PB-007](problems/pb-007-file-view.md) | 文件系统统一视图 | FUSE vs RPC vs 不统一 |
| [PB-008](problems/pb-008-inter-agent.md) | Agent 之间如何协作？ | SubAgent vs MultiAgent Thread |
| [PB-009](problems/pb-009-model-mcp-routing.md) | 多模型/多 MCP 如何路由？ | 静态配置 vs Gateway 动态注入 |

## 工具研究（research/）

| 编号 | 主题 | 来源 |
|------|------|------|
| [RH-001](research/rh-001-claude-agent-sdk.md) | Claude Agent SDK 架构 | SRC-01, SRC-02 |
| [RH-002](research/rh-002-opencode-arch.md) | OpenCode 五层核心概念 | SRC-03 |
| [RH-003](research/rh-003-opencode-instance-runner.md) | OpenCode Instance 与 Runner | SRC-03 |
| [RH-004](research/rh-004-opencode-hooks-plugins.md) | OpenCode Hook 与 Plugin 系统 | SRC-03 |
| [RH-005](research/rh-005-opencode-workspace-proxy.md) | OpenCode Workspace 代理 | SRC-03 |
| [RH-006](research/rh-006-pi-arch.md) | Pi 架构全览 | SRC-04 |
| [RH-007](research/rh-007-pi-executionenv.md) | Pi 的 ExecutionEnv 接口 | SRC-04 |
| [RH-008](research/rh-008-pi-agent-harness.md) | Pi 的 AgentHarness 编排层 | SRC-04 |
| [RH-009](research/rh-009-sqlite-internals.md) | SQLite 原理 | — |
| [RH-010](research/rh-010-json-vs-sqlite.md) | JSON vs SQLite 对比 | — |
| [RH-011](research/rh-011-managed-agents-api.md) | Managed Agents API 全貌 | DOC-01 ~ DOC-07 |
| [RH-012](research/rh-012-openclaw-arch.md) | OpenClaw 架构 | SRC-05 |
| [RH-013](research/rh-013-hermes-arch.md) | Hermes Agent 架构 | SRC-06 |

## 参考架构（reference-arch/）

| 编号 | 主题 | 说明 |
|------|------|------|
| [RA-001](reference-arch/ra-001-pi-harness-anthro-api.md) | 以 Pi 为 Harness 的参考架构 | 综合方案设想 |

## 技术附录（appendix/）

| 编号 | 主题 | 关联问题 |
|------|------|---------|
| [AP-001](appendix/ap-001-fuse-sandbox.md) | FUSE 统一沙盒方案 | PB-007 |
| [AP-002](appendix/ap-002-relay-protocol.md) | Relay + mcpc 协议设计 | PB-006 |
| [AP-003](appendix/ap-003-mcp-penetration.md) | MCP NAT 穿透方案 | PB-009 |
| [AP-004](appendix/ap-004-bash-routing.md) | Bash 路径路由规则 | PB-006 |
| [AP-005](appendix/ap-005-file-sync-strategies.md) | 跨 Workspace 文件同步 | PB-007 |
| [AP-006](appendix/ap-006-pi-vs-opencode.md) | Pi vs OpenCode 详细对比 | PB-001 |
| [AP-007](appendix/ap-007-absorb-opencode.md) | 可吸收的 OpenCode 设计清单 | RA-001 |
| [AP-008](appendix/ap-008-managed-agents-mapping.md) | Managed Agents API 契约映射 | RA-001 |
| [AP-009](appendix/ap-009-five-frameworks-compare.md) | 五框架综合对比 | PB-001 |
