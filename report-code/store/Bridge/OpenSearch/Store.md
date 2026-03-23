# OpenSearch/Store

OpenSearch 的向量存储实现，使用 knn_vector 映射和 KNN 查询。

## 关键方法/属性
- `__construct(Client, string $index, int $dimensions)`
- `setup` — 创建含 knn_vector 映射的索引
- `add/remove/query/drop` — 标准操作

## 设计模式
- 适配器模式：封装 OpenSearch PHP 客户端

## 与其他文件的关系
- 实现 `ManagedStoreInterface`、`StoreInterface`
- 仅支持 `VectorQuery`
