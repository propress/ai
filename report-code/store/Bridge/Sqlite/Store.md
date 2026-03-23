# Sqlite/Store

SQLite + sqlite-vss/sqlite-vec 扩展的向量存储实现，适用于轻量级本地向量搜索场景。

## 关键方法/属性
- `__construct(PDO, string $table, int $dimensions)`
- `setup/drop` — 建表/删表
- `add/remove/query` — 使用 PDO 执行 SQL
- `supports()` — 支持 `VectorQuery` 及可选 `TextQuery`

## 设计模式
- 适配器模式：封装 PDO SQLite

## 与其他文件的关系
- 实现 `ManagedStoreInterface`、`StoreInterface`
