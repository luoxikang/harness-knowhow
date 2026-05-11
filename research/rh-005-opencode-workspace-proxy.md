# RH-005: OpenCode Workspace 代理机制

## 来源
- [SRC-03] https://github.com/anomalyco/opencode (157k⭐)
- 关键源文件：`src/server/shared/workspace-routing.ts`, `src/server/routes/instance/httpapi/middleware/proxy.ts`, `src/control-plane/types.ts`

## 核心概念：Target

```typescript
// control-plane/types.ts
type Target =
  | { type: "local"; directory: string }               // 本地目录
  | { type: "remote"; url: string | URL; headers?: HeadersInit }  // 远程实例
```

每个 Workspace 通过 `adapter.target(info)` 返回 Target。如果是 remote，所有请求透明代理。

## 路由决策流程

```typescript
// workspace-routing.ts
function planRequest(request, sessionWorkspaceID) {
    const url = new URL(request.url)
    const workspaceID = selectedWorkspaceID(url, sessionWorkspaceID)
    const workspace = resolveWorkspace(workspaceID)
    
    if (workspace → remote)
        return RequestPlan.Remote({ target.url, headers, ... })
    
    return RequestPlan.Local({ directory })
}
```

路由规则（`workspace-routing.ts`）：
- `GET /session` → 本地处理（聚合所有 session）
- `GET /session/{id}` → 转发到目标 workspace
- `POST /session/{id}/prompt` → 转发
- `GET /session/status` → 转发
- `/experimental/workspace` → 本地处理

## 代理实现

HTTP 代理（`proxy.ts`）：
```typescript
function http(client, url, extraHeaders, request) {
    // 构造目标请求（方法/头/body 完全透传）
    const response = client.execute(
        HttpClientRequest.make(method)(url, {
            headers: ProxyUtil.headers(request.headers, extraHeaders),
            body: requestBody(request)
        })
    )
    // 流式回传
    return HttpServerResponse.stream(response.stream, headers)
}
```

WebSocket 代理（`proxy.ts`）：
```typescript
function websocket(request, target) {
    const inbound = request.upgrade
    const outbound = Socket.makeWebSocket(target)
    // 双向管道：outbound.runRaw → writeInbound, inbound.runRaw → writeOutbound
}
```

## 架构含义

这本质上是**内置的反向代理 + 服务发现**：

```
客户端 → Server A (:4096)
           ├── Workspace 1: local → /home/alice
           ├── Workspace 2: local → /home/bob
           ├── Workspace 3: remote → http://Server-B:4097
           └── Workspace 4: remote → http://Server-C:4098
```

客户端只需知道 A 的地址。B 和 C 对客户端完全透明。
