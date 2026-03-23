# Pinecone/Store

Pinecone 托管向量数据库的存储适配器，通过 HTTP 调用 Pinecone REST API。

## 关键方法/属性
- `__construct(HttpClientInterface, string $host, string $apiKey, string $namespace)`
- `setup` — 验证索引连通性
- `add/remove/query/drop` — 标准操作

## 设计模式
- 适配器模式：封装 Pinecone REST API

## 与其他文件的关系
- 实现 `ManagedStoreInterface`、`StoreInterface`
- 仅支持 `VectorQuery`
