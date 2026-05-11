# AP-002: Relay + mcpc 协议设计

## 关联问题
- [PB-006] Bash/工具在哪执行？

## 架构

```
云端 OpenCode Plugin           Relay Server           mcpc（用户本地）
      │                           │                       │
      │  WebSocket                │  WebSocket            │
      │◄─────────────────────────►│◄─────────────────────►│
      │                           │                       │
      │  RPC: bash.exec           │  转发到对应用户的 ws    │  执行命令
      │  RPC: file.read           │                       │  读文件
      │  RPC: mcp.request         │                       │  转发 MCP
```

## Relay Server 实现要点

```javascript
// relay-server.js
const connections = new Map()  // userId → WebSocket
const portMap = new Map()      // userId → localPort

wss.on("connection", (ws, req) => {
  const token = parseToken(req.url)
  const { userId, workspaces } = authenticate(token)

  // 注册 WebSocket 连接
  for (const wsp of workspaces) {
    connections.set(`${userId}:${wsp}`, ws)
  }

  // 分配本地 TCP 端口，启动 TCP→WebSocket 转发器
  const localPort = allocatePort()
  portMap.set(`${userId}:${wsp}`, localPort)
  startTcpToWsProxy(localPort, ws)

  ws.on("close", () => {
    connections.delete(`${userId}:${wsp}`)
    releasePort(localPort)
  })
})
```

## mcpc（用户本地进程）实现要点

```javascript
// mcpc.js
const ws = new WebSocket(`ws://relay.cloud.com:9000?token=${TOKEN}`)

ws.on("message", (raw) => {
  const msg = JSON.parse(raw)

  switch (msg.type) {
    case "bash.exec":
      exec(msg.command, { cwd: msg.cwd, timeout: msg.timeout }, (err, stdout, stderr) => {
        ws.send(JSON.stringify({ id: msg.id, type: "bash.result", exitCode: err?.code || 0, stdout, stderr }))
      })
      break

    case "file.read":
      fs.readFile(msg.path, "utf8", (err, data) => {
        ws.send(JSON.stringify({ id: msg.id, type: "file.result", data, error: err?.message }))
      })
      break

    case "file.write":
      fs.writeFile(msg.path, msg.data, (err) => {
        ws.send(JSON.stringify({ id: msg.id, type: "write.result", error: err?.message }))
      })
      break

    case "file.list":
      fs.readdir(msg.path, { withFileTypes: true }, (err, entries) => {
        ws.send(JSON.stringify({ id: msg.id, type: "list.result", entries }))
      })
      break

    case "mcp.request":
      forwardToLocalMcp(msg)
      break
  }
})
```

## RPC 消息类型

| 类型 | 方向 | 说明 |
|------|------|------|
| `bash.exec` | 云→本地 | 执行命令 |
| `bash.result` | 本地→云 | 执行结果 |
| `file.read` | 云→本地 | 读文件 |
| `file.result` | 本地→云 | 文件内容 |
| `file.write` | 云→本地 | 写文件 |
| `write.result` | 本地→云 | 写入结果 |
| `file.list` | 云→本地 | 列目录 |
| `list.result` | 本地→云 | 目录条目 |
| `mcp.request` | 云→本地 | MCP 请求 |
| `mcp.result` | 本地→云 | MCP 响应 |

## 路径路由

```
Bash 执行路径：/ws/{user}/{workspace}/...
               → routes["{user}:{workspace}"] → mcpc ws
               → mcpc 在本地执行：cd /home/{user}/{workspace-映射}/... && {command}
```
