# Store 模块分析报告

## 模块概述

Store 模块是 Symfony AI 框架的向量数据库抽象层，提供文档的索引（存入）与检索（取出）能力。它将文档经过加载、过滤、转换、向量化后存入向量数据库，并支持通过语义相似度检索相关文档，是构建 RAG（检索增强生成）应用的核心组件。

---

## 架构总览

```
LoaderInterface
      ↓ 加载原始文档 (EmbeddableDocumentInterface)
FilterInterface
      ↓ 过滤不需要的文档
TransformerInterface
      ↓ 转换/切分文档 (TextDocument)
VectorizerInterface  ←→  Platform (嵌入模型)
      ↓ 生成向量 (VectorDocument)
StoreInterface  (向量数据库)
      ↓ 查询 (QueryInterface)
RetrieverInterface
      ↓ 可选重排 (RerankerInterface)
最终结果 (VectorDocument[])
```

### 核心流程说明

| 阶段 | 接口/类 | 说明 |
|------|---------|------|
| 加载 | `LoaderInterface` | 从文件、URL、RSS 等来源加载文档 |
| 过滤 | `FilterInterface` | 过滤不符合条件的文档 |
| 转换 | `TransformerInterface` | 切分、清洗、摘要生成等 |
| 向量化 | `VectorizerInterface` / `Vectorizer` | 调用 Platform 嵌入模型生成向量 |
| 存储 | `StoreInterface` | 写入向量数据库 |
| 检索 | `RetrieverInterface` / `Retriever` | 向量相似度查询 |
| 重排 | `RerankerInterface` / `Reranker` | 用交叉编码器模型重新排序结果 |

---

## 核心接口

### `StoreInterface`

向量数据库的统一抽象接口，所有 Bridge 实现都必须实现此接口。

- `add(VectorDocument|array $documents): void` — 添加向量文档
- `remove(string|array $ids, array $options = []): void` — 删除文档
- `query(QueryInterface $query, array $options = []): iterable` — 查询文档
- `supports(string $queryClass): bool` — 检查是否支持某种查询类型

### `IndexerInterface`

文档索引处理管道的入口接口。

- `index(string|iterable|object $input, array $options = []): void` — 处理输入并完成索引

### `RetrieverInterface`

从向量存储中按查询字符串检索文档的接口。

- `retrieve(string $query, array $options = []): iterable` — 检索相似文档

### `ManagedStoreInterface`

支持生命周期管理的存储接口（创建/销毁 Schema）。

- `setup(array $options = []): void` — 初始化存储基础设施
- `drop(array $options = []): void` — 销毁存储基础设施

---

## 查询类型

| 类 | 说明 |
|----|------|
| `VectorQuery` | 纯向量相似度搜索 |
| `TextQuery` | 全文关键词搜索 |
| `HybridQuery` | 混合搜索（向量 + 全文），支持语义比率调节 |

---

## 文档类型

| 类 | 说明 |
|----|------|
| `TextDocument` | 文本内容文档，实现 `EmbeddableDocumentInterface` |
| `VectorDocument` | 向量化后的文档，包含 Vector + Metadata + 可选 Score |
| `Metadata` | 扩展自 `ArrayObject`，预定义标准键（`_text`, `_source`, `_parent_id` 等） |

---

## Bridge 列表（支持的向量数据库）

| Bridge | 数据库 |
|--------|--------|
| `AzureSearch` | Azure AI Search |
| `Cache` | Symfony Cache（PSR-16） |
| `ChromaDb` | Chroma |
| `ClickHouse` | ClickHouse |
| `Cloudflare` | Cloudflare Vectorize |
| `Elasticsearch` | Elasticsearch |
| `ManticoreSearch` | Manticore Search |
| `MariaDb` | MariaDB |
| `Meilisearch` | Meilisearch |
| `Milvus` | Milvus |
| `MongoDb` | MongoDB Atlas Vector Search |
| `Neo4j` | Neo4j |
| `OpenSearch` | OpenSearch |
| `Pinecone` | Pinecone |
| `Postgres` | PostgreSQL + pgvector |
| `Qdrant` | Qdrant |
| `Redis` | Redis Stack |
| `S3Vectors` | AWS S3 Vectors |
| `Sqlite` | SQLite |
| `Supabase` | Supabase |
| `SurrealDb` | SurrealDB |
| `Typesense` | Typesense |
| `Vektor` | Vektor |
| `Weaviate` | Weaviate |

还有 `InMemory/Store` 用于测试和开发场景。

---

## Store 与 Platform 的关系

`Vectorizer` 类是 Store 与 Platform 之间的桥梁：

```
Vectorizer
  └── PlatformInterface::invoke($model, $content) → asVectors()
```

- `Vectorizer` 持有 `PlatformInterface` 实例与模型名称
- 调用嵌入模型将文本转换为 `Vector` 对象
- 支持批量向量化（检测模型是否支持 `Capability::INPUT_MULTIPLE`）
- 自动将原始文本保存到 `Metadata::KEY_TEXT` 供后续重排使用

`Reranker` 同样依赖 Platform：

```
Reranker
  └── PlatformInterface::invoke($model, ['query'=>..., 'texts'=>...]) → asReranking()
```

---

## 主要组件文件索引

| 文件 | 说明 |
|------|------|
| `StoreInterface.php` | 核心存储接口 |
| `IndexerInterface.php` | 索引接口 |
| `RetrieverInterface.php` | 检索接口 |
| `ManagedStoreInterface.php` | 生命周期管理接口 |
| `CombinedStore.php` | 组合存储（RRF 混合检索） |
| `Retriever.php` | 检索器实现，支持事件分发 |
| `Indexer/DocumentIndexer.php` | 直接文档索引器 |
| `Indexer/SourceIndexer.php` | 基于加载器的来源索引器 |
| `Indexer/DocumentProcessor.php` | 文档处理管道核心 |
| `Document/Vectorizer.php` | 向量化实现 |
| `InMemory/Store.php` | 内存存储（测试用） |
| `Distance/DistanceCalculator.php` | 距离计算（余弦、欧氏等） |
| `Reranker/Reranker.php` | 重排器实现 |
