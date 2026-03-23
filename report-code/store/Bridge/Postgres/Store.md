# Postgres/Store

PostgreSQL + pgvector 扩展的向量存储实现，支持多种距离度量和元数据 JSON 列。

## 关键方法/属性
- `__construct(Connection, string $table, int $dimensions, Distance $distance)`
- `setup` — 建表并安装 pgvector 扩展
- `add/remove/query/drop` — 使用 Doctrine DBAL 执行 SQL

## 设计模式
- 适配器模式：封装 Doctrine DBAL
- 策略模式：使用 `Distance` 枚举选择距离算子

## 与其他文件的关系
- 实现 `ManagedStoreInterface`、`StoreInterface`
- 依赖 `Bridge/Postgres/Distance` 枚举
