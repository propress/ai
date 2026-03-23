# ClickHouse/Store

ClickHouse 列式数据库的向量存储实现，利用其内置向量函数进行相似度检索。

## 关键方法/属性
- `__construct(Client, string $database, string $table)`
- `setup/drop` — 建表/删表
- `add/remove` — 插入/删除文档
- `query(QueryInterface, array $options, ?float $minScore)` — 支持最低分数过滤

## 设计模式
- 适配器模式：封装 ClickHouse 客户端 SDK

## 与其他文件的关系
- 实现 `ManagedStoreInterface`、`StoreInterface`
- 仅支持 `VectorQuery`
