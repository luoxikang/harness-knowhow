# RH-001: Claude Agent SDK 架构

## 来源
- [SRC-01] https://github.com/anthropics/claude-agent-sdk-python (6.7k⭐)
- [SRC-02] https://github.com/anthropics/claude-agent-sdk-typescript (1.4k⭐)

## 通信机制

SDK 通过 spawn `claude` CLI 子进程，走 **stdin/stdout JSONL** 通信。

```python
# subprocess_cli.py — SubprocessCLITransport
class SubprocessCLITransport(Transport):
    """Subprocess transport using Claude Code CLI."""
```

- 启动 `claude` 二进制作为子进程
- stdin 写入 JSONL 指令
- stdout 读取 JSONL 事件
- 进程退出时自动 SIGTERM 清理

## 核心 API 入口

| API | 用途 |
|-----|------|
| `query(prompt, options)` | 一次性、无状态、单向 streaming |
| `ClaudeSDKClient` | 双向、交互式、多轮对话、支持 interrupt |

```python
# 简单查询
async for message in query(prompt="What is 2+2?"):
    if isinstance(message, AssistantMessage):
        for block in message.content:
            if isinstance(block, TextBlock):
                print(block.text)
```

## 高级特性

### Hook 系统（完整覆盖 agent 生命周期）
```
PreToolUse / PostToolUse / PostToolUseFailure
UserPromptSubmit / Stop / SubagentStop
PreCompact / Notification / SubagentStart
PermissionRequest
```

### MCP 支持
- 外部 MCP Server（子进程通信）
- SDK MCP Server（进程内，通过 `create_sdk_mcp_server()` + `@tool` 装饰器）

### SessionStore
- 抽象接口，内置 InMemory 实现
- 示例中有 Redis/S3/Postgres store
- 支持 session 的 import/export/fork/rename

### Transport
- 可注入自定义 Transport
- 默认 SubprocessCLITransport

## 关键限制

- **没有 HTTP Server** — 纯 SDK 库
- **依赖本地 claude CLI** — 必须安装 Claude Code
- **每 session 一个子进程** — 资源隔离好，但不能支持数百并发
- **无多租户** — 需自己封装
- **Claude 独占** — 不支持其他 LLM provider

## SDK 进程模型
```
用户代码
  → query() 或 ClaudeSDKClient
  → spawn claude 子进程
  → stdin/stdout JSONL
  → claude 子进程内部执行 LLM 循环 + 工具调用
  → 返回结果
```

SDK 本身不跑 agent loop。agent loop 在 `claude` CLI 进程内部。SDK 只是一个通信代理。
