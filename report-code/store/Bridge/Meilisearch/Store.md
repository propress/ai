# Meilisearch/Store

Meilisearch 搜索引擎的向量存储适配器，支持语义向量搜索能力。

## 关键方法/属性
- `__construct(Client, string $indexName, string $vectorField, int $dimensions)`
- `setup` — 配置索引并启用向量搜索
- `add/remove/query/drop` — 标准操作

## 设计模式
- 适配器模式：封装 Meilisearch PHP SDK

## 与其他文件的关系
- 实现 `ManagedStoreInterface`、`StoreInterface`
- 仅支持 `VectorQuery`
