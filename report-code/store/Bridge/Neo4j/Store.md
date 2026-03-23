# Neo4j/Store

Neo4j 图数据库的向量存储实现，使用 Cypher 查询语言和向量索引进行相似度搜索。

## 关键方法/属性
- `__construct(Client, string $label, string $vectorProperty, int $dimensions)`
- `setup` — 创建向量索引
- `add/remove/query/drop` — Cypher 语句执行 CRUD

## 设计模式
- 适配器模式：封装 Neo4j PHP 客户端

## 与其他文件的关系
- 实现 `ManagedStoreInterface`、`StoreInterface`
- 仅支持 `VectorQuery`
