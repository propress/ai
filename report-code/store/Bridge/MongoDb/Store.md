# MongoDb/Store

MongoDB Atlas 向量搜索的存储实现，使用 `$vectorSearch` 聚合操作符。

## 关键方法/属性
- `__construct(Client, string $database, string $collection, string $indexName, string $vectorField, int $numCandidates)`
- `setup` — 创建向量搜索索引
- `add/remove/query/drop` — 标准操作

## 设计模式
- 适配器模式：封装 `mongodb/mongodb` 客户端

## 与其他文件的关系
- 实现 `ManagedStoreInterface`、`StoreInterface`
- 仅支持 `VectorQuery`
