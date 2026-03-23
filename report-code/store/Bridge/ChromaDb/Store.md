# ChromaDb/Store

ChromaDB 向量数据库的存储适配器，支持创建/删除集合、添加/删除向量文档及向量和文本查询。

## 关键方法/属性
- `__construct(Client $client, string $collectionName)`
- `setup/drop` — 创建/删除 ChromaDB 集合
- `add/remove` — 批量操作文档
- `query()` — 支持 `VectorQuery` 和 `TextQuery`
- `transformResponse()` — 将 ChromaDB 响应转为 `VectorDocument`

## 设计模式
- 适配器模式：封装 `codewithkyrian/chromadb-php` 客户端
- 策略模式：分发向量和文本查询

## 与其他文件的关系
- 实现 `ManagedStoreInterface`、`StoreInterface`
- 依赖 `VectorDocument`、`Metadata`、`VectorQuery`、`TextQuery`
- 抛出 `RuntimeException`、`UnsupportedQueryTypeException`
