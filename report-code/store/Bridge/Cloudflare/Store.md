# Cloudflare/Store

Cloudflare Vectorize 向量数据库的存储适配器，通过 HTTP API 进行向量索引与检索。

## 关键方法/属性
- `__construct(HttpClientInterface, string $accountId, string $apiKey, string $indexName)`
- `setup/drop` — 创建/删除 Vectorize 索引
- `add/remove/query` — 标准向量存储操作

## 设计模式
- 适配器模式：封装 Cloudflare REST API

## 与其他文件的关系
- 实现 `ManagedStoreInterface`、`StoreInterface`
- 仅支持 `VectorQuery`
