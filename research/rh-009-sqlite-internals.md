# RH-009: SQLite 原理

## 来源
- [SRC-03] OpenCode 使用 SQLite 作为持久化层
- SQLite 官方文档：https://sqlite.org/arch.html

## SQLite 是什么

一个**嵌入式的 C 库**（libsqlite3），不是服务器。

```
MySQL：   App → TCP → mysqld → 数据文件
SQLite：  App → 函数调用 → libsqlite3 → 一个 .db 文件
```

## 存储结构：B-tree 页

整个数据库是一个文件，由固定大小的**页**（默认 4KB）组成：

```
Page 0: Header ("SQLite format 3\000", page_size=4096, ...)
Page 1: B-tree Root（schema 表：session, message, part, ...）
Page 2-N: B-tree Interior Nodes & Leaf Pages（实际行数据）
Page N+1-...: Free Pages
```

每个表是一棵 B-tree。查找一条记录最多 2-3 次磁盘读取（O(log n)）。叶子节点存的是实际行数据。

## WAL（Write-Ahead Logging）

```typescript
// OpenCode 的启动配置
db.run("PRAGMA journal_mode = WAL")   // 启用 WAL
db.run("PRAGMA synchronous = NORMAL")
db.run("PRAGMA busy_timeout = 5000")
db.run("PRAGMA cache_size = -64000")
db.run("PRAGMA foreign_keys = ON")
```

WAL 原理：
```
Writer → 追加写入 WAL 文件（顺序写，极快）
Reader → 读主 .db 文件（不受 Writer 影响）
Checkpoint → 定期将 WAL 合并入主文件

结果：N 个并发读 + 1 个写，互不阻塞。
```

## 原子性保证

```
1. 开始事务
2. 把要修改的页的原内容复制到 Rollback Journal
3. 直接修改主数据库页
4. 成功 → 删 Journal。失败/断电 → 下次启动检测到 Journal → 恢复原页
```

## SQLite 的取舍

| 获得的 | 牺牲的 |
|--------|--------|
| 零运维（无守护进程） | 并发写：同一时刻只能一个写（文件锁） |
| 极低延迟（<1ms，函数调用） | 无网络访问（需自己封装网关） |
| 单文件便携（备份=cp） | 无用户权限系统（能打开文件就能读写） |
| 600KB 库文件 | 无水平扩展（不能跨 .db JOIN） |
| ACID 不缩水 | 查询单线程（一个查询只用一个核） |

## 适用场景

- 单进程应用的数据存储
- 边缘/客户端/移动端
- 数据量 <10GB（舒适区）
- 写并发 <每秒几百个事务
- 需要零运维的嵌入式场景
