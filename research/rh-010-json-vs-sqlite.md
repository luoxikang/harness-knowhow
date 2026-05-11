# RH-010: JSON vs SQLite 对比

## 来源
- Claude Code 使用 JSON 文件（`~/.claude/sessions/*.json`, `~/.claude/history.jsonl`）
- OpenCode 使用 SQLite（`~/.local/share/opencode/opencode.db`）
- Pi 使用 JSONL（每 session 一个 .jsonl 文件）

## Claude Code 的 JSON 存储

```
~/.claude/
├── sessions/33775.json     ← 每个 session 一个文件，~300 bytes
├── history.jsonl           ← 所有 prompt 历史，~3MB
└── settings.json           ← 配置
```

Session JSON 结构：
```json
{
  "sessionId": "ffce56d9-...",
  "cwd": "/Users/siliconluo/proj",
  "kind": "interactive",
  "status": "busy"
}
```

## 读模型对比

```
JSON:  O(n) 全量加载 → 遍历
  一条记录 1KB → 打开文件 + JSON.parse 1KB → 0.1ms
  100 万条记录 → 打开文件 + JSON.parse 1GB → 崩了

SQLite: O(log n) B-tree 定位
  一条记录 → 2-3 次 4KB 页读取 → 0.5ms
  100 万条记录 → 同样是 2-3 次页读取 → 0.5ms（索引深度不变）
```

## 写模型对比

```
JSON:
  改一行 → serialize(整个对象) → write(整个文件)
  1KB 文件改一行 = 写 1KB  → 能接受
  1GB 文件改一行 = 写 1GB  → 不可接受

SQLite:
  改一行 → 找到所在 4KB 页 → 修改该页 → 写入 WAL
  文件大小无关，只写 4KB
```

## 取舍矩阵

| | JSON | SQLite |
|---|---|---|
| 读 (查找一条) | O(n)，大文件慢 | O(log n)，恒定快 |
| 写 (改一条) | 覆写全文件 | 只写受影响的页 |
| 跨记录查询 | 需遍历所有文件 | SQL 搞定 |
| 人类可读 | `cat`/`vim`/`jq` | 二进制，需 sqlite3 客户端 |
| 版本控制 | `git diff` 看每次修改 | `.db` 二进制无意义 |
| 一个文件坏掉 | 只影响该文件 | 可能整个 DB 不可用 |
| 依赖 | 零（Node.js 原生） | 需 libsqlite3（600KB） |
| ACID | 无（可能半写） | 完整 |
| Schema 变更 | 自动兼容（加 key 就行） | 需写 migration SQL |

## 为什么 Claude Code 选 JSON 没问题

- 每个 session 独立一个文件，~300 bytes
- 不需要跨 session 查询
- `rm` 一个文件 = 删一个 session
- JSON 人类可读 → 调试友好

## 为什么 OpenCode 选 SQLite

- 所有 session 存在一起，需跨 session 查询
- 消息量大（一次对话 50+ 条消息）
- 需要 ACID（消息写入不能丢）
- 需要索引（按时间/session_id/type 查消息）

## Pi 的 JSONL 方案

- 每 session 一个 .jsonl 文件
- 每行一个 SessionTreeEntry（append-only）
- 跨 session 查询 = 遍历文件（O(n)）
- 可替换为 SQLiteSessionStorage（实现 SessionStorage 接口即可）
