# PB-003: Session 的数据模型？

## 问题

一次 Agent 对话应该用什么数据结构存储？可变还是不可变？文件还是数据库？

## 关键维度

- 是否需要跨 session 查询？
- 是否需要审计/回放能力？
- 是否需要人类直接查看原始数据？
- 数据量有多大？

## 参考方案

### 方案 A：JSON 文件 + 可变（Claude Code）

```
~/.claude/sessions/{pid}.json — 一个 session 一个 ~300B JSON
~/.claude/history.jsonl — 所有 prompt 的 JSONL
```

- 读：O(n)，每次读整个文件
- 写：覆盖写整个文件
- 优点：人类可读，零依赖，`rm` = 删
- 缺点：不能跨 session 查询，大文件性能差

### 方案 B：SQLite + 可变（OpenCode）

```
opencode.db → session / message / part 表
```

- 读：O(log n)，B-tree 索引
- 写：增量写 4KB 页
- 优点：SQL 查询能力，ACID 事务
- 缺点：二进制不可人类直接读，需 migration

### 方案 C：JSONL + 树形 append-only（Pi）

```
每 session 一个 .jsonl 文件
每行一个 SessionTreeEntry（message / compaction / model_change / ...）
只追加，不修改。
```

- 读：O(n)，顺序读取
- 写：追加一行 JSON
- 优点：不可变 = 审计友好，可回放，人类可读
- 缺点：不能跨 session 查询（需遍历文件），大文件慢

### 方案 D：Event Log + 不可变（Anthropic 文章）

```
Session = append-only event log
每个 event 有 type + data + processed_at
```

- 这是概念模型，不是存储方案
- 可用 JSONL、SQLite、Kafka 等实现
- 核心理念：所有状态变更都是 event，状态可回放重建

## 关键取舍

| | JSON | SQLite | JSONL Tree | Event Log |
|---|---|---|---|---|
| 读性能 | O(n) | O(log n) | O(n) | 取决于实现 |
| 跨 session 查询 | 遍历文件 | SQL ✓ | 遍历文件 | 取决于实现 |
| 不可变性 | ✗ | ✗（可变） | ✓ | ✓ |
| 人类可读 | ✓ | ✗ | ✓ | ✓（若文本格式） |
| Schema 变更 | 自动 | migration 流程 | 自动 | 自动 |
| ACID | ✗ | ✓ | ✗（文件追加） | 取决于实现 |

## 相关研究
- [RH-009] SQLite 原理
- [RH-010] JSON vs SQLite 对比
- [RH-011] Managed Agents API
- [RH-008] Pi 的 Session 树形结构
