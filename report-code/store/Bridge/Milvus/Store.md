# Milvus/Store

Milvus 向量数据库的存储适配器，支持集合管理、向量插入和 ANN 搜索。

## 关键方法/属性
- `__construct(HttpClientInterface, string $host, string $collectionName, int $dimensions)`
- `setup` — 创建集合并配置向量索引
- `add/remove/query/drop` — 标准向量操作

## 设计模式
- 适配器模式：通过 HTTP 封装 Milvus REST API

## 与其他文件的关系
- 实现 `ManagedStoreInterface`、`StoreInterface`
- 仅支持 `VectorQuery`
