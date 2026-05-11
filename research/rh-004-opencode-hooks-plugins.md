# RH-004: OpenCode Hook 与 Plugin 系统

## 来源
- [SRC-03] https://github.com/anomalyco/opencode (157k⭐)
- 关键源文件：`packages/plugin/src/index.ts`, `packages/opencode/src/plugin/index.ts`

## Hook 接口全景

```typescript
// packages/plugin/src/index.ts
export interface Hooks {
  event?: (input: { event: Event }) => Promise<void>
  config?: (input: Config) => Promise<void>
  tool?: { [key: string]: ToolDefinition }
  auth?: AuthHook
  provider?: ProviderHook

  // 消息生命周期
  "chat.message"?: (input, output) => Promise<void>     // 收到新消息
  "chat.params"?: (input, output) => Promise<void>      // 修改 LLM 参数
  "chat.headers"?: (input, output) => Promise<void>     // 修改 LLM 请求头

  // 工具执行
  "tool.execute.before"?: (input, output) => Promise<void> // 工具执行前
  "tool.execute.after"?: (input, output) => Promise<void>  // 工具执行后

  // 权限
  "permission.ask"?: (input, output) => Promise<void>   // 权限询问

  // 命令
  "command.execute.before"?: (input, output) => Promise<void>

  // Shell 环境
  "shell.env"?: (input, output) => Promise<void>        // 注入环境变量

  // 消息变换
  "experimental.chat.messages.transform"?: (input, output) => Promise<void>
  "experimental.chat.system.transform"?: (input, output) => Promise<void>

  // 压缩
  "experimental.session.compacting"?: (input, output) => Promise<void>
  "experimental.compaction.autocontinue"?: (input, output) => Promise<void>

  // 工具定义修改
  "tool.definition"?: (input, output) => Promise<void>
}
```

## 分发机制

```typescript
// plugin/index.ts
interface Interface {
  readonly trigger: <Name extends TriggerName>(
    name: Name,
    input: Input,
    output: Output,
  ) => Effect.Effect<Output>
}
```

- Plugin 通过 npm 安装
- 每个 Plugin 导出 `server(input: PluginInput) → Promise<Hooks>`
- 多个 Plugin 并行初始化
- hook 触发时遍历所有注册的 handler

## 独有 Hook 说明

| Hook | 作用 |
|------|------|
| `chat.params` | 每次 LLM 调用前修改 temperature/topP/maxTokens |
| `chat.headers` | 注入自定义 HTTP 头到 LLM 请求（如代理认证） |
| `shell.env` | 注入环境变量到 Bash 执行 |
| `tool.definition` | 动态修改工具描述和参数 schema |
| `chat.messages.transform` | 修改发给 LLM 的消息链（增删改） |

## 与 Claude Code Hook 的对比

| Claude Code | OpenCode | 说明 |
|-------------|----------|------|
| PreToolUse | tool.execute.before | 对等 |
| PostToolUse | tool.execute.after | 对等 |
| PostToolUseFailure | 无 | OpenCode 缺失 |
| UserPromptSubmit | chat.message | 对等 |
| Stop | 无专用 hook | 需用 chat.message + 消息类型判断 |
| SubagentStart | 无 | 缺失 |
| SubagentStop | 无 | 缺失 |
| PreCompact | session.compacting | 对等 |
| Notification | event | 对等 |
| PermissionRequest | permission.ask | 对等 |
