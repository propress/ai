# SearchStore

Azure AI Search 的向量存储实现，通过 HTTP 客户端与 Azure 认知搜索服务通信，支持向量索引和相似度搜索。

## 关键方法/属性
- `__construct(HttpClientInterface, string $endpointUrl, string $apiKey, string $indexName, string $apiVersion, string $vectorFieldName)`
- `add(VectorDocument|array $documents): void` — 批量索引文档
- `remove(string|array $ids, array $options): void` — 删除文档
- `query(QueryInterface $query, array $options): iterable` — 执行向量查询
- `supports(string $queryClass): bool` — 仅支持 `VectorQuery`

## 设计模式
- 适配器模式：将 `StoreInterface` 适配到 Azure Search REST API
- 策略模式：通过 `buildVectorQuery()` 构建 Azure 专有查询结构

## 与其他文件的关系
- 实现 `StoreInterface`
- 依赖 `VectorQuery`、`VectorDocument`、`Metadata`
- 抛出 `UnsupportedQueryTypeException`
