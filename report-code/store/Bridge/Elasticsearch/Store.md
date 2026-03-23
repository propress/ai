# Elasticsearch/Store

Elasticsearch 的向量存储实现，使用 dense_vector 字段和 kNN 搜索能力。

## 关键方法/属性
- `__construct(Client, string $index, int $dimensions)`
- `setup` — 创建含 dense_vector 映射的索引
- `drop` — 删除索引
- `add/remove/query` — 标准 CRUD 及 kNN 查询

## 设计模式
- 适配器模式：封装 `elastic/elasticsearch-php` 客户端

## 与其他文件的关系
- 实现 `ManagedStoreInterface`、`StoreInterface`
- 仅支持 `VectorQuery`
