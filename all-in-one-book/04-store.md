# 4 Store —— RAG

## 

 Store 25+ // RAG

---

## 1. 

 [ 3 Agent ](03-agent.md) Agent 

- ****Agent LLM → → → 
- **Toolbox ** `#[AsTool]` PHP AI 
- ****`InputProcessor` `OutputProcessor` 
- ** Agent ** `AgentTool` Agent 

Agent AI AI ****——** AI **

---

## 2. RAG 

### 2.1 Vector Embedding

 AI ****

```
"如何退款？"    → [0.12, -0.34, 0.56, ..., 0.78]  (1536 维)
"退款政策是什么？" → [0.11, -0.33, 0.55, ..., 0.77]  (非常接近！)
"今天天气如何？"  → [0.89, 0.23, -0.41, ..., 0.15]  (距离很远)
```

**Embedding Model** OpenAI `text-embedding-3-small` 1536 

### 2.2 RAG

RAGRetrieval-Augmented Generation AI **** LLM ****

```mermaid
flowchart LR
    A["📄 文档"] --> B["🔢 向量化"]
    B --> C["💾 存储"]
    C --> D["🔍 查询"]
    D --> E["📋 检索"]
    E --> F["🤖 生成"]

    style A fill:#e3f2fd
    style C fill:#fff3e0
    style F fill:#e8f5e9
```

 AI RAG 

| | LLM | RAG |
|---|---|---|
| **** | | |
| **** | | |
| **** | | |

### 2.3 Store 

Store Symfony AI **RAG **

```mermaid
flowchart TB
    subgraph "索引阶段（离线）"
        L["Loader<br>加载文档"] --> T["Transformer<br>分块/清洗"]
        T --> V["Vectorizer<br>生成向量"]
        V --> S["Store<br>持久化存储"]
    end

    subgraph "查询阶段（在线）"
        Q["用户提问"] --> V2["Vectorizer<br>向量化问题"]
        V2 --> R["Retriever<br>相似度检索"]
        R --> S
        R --> RR["Reranker<br>重排序"]
        RR --> A["Agent / LLM<br>生成回答"]
    end

    style L fill:#e3f2fd
    style S fill:#fff3e0
    style A fill:#e8f5e9
```

> Store ********

---

## 3. 

Store `TextDocument``VectorDocument``Metadata`

### 3.1 TextDocument

`TextDocument` ****

```php
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Metadata;

$document = new TextDocument(
    id: 'doc-001',                     // 文档唯一标识
    content: '退款政策：购买后 14 天内可申请全额退款。', // 文本内容
    metadata: new Metadata([
        '_title'  => '退款政策',
        '_source' => 'docs/refund.md',
        'category' => 'billing',
    ]),
);

echo $document->getId();      // 'doc-001'
echo $document->getContent(); // '退款政策：购买后 14 天内可申请全额退款。'

// TextDocument 是不可变的，withContent 返回新实例
$updated = $document->withContent('更新后的退款政策...');
```

### 3.2 VectorDocument

`VectorDocument` 

```php
use Symfony\AI\Store\Document\VectorDocument;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Platform\Vector\Vector;

$vectorDoc = new VectorDocument(
    id: 'doc-001',
    vector: new Vector([0.12, -0.34, 0.56, ...]), // 嵌入向量
    metadata: new Metadata(['_text' => '原始文本内容']),
    score: null,  // 存储时为 null，查询结果中有得分
);

// 查询后带得分的文档（不可变更新）
$scored = $vectorDoc->withScore(0.95);
echo $scored->getScore(); // 0.95
```

> `TextDocument`→ `Vectorizer` → `VectorDocument`→ `Store` → `score` `VectorDocument`

### 3.3 Metadata

`Metadata` `\ArrayObject`

```php
use Symfony\AI\Store\Document\Metadata;

$metadata = new Metadata();

// 内置键的便捷方法
$metadata->setText('原始文本内容');           // _text —— 向量化时自动填充
$metadata->setSource('/docs/refund.md');    // _source —— 文档来源路径
$metadata->setTitle('退款政策');             // _title —— 文档标题
$metadata->setSummary('14 天退款');          // _summary —— AI 生成的摘要
$metadata->setParentId('parent-doc-001');   // _parent_id —— 分块后关联原文档
$metadata->setDepth(2);                    // _depth —— 目录树深度

// 读取
if ($metadata->hasText()) {
    echo $metadata->getText();
}

// 自定义键
$metadata['department'] = 'HR';
$metadata['year'] = 2024;
```

| | | |
|--------|------|------|
| `_text` | `Metadata::KEY_TEXT` | `Vectorizer` |
| `_source` | `Metadata::KEY_SOURCE` | / URL |
| `_title` | `Metadata::KEY_TITLE` | |
| `_summary` | `Metadata::KEY_SUMMARY` | AI |
| `_parent_id` | `Metadata::KEY_PARENT_ID` | ID |
| `_depth` | `Metadata::KEY_DEPTH` | |

> `Metadata::KEY_TEXT` ——`Vectorizer` `Reranker` 

---

## 4. Loaders

 `TextDocument` 

### 4.1 LoaderInterface

```php
namespace Symfony\AI\Store\Document;

interface LoaderInterface
{
    /**
     * @return iterable<TextDocument>
     */
    public function load(?string $source = null, array $options = []): iterable;
}
```

### 4.2 

| | | | |
|--------|--------|------|----------|
| `TextFileLoader` | | 1 TextDocument / | |
| `MarkdownLoader` | Markdown | 1 TextDocument / | README |
| `CsvLoader` | CSV | 1 TextDocument | FAQ |
| `JsonFileLoader` | JSON | 1 TextDocument | API |
| `RssFeedLoader` | RSS/Atom URL | 1 TextDocument | |
| `RstLoader` | RST | 1 TextDocument / | Sphinx |
| `RstToctreeLoader` | RST | | Sphinx |
| `InMemoryLoader` | | | |

### 4.3 MarkdownLoader

```php
use Symfony\AI\Store\Document\Loader\MarkdownLoader;

$loader = new MarkdownLoader();

foreach ($loader->load('/docs/guide.md') as $document) {
    echo $document->getMetadata()->getTitle(); // 自动从 # 标题提取
    echo $document->getContent();
}
```

### 4.4 CsvLoader

```php
use Symfony\AI\Store\Document\Loader\CsvLoader;

$loader = new CsvLoader(
    contentColumn: 'answer',           // 内容列（列名或索引）
    idColumn: 'id',                    // ID 列（可选）
    metadataColumns: ['question', 'category'], // 元数据列
    delimiter: ',',
    hasHeader: true,
);

// faq.csv 格式：id,question,answer,category
foreach ($loader->load('/data/faq.csv') as $document) {
    echo $document->getContent();              // answer 列的内容
    echo $document->getMetadata()['category']; // 元数据
}
```

### 4.5 JsonFileLoader

```php
use Symfony\AI\Store\Document\Loader\JsonFileLoader;

$loader = new JsonFileLoader(
    id: '$.articles[*].id',
    content: '$.articles[*].body',
    metadata: [
        'title'  => '$.articles[*].title',
        'author' => '$.articles[*].author.name',
    ],
);

foreach ($loader->load('/data/articles.json') as $document) {
    echo $document->getMetadata()['title'];
}
```

### 4.6 RssFeedLoader

```php
use Symfony\AI\Store\Document\Loader\RssFeedLoader;

$loader = new RssFeedLoader(
    httpClient: $httpClient,
    uuidNamespace: '6ba7b810-9dad-11d1-80b4-00c04fd430c8',
);

foreach ($loader->load('https://example.com/feed.rss') as $document) {
    echo $document->getMetadata()['title'];
    echo $document->getMetadata()['link'];
}
```

### 4.7 InMemoryLoader



```php
use Symfony\AI\Store\Document\Loader\InMemoryLoader;
use Symfony\AI\Store\Document\TextDocument;

$documents = [
    new TextDocument('doc-1', '第一篇文档内容'),
    new TextDocument('doc-2', '第二篇文档内容'),
];

$loader = new InMemoryLoader($documents);

foreach ($loader->load() as $document) {
    echo $document->getContent();
}
```

### 4.8 

```php
use Symfony\AI\Store\Document\Loader\MarkdownLoader;

$loader = new MarkdownLoader();
$allDocuments = [];

// 遍历目录中的所有 Markdown 文件
foreach (glob('/docs/knowledge-base/*.md') as $file) {
    foreach ($loader->load($file) as $document) {
        $allDocuments[] = $document;
    }
}

echo count($allDocuments) . ' 篇文档已加载';
```

---

## 5. Transformers

————

### 5.1 TransformerInterface

```php
namespace Symfony\AI\Store\Document;

interface TransformerInterface
{
    /**
     * @param iterable<TextDocument> $documents
     * @return iterable<TextDocument>
     */
    public function transform(iterable $documents, array $options = []): iterable;
}
```

### 5.2 TextSplitTransformer

 RAG 

```php
use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;

$splitter = new TextSplitTransformer(
    chunkSize: 1000, // 每块最大字符数
    overlap: 200,    // 相邻块重叠字符数
);

$chunks = $splitter->transform($documents);

foreach ($chunks as $chunk) {
    echo "块内容：" . mb_substr($chunk->getContent(), 0, 50) . "...\n";
    echo "原文档 ID：" . $chunk->getMetadata()->getParentId() . "\n";
}
```

**overlap** 

```
原始文档（2600 字符）:
┌─────────────────────────────────────────────────────────┐
│ ...关于退款的详细说明...在14天内可以申请退款...退款流程...  │
└─────────────────────────────────────────────────────────┘

chunkSize=1000, overlap=200:
块 1: [0        ......        1000]
块 2:          [800   ......        1800]
块 3:                   [1600  ......     2600]
                ↑ 重叠区 ↑
```

 `$options` 

```php
$chunks = $splitter->transform($documents, [
    TextSplitTransformer::OPTION_CHUNK_SIZE => 500,
    TextSplitTransformer::OPTION_OVERLAP => 100,
]);
```

> ** RAG **

| chunkSize | overlap | | |
|-----------|---------|------|----------|
| 256 | 32 | | |
| 500 | 50 | | FAQWiki |
| 1000 | 200 | | |
| 2000 | 400 | | |

### 5.3 SummaryGeneratorTransformerAI 

 LLM `_summary` `yieldSummaryDocuments` 

```php
use Symfony\AI\Store\Document\Transformer\SummaryGeneratorTransformer;

$summarizer = new SummaryGeneratorTransformer(
    platform: $platform,
    model: 'gpt-4o-mini',
    yieldSummaryDocuments: true,  // 同时生成独立摘要文档
    systemPrompt: '用 2-3 句话总结以下文本的核心内容。',
);

foreach ($summarizer->transform($documents) as $doc) {
    echo $doc->getMetadata()->getSummary(); // AI 生成的摘要
}
```

> **Dual Indexing** `_parent_id` 

### 5.4 

```php
use Symfony\AI\Store\Document\Transformer\TextTrimTransformer;
use Symfony\AI\Store\Document\Transformer\TextReplaceTransformer;
use Symfony\AI\Store\Document\Transformer\ChunkDelayTransformer;

// TextTrimTransformer：去除首尾空白
$trimmer = new TextTrimTransformer();

// TextReplaceTransformer：文本替换
$replacer = new TextReplaceTransformer(
    search: '{{DRAFT}}',
    replace: '',
);

// ChunkDelayTransformer：在块之间添加延迟，避免 API 限速
$delayer = new ChunkDelayTransformer(delayMs: 1000);
```

### 5.5 ChainTransformer



```php
use Symfony\AI\Store\Document\Transformer\ChainTransformer;
use Symfony\AI\Store\Document\Transformer\TextTrimTransformer;
use Symfony\AI\Store\Document\Transformer\TextReplaceTransformer;
use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;

$transformer = new ChainTransformer(
    new TextTrimTransformer(),                          // 1. 清除空白
    new TextReplaceTransformer('<!-- TODO -->', ''),     // 2. 移除注释标记
    new TextSplitTransformer(chunkSize: 800, overlap: 150), // 3. 分块
);

$processedDocs = $transformer->transform($rawDocuments);
```

> 

---

## 6. Vectorization

### 6.1 VectorizerInterface Vectorizer

`Vectorizer` Platform 

```php
use Symfony\AI\Store\Document\Vectorizer;

$vectorizer = new Vectorizer(
    platform: $platform,              // PlatformInterface 实例
    model: 'text-embedding-3-small',  // 嵌入模型名称
    logger: $logger,                  // 可选的 PSR-3 日志记录器
);
```

`vectorize()` 

```php
// 1. 单个字符串 → Vector
$vector = $vectorizer->vectorize('如何申请退款？');
// 返回: Vector([0.12, -0.34, 0.56, ...])

// 2. 单个 TextDocument → VectorDocument
$vectorDoc = $vectorizer->vectorize($textDocument);
// 返回: VectorDocument（自动将原文保存到 metadata._text）

// 3. 字符串数组 → Vector 数组（批量）
$vectors = $vectorizer->vectorize(['文本一', '文本二', '文本三']);

// 4. TextDocument 数组 → VectorDocument 数组（批量）
$vectorDocs = $vectorizer->vectorize([$doc1, $doc2, $doc3]);
```

> `Vectorizer` `Capability::INPUT_MULTIPLE` OpenAI text-embedding API 

### 6.2 

| | | | | |
|------|--------|------|-----------|----------|
| `text-embedding-3-small` | OpenAI | 1536 | ~6KB | RAG |
| `text-embedding-3-large` | OpenAI | 3072 | ~12KB | |
| `text-embedding-ada-002` | OpenAI | 1536 | ~6KB | |
| `nomic-embed-text` | Ollama | 768 | ~3KB | |
| `mxbai-embed-large` | Ollama | 1024 | ~4KB | |

### 6.3 

```php
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\Component\HttpClient\HttpClient;

// 创建 Platform 和 Vectorizer
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());
$vectorizer = new Vectorizer($platform, 'text-embedding-3-small');

// 向量化单个文档
$textDoc = new TextDocument('doc-001', '退款政策：14 天内可全额退款。');
$vectorDoc = $vectorizer->vectorize($textDoc);

echo '向量维度：' . count($vectorDoc->getVector()->getData()) . "\n"; // 1536
echo '原始文本：' . $vectorDoc->getMetadata()->getText() . "\n";     // 自动保存
```

---

## 7. Indexer



### 7.1 DocumentProcessor

`DocumentProcessor` ** → → → ** 

```php
use Symfony\AI\Store\Indexer\DocumentProcessor;
use Symfony\AI\Store\Document\Transformer\TextTrimTransformer;
use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;
use Symfony\AI\Store\Document\Filter\TextContainsFilter;

$processor = new DocumentProcessor(
    vectorizer: $vectorizer,
    store: $store,
    filters: [
        new TextContainsFilter('关键词'), // 只保留包含关键词的文档
    ],
    transformers: [
        new TextTrimTransformer(),
        new TextSplitTransformer(chunkSize: 1000, overlap: 200),
    ],
    logger: $logger,
);

// 直接处理文档集合
$processor->process($documents, [
    'chunk_size' => 50,                        // 每批向量化的文档数
    'platform_options' => ['timeout' => 30],   // 传递给 Platform 的选项
]);
```

```mermaid
flowchart LR
    A["TextDocument<br>集合"] --> B["FilterInterface<br>过滤"]
    B --> C["TransformerInterface<br>转换/分块"]
    C --> D["VectorizerInterface<br>向量化"]
    D --> E["StoreInterface<br>存储"]

    style A fill:#e3f2fd
    style E fill:#e8f5e9
```

### 7.2 SourceIndexer

`SourceIndexer` URL Loader `DocumentProcessor` 

```php
use Symfony\AI\Store\Indexer\SourceIndexer;
use Symfony\AI\Store\Document\Loader\MarkdownLoader;

$indexer = new SourceIndexer(
    loader: new MarkdownLoader(),
    processor: $processor,
);

// 索引单个文件
$indexer->index('/docs/guide.md');

// 索引多个文件
$indexer->index([
    '/docs/chapter1.md',
    '/docs/chapter2.md',
    '/docs/chapter3.md',
]);
```

### 7.3 DocumentIndexer

 `TextDocument` Loader 

```php
use Symfony\AI\Store\Indexer\DocumentIndexer;

$indexer = new DocumentIndexer($processor);

// 索引单个文档
$indexer->index($textDocument);

// 批量索引
$indexer->index([$doc1, $doc2, $doc3]);
```

### 7.4 ConfiguredSourceIndexer

 Symfony 

```php
use Symfony\AI\Store\Indexer\ConfiguredSourceIndexer;

$indexer = new ConfiguredSourceIndexer(
    indexer: $sourceIndexer,
    defaultSource: '/var/knowledge-base/', // 默认来源
);

// 使用默认来源
$indexer->index();

// 运行时覆盖
$indexer->index('/var/other-docs/');
```

### 7.5 

```php
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Store\Bridge\Postgres\Store;
use Symfony\AI\Store\Document\Loader\MarkdownLoader;
use Symfony\AI\Store\Document\Transformer\TextTrimTransformer;
use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;
use Symfony\AI\Store\Document\Transformer\ChainTransformer;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentProcessor;
use Symfony\AI\Store\Indexer\SourceIndexer;
use Symfony\Component\HttpClient\HttpClient;

// 1. 创建平台和向量化器
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());
$vectorizer = new Vectorizer($platform, 'text-embedding-3-small');

// 2. 创建 PostgreSQL 存储
$pdo = new \PDO('pgsql:host=localhost;dbname=knowledge_base', 'postgres', 'secret');
$store = Store::fromPdo($pdo, 'document_embeddings');
$store->setup(); // 自动创建表和 pgvector 索引

// 3. 构建完整管线
$processor = new DocumentProcessor(
    vectorizer: $vectorizer,
    store: $store,
    transformers: [
        new TextTrimTransformer(),
        new TextSplitTransformer(chunkSize: 1000, overlap: 200),
    ],
);

$indexer = new SourceIndexer(
    loader: new MarkdownLoader(),
    processor: $processor,
);

// 4. 索引整个文档目录
foreach (glob('/docs/knowledge-base/*.md') as $file) {
    $indexer->index($file);
    echo "✅ 已索引：{$file}\n";
}

echo "索引完成！\n";
```

---

## 8. 

### 8.1 StoreInterface

```php
namespace Symfony\AI\Store;

interface StoreInterface
{
    /**
     * @param VectorDocument|VectorDocument[] $documents
     */
    public function add(VectorDocument|array $documents): void;

    /**
     * @param string|array<string> $ids
     */
    public function remove(string|array $ids, array $options = []): void;

    /**
     * @return iterable<VectorDocument>
     * @throws UnsupportedQueryTypeException
     */
    public function query(QueryInterface $query, array $options = []): iterable;

    /**
     * @param class-string<QueryInterface> $queryClass
     */
    public function supports(string $queryClass): bool;
}
```

### 8.2 ManagedStoreInterface

 `ManagedStoreInterface`

```php
interface ManagedStoreInterface
{
    public function setup(array $options = []): void; // 创建表/集合/索引
    public function drop(array $options = []): void;  // 删除所有数据
}
```

 CLI 

```bash
# 初始化存储（创建表/索引）
php bin/console ai:store:setup

# 清空存储（危险操作！）
php bin/console ai:store:drop
```

### 8.3 25+ 

Store 

#### 

| | | |
|------|------|------|
| **Pinecone** | `symfony/ai-pinecone-store` | |
| **Qdrant Cloud** | `symfony/ai-qdrant-store` | |
| **Weaviate** | `symfony/ai-weaviate-store` | GraphQL |
| **Milvus** | `symfony/ai-milvus-store` | GPU |
| **Supabase** | `symfony/ai-supabase-store` | PostgreSQL + pgvector REST API |
| **Azure AI Search** | `symfony/ai-azure-search-store` | Azure |
| **S3Vectors** | `symfony/ai-s3-vectors-store` | AWS S3 |
| **Cloudflare** | `symfony/ai-cloudflare-store` | |

#### 

| | | |
|------|------|------|
| **PostgreSQL** | `symfony/ai-postgres-store` | pgvector ACID |
| **Redis** | `symfony/ai-redis-store` | RedisSearch |
| **Elasticsearch** | `symfony/ai-elasticsearch-store` | kNN |
| **OpenSearch** | `symfony/ai-opensearch-store` | Elasticsearch |
| **ChromaDB** | `symfony/ai-chroma-db-store` | Python |
| **MongoDB** | `symfony/ai-mongodb-store` | Atlas Vector Search |
| **Neo4j** | `symfony/ai-neo4j-store` | + |
| **MariaDB** | `symfony/ai-mariadb-store` | |
| **SQLite** | `symfony/ai-sqlite-store` | sqlite-vec |
| **ClickHouse** | `symfony/ai-clickhouse-store` | |
| **SurrealDB** | `symfony/ai-surrealdb-store` | |

#### 

| | | |
|------|------|------|
| **Meilisearch** | `symfony/ai-meilisearch-store` | + |
| **Typesense** | `symfony/ai-typesense-store` | |
| **ManticoreSearch** | `symfony/ai-manticore-search-store` | |

#### 

| | | |
|------|------|------|
| **InMemory** | | |
| **Cache** | `symfony/ai-cache-store` | PSR-6 TTL |
| **CombinedStore** | | RRF |
| **Vektor** | `symfony/ai-vektor-store` | |

### 8.4 

| | VectorQuery | TextQuery | HybridQuery | ManagedStore |
|----------|:-----------:|:---------:|:-----------:|:------------:|
| **InMemory** | ✅ | ✅ | ✅ | ✅ |
| **PostgreSQL** | ✅ | ✅ | ✅ | ✅ |
| **SQLite** | ✅ | ✅ | ✅ | ✅ |
| **Cache** | ✅ | ✅ | ✅ | ✅ |
| **ChromaDB** | ✅ | ✅ | ❌ | ✅ |
| **Meilisearch** | ✅ | ✅ | ❌ | ✅ |
| **Elasticsearch** | ✅ | ✅ | ✅ | ✅ |
| **OpenSearch** | ✅ | ✅ | ✅ | ✅ |
| **AzureSearch** | ✅ | ✅ | ❌ | ✅ |
| **Typesense** | ✅ | ✅ | ❌ | ✅ |
| **ManticoreSearch** | ✅ | ✅ | ❌ | ✅ |
| **Qdrant** | ✅ | ❌ | ❌ | ✅ |
| **Redis** | ✅ | ❌ | ❌ | ✅ |
| **Pinecone** | ✅ | ❌ | ❌ | ✅ |
| **MongoDB** | ✅ | ❌ | ❌ | ✅ |
| **Milvus** | ✅ | ❌ | ❌ | ✅ |
| **Weaviate** | ✅ | ❌ | ❌ | ✅ |
| **Neo4j** | ✅ | ❌ | ❌ | ✅ |
| **Cloudflare** | ✅ | ❌ | ❌ | ✅ |
| **S3Vectors** | ✅ | ❌ | ❌ | ✅ |
| **MariaDB** | ✅ | ❌ | ❌ | ✅ |
| **SurrealDB** | ✅ | ❌ | ❌ | ✅ |
| **ClickHouse** | ✅ | ❌ | ❌ | ✅ |
| **Supabase** | ✅ | ❌ | ❌ | ❌ |
| **CombinedStore** | ✅ | ✅ | ✅RRF | ❌ |

> `HybridQuery` `CombinedStore` 

### 8.5 PostgreSQLpgvector

PostgreSQL 

```bash
# 安装
composer require symfony/ai-postgres-store

# Docker 启动 PostgreSQL + pgvector
docker run -d --name pgvector \
    -e POSTGRES_PASSWORD=secret \
    -e POSTGRES_DB=knowledge_base \
    -p 5432:5432 \
    pgvector/pgvector:pg17
```

```php
use Symfony\AI\Store\Bridge\Postgres\Store;
use Symfony\AI\Store\Bridge\Postgres\Distance;

$pdo = new \PDO('pgsql:host=localhost;dbname=knowledge_base', 'postgres', 'secret');

$store = new Store(
    connection: $pdo,
    tableName: 'embeddings',
    vectorFieldName: 'embedding',
    distance: Distance::Cosine,  // Cosine, InnerProduct, L1, L2
);

// 创建表和索引
$store->setup([
    'vector_type' => 'vector',    // 或 'halfvec'（半精度，省空间）
    'vector_size' => 1536,
    'index_method' => 'hnsw',     // HNSW 索引加速查询
]);
```

### 8.6 

#### Qdrant

```php
use Symfony\AI\Store\Bridge\Qdrant\Store;

$store = new Store(
    httpClient: $httpClient,
    collectionName: 'my_collection',
    embeddingsDimension: 1536,
    embeddingsDistance: 'Cosine',
);
$store->setup();
```

#### RedisRedisSearch

```php
use Symfony\AI\Store\Bridge\Redis\Store;
use Symfony\AI\Store\Bridge\Redis\Distance;

$store = new Store(
    redis: $redis,
    indexName: 'vector_index',
    keyPrefix: 'vector:',
    distance: Distance::Cosine,
);
$store->setup(['vector_size' => 1536, 'index_method' => 'HNSW']);
```

#### InMemory/

```php
use Symfony\AI\Store\InMemory\Store;
use Symfony\AI\Store\Distance\DistanceStrategy;

$store = new Store(
    strategy: DistanceStrategy::COSINE_DISTANCE,
);
```

#### CombinedStore

```php
use Symfony\AI\Store\CombinedStore;
use Symfony\AI\Store\Query\HybridQuery;

// 用 Qdrant 做向量搜索，Elasticsearch 做全文搜索
$combined = new CombinedStore(
    vectorStore: $qdrantStore,
    textStore: $elasticsearchStore,
    rrfK: 60,  // Reciprocal Rank Fusion 常数
);

// 写入时同步到两个后端
$combined->add($vectorDocuments);

// HybridQuery 自动分解为向量+文本查询，结果通过 RRF 算法融合
$results = $combined->query(
    new HybridQuery($vector, 'search terms', semanticRatio: 0.7),
    ['limit' => 10],
);
```

### 8.7 

```mermaid
flowchart TD
    A["需要持久化?"] -->|否| B["InMemory Store<br>（测试/原型）"]
    A -->|是| C["已有数据库?"]
    C -->|PostgreSQL| D["Postgres Bridge<br>（pgvector）"]
    C -->|Redis| E["Redis Bridge<br>（RedisSearch）"]
    C -->|Elasticsearch| F["Elasticsearch Bridge"]
    C -->|无| G["需要什么规模?"]
    G -->|"< 100万文档"| H["SQLite / ChromaDB<br>（轻量级）"]
    G -->|"> 100万文档"| I["Qdrant / Milvus<br>（高性能）"]
    G -->|全托管云服务| J["Pinecone / Qdrant Cloud"]

    style B fill:#e8f5e9
    style D fill:#e3f2fd
    style I fill:#fff3e0
```

---

## 9. 

### 9.1 QueryInterface

 `QueryInterface`

```php
namespace Symfony\AI\Store\Query;

interface QueryInterface {}
```

### 9.2 VectorQuery



```php
use Symfony\AI\Store\Query\VectorQuery;
use Symfony\AI\Platform\Vector\Vector;

$query = new VectorQuery(
    vector: new Vector([0.12, -0.34, 0.56, ...]),
);

$results = $store->query($query, ['limit' => 5]);

foreach ($results as $doc) {
    echo $doc->getMetadata()->getText() . "\n";
    echo "相似度得分：" . $doc->getScore() . "\n\n";
}
```

### 9.3 TextQuery



```php
use Symfony\AI\Store\Query\TextQuery;

// 单个文本搜索
$query = new TextQuery('退款政策');

// 多文本搜索（OR 逻辑）
$query = new TextQuery(['退款', '政策', 'refund']);

$results = $store->query($query, ['limit' => 10]);
```

### 9.4 HybridQuery

 `semanticRatio` 

```php
use Symfony\AI\Store\Query\HybridQuery;

$query = new HybridQuery(
    vector: $vector,
    text: 'Symfony AI 退款',
    semanticRatio: 0.7,  // 70% 语义权重，30% 关键词权重
);

echo $query->getSemanticRatio(); // 0.7
echo $query->getKeywordRatio();  // 0.3（自动计算）
```

| semanticRatio | | | |
|:-------------:|:--------:|:----------:|----------|
| 0.0 | 0% | 100% | |
| 0.3 | 30% | 70% | |
| 0.5 | 50% | 50% | FAQ |
| 0.7 | 70% | 30% | |
| 1.0 | 100% | 0% | |

### 9.5 

 `$options` 

```php
$results = $store->query($query, [
    'limit' => 10,            // 返回结果数量
    'where' => [...],         // 元数据过滤条件（部分后端支持）
    'params' => [...],        // 后端特定参数
]);
```

### 9.6 supports() 



```php
use Symfony\AI\Store\Query\HybridQuery;
use Symfony\AI\Store\Query\VectorQuery;
use Symfony\AI\Store\Query\TextQuery;

if ($store->supports(HybridQuery::class)) {
    $query = new HybridQuery($vector, $searchText, semanticRatio: 0.7);
} elseif ($store->supports(VectorQuery::class)) {
    $query = new VectorQuery($vector);
} else {
    $query = new TextQuery($searchText);
}

$results = $store->query($query, ['limit' => 10]);
```

---

## 10. Retriever

### 10.1 Retriever 

`Retriever` Store API + + 

```php
use Symfony\AI\Store\Retriever;

$retriever = new Retriever(
    store: $store,
    vectorizer: $vectorizer,
    eventDispatcher: $dispatcher,  // 可选，用于 Pre/PostQueryEvent
    logger: $logger,               // 可选 PSR-3 日志
);

// 只需传入自然语言查询
$documents = $retriever->retrieve('如何申请退款？', ['limit' => 5]);

foreach ($documents as $doc) {
    echo $doc->getMetadata()->getText() . "\n";
    echo "得分：" . $doc->getScore() . "\n\n";
}
```

### 10.2 Retriever 

`Retriever` Store 

```mermaid
flowchart TD
    A["retrieve(query, options)"] --> B{"有 Vectorizer?"}
    B -->|否| C["TextQuery<br>（纯关键词）"]
    B -->|是| D{"Store 支持<br>HybridQuery?"}
    D -->|是| E["HybridQuery<br>（向量 + 关键词）"]
    D -->|否| F{"Store 支持<br>VectorQuery?"}
    F -->|是| G["VectorQuery<br>（纯语义）"]
    F -->|否| C

    style E fill:#e8f5e9
    style G fill:#e3f2fd
    style C fill:#fff3e0
```

### 10.3 Agent SimilaritySearch 

Agent `SimilaritySearch` AI 

```php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 1. 创建 SimilaritySearch 工具
$similaritySearch = new SimilaritySearch($vectorizer, $store);

// 2. 构建 Agent
$toolbox = new Toolbox([$similaritySearch]);
$agentProcessor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'gpt-4o-mini', [$agentProcessor], [$agentProcessor]);

// 3. Agent 会自动调用 SimilaritySearch 检索相关文档
$result = $agent->call(new MessageBag(
    Message::forSystem('你是产品客服。根据 SimilaritySearch 检索到的文档回答用户问题。'),
    Message::ofUser('你们支持退款吗？'),
));

echo $result->getContent();
```

> `SimilaritySearch` `#[AsTool('similarity_search')]` LLM Store

---

## 11. Reranking

### 11.1 

**Bi-Encoder**——

**Cross-Encoder** 

**** 

```
向量搜索（快，取 50 条）→ Reranker 精排（慢但准，取 5 条）→ LLM 生成回答
```

```mermaid
flowchart LR
    A["用户提问"] --> B["向量搜索<br>（Bi-Encoder）<br>快速取 50 条"]
    B --> C["Reranker<br>（Cross-Encoder）<br>精排取 5 条"]
    C --> D["LLM<br>基于 5 条精排文档<br>生成回答"]

    style B fill:#e3f2fd
    style C fill:#fff3e0
    style D fill:#e8f5e9
```

### 11.2 RerankerInterface Reranker

```php
use Symfony\AI\Store\Reranker\Reranker;

$reranker = new Reranker(
    platform: $platform,
    model: 'rerank-v3.5',  // Cohere 等提供的重排序模型
    logger: $logger,
);

// 对检索结果重排序
$reranked = $reranker->rerank(
    query: '如何申请退款？',
    documents: $initialResults,  // VectorDocument 数组
    topK: 5,                     // 返回前 5 条
);

foreach ($reranked as $doc) {
    echo $doc->getMetadata()->getText() . "\n";
    echo "交叉编码器得分：" . $doc->getScore() . "\n\n";
}
```

> **Reranker `Metadata::getText()`** `Vectorizer` `_text` Reranker

### 11.3 RerankerListener

****

```php
use Symfony\AI\Store\EventListener\RerankerListener;
use Symfony\AI\Store\Event\PostQueryEvent;

$listener = new RerankerListener(
    reranker: $reranker,
    topK: 5,
);

// 注册到事件分发器
$dispatcher->addListener(PostQueryEvent::class, $listener);

// 之后所有 Retriever::retrieve() 调用都自动经过 Reranker
$docs = $retriever->retrieve('如何退款？', ['limit' => 50]);
// 实际返回 5 条重排序后的精排结果
```

### 11.4 Reranker 

| | | | |
|------|--------|----------|------|
| `rerank-v3.5` | Cohere | | |
| `rerank-english-v3.0` | Cohere | | |
| `jina-reranker-v2-base-multilingual` | Jina AI | | |
| `bge-reranker-large` | BAAI | | |

---

## 12. 

Store Symfony 

### 12.1 PreQueryEvent



```php
use Symfony\AI\Store\Event\PreQueryEvent;

// 示例：拼写纠正
$dispatcher->addListener(PreQueryEvent::class, function (PreQueryEvent $event): void {
    $corrected = $spellingCorrector->correct($event->getQuery());
    $event->setQuery($corrected);
});

// 示例：同义词扩展
$dispatcher->addListener(PreQueryEvent::class, function (PreQueryEvent $event): void {
    $options = $event->getOptions();
    $options['synonyms'] = $synonymDict->expand($event->getQuery());
    $event->setOptions($options);
});
```

****A/B `semanticRatio`

### 12.2 PostQueryEvent



```php
use Symfony\AI\Store\Event\PostQueryEvent;

// 示例：过滤低分结果
$dispatcher->addListener(PostQueryEvent::class, function (PostQueryEvent $event): void {
    $docs = iterator_to_array($event->getDocuments());
    $filtered = array_filter($docs, fn ($doc) => $doc->getScore() > 0.7);
    $event->setDocuments(array_values($filtered));
});

// 示例：搜索日志
$dispatcher->addListener(PostQueryEvent::class, function (PostQueryEvent $event): void {
    $logger->info('搜索查询', [
        'query' => $event->getQuery(),
        'result_count' => count(iterator_to_array($event->getDocuments())),
    ]);
});
```

****`RerankerListener`

---

## 13. 

### 13.1 DistanceStrategy

`DistanceStrategy` 

```php
use Symfony\AI\Store\Distance\DistanceStrategy;

DistanceStrategy::COSINE_DISTANCE;    // 余弦距离（最常用）
DistanceStrategy::ANGULAR_DISTANCE;   // 角距离
DistanceStrategy::EUCLIDEAN_DISTANCE; // 欧几里得距离（L2）
DistanceStrategy::MANHATTAN_DISTANCE; // 曼哈顿距离（L1）
DistanceStrategy::CHEBYSHEV_DISTANCE; // 切比雪夫距离
```

| | | | |
|----------|------|----------|----------|
| | `1 - cos(A, B)` | | OpenAI Embedding |
| | `√Σ(aᵢ-bᵢ)²` | | CLIP |
| | `Σ(aᵢ·bᵢ)` | | |
| | `Σ\|aᵢ-bᵢ\|` | | — |
| | `max\|aᵢ-bᵢ\|` | | — |

> **** OpenAI text-embedding-3-*****

### 13.2 DistanceCalculator

`DistanceCalculator` InMemory Store 

```php
use Symfony\AI\Store\Distance\DistanceCalculator;
use Symfony\AI\Store\Distance\DistanceStrategy;

$calculator = new DistanceCalculator(
    strategy: DistanceStrategy::COSINE_DISTANCE,
    batchSize: 100, // 大数据集分批计算，控制内存
);

// 计算查询向量与文档集合的距离并排序
$sorted = $calculator->calculate(
    documents: $allDocuments,
    vector: $queryVector,
    maxItems: 10,
);
```

---

## 14. RAG 

 RAG AI 

### 14.1 

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Store\Bridge\Postgres\Store;
use Symfony\AI\Store\Document\Loader\InMemoryLoader;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;
use Symfony\AI\Store\Document\Transformer\TextTrimTransformer;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentProcessor;
use Symfony\AI\Store\Indexer\SourceIndexer;
use Symfony\Component\HttpClient\HttpClient;
use Symfony\Component\Uid\Uuid;

// 1. 创建 Platform 和 Vectorizer
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());
$vectorizer = new Vectorizer($platform, 'text-embedding-3-small');

// 2. 创建 PostgreSQL 存储
$pdo = new \PDO('pgsql:host=localhost;dbname=knowledge_base', 'postgres', 'secret');
$store = Store::fromPdo($pdo, 'document_embeddings');
$store->setup();

// 3. 准备知识库文档
$knowledgeBase = [
    new TextDocument(
        id: Uuid::v4(),
        content: 'CloudFlow 提供 14 天无理由退款。用户在购买后 14 天内可以申请全额退款。'
            . '退款将在 3-5 个工作日内退回原支付方式。',
        metadata: new Metadata(['_title' => '退款政策', 'category' => 'billing']),
    ),
    new TextDocument(
        id: Uuid::v4(),
        content: 'CloudFlow 有三个版本：基础版 ¥99/月（5 用户、10GB 存储）；'
            . '专业版 ¥299/月（20 用户、100GB 存储、API 访问）；'
            . '企业版按需定价（无限用户、私有部署）。',
        metadata: new Metadata(['_title' => '定价方案', 'category' => 'billing']),
    ),
    new TextDocument(
        id: Uuid::v4(),
        content: 'CloudFlow 数据安全：AES-256 加密、SOC 2 Type II 和 ISO 27001 认证。'
            . '支持 SSO（SAML 2.0）和双因素认证。企业版支持私有部署。',
        metadata: new Metadata(['_title' => '数据安全', 'category' => 'security']),
    ),
];

// 4. 构建索引管线并执行
$processor = new DocumentProcessor(
    vectorizer: $vectorizer,
    store: $store,
    transformers: [
        new TextTrimTransformer(),
        new TextSplitTransformer(chunkSize: 800, overlap: 150),
    ],
);

$indexer = new SourceIndexer(
    loader: new InMemoryLoader($knowledgeBase),
    processor: $processor,
);

$indexer->index();
echo "✅ 索引了 " . count($knowledgeBase) . " 篇文档\n";
```

### 14.2 Retriever

```php
use Symfony\AI\Store\Retriever;
use Symfony\AI\Store\Reranker\Reranker;
use Symfony\AI\Store\EventListener\RerankerListener;
use Symfony\AI\Store\Event\PostQueryEvent;
use Symfony\Component\EventDispatcher\EventDispatcher;

// 5. 可选：配置 Reranker
$dispatcher = new EventDispatcher();
$reranker = new Reranker($platform, 'rerank-v3.5');
$dispatcher->addListener(
    PostQueryEvent::class,
    new RerankerListener($reranker, topK: 3),
);

// 6. 创建 Retriever
$retriever = new Retriever(
    store: $store,
    vectorizer: $vectorizer,
    eventDispatcher: $dispatcher,
);

// 7. 检索相关文档
$documents = $retriever->retrieve('你们支持退款吗？', ['limit' => 20]);

foreach ($documents as $doc) {
    echo "📄 " . $doc->getMetadata()->getTitle() . "\n";
    echo "   " . $doc->getMetadata()->getText() . "\n";
    echo "   得分：" . round($doc->getScore(), 4) . "\n\n";
}
```

### 14.3 Agent

```php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 8. 构建 RAG Agent
$similaritySearch = new SimilaritySearch($vectorizer, $store);
$toolbox = new Toolbox([$similaritySearch]);
$agentProcessor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'gpt-4o-mini', [$agentProcessor], [$agentProcessor]);

$systemPrompt = '你是 CloudFlow 产品客服。'
    . '只根据 SimilaritySearch 工具检索到的文档内容回答用户问题。'
    . '如果没有找到相关信息，请诚实告知用户。';

// 9. 用户提问
$questions = [
    '你们支持退款吗？多久能到账？',
    '最便宜的方案多少钱？',
    '你们通过了哪些安全认证？',
    '你们支持 GraphQL 吗？',  // 知识库中没有的信息
];

foreach ($questions as $question) {
    echo "👤 用户：{$question}\n";

    $result = $agent->call(new MessageBag(
        Message::forSystem($systemPrompt),
        Message::ofUser($question),
    ));

    echo "🤖 客服：" . $result->getContent() . "\n\n";
}
```

### 14.4 Symfony 

 Symfony AI Bundle 

```yaml
# config/services.yaml
services:
    # 向量化器
    Symfony\AI\Store\Document\Vectorizer:
        arguments:
            $platform: '@Symfony\AI\Platform\PlatformInterface'
            $model: 'text-embedding-3-small'

    # PostgreSQL 存储
    Symfony\AI\Store\Bridge\Postgres\Store:
        arguments:
            $connection: '@database_connection'
            $tableName: 'document_embeddings'

    # 文档处理器
    Symfony\AI\Store\Indexer\DocumentProcessor:
        arguments:
            $vectorizer: '@Symfony\AI\Store\Document\Vectorizer'
            $store: '@Symfony\AI\Store\Bridge\Postgres\Store'
            $transformers:
                - '@Symfony\AI\Store\Document\Transformer\TextTrimTransformer'
                - '@text_split_transformer'

    text_split_transformer:
        class: Symfony\AI\Store\Document\Transformer\TextSplitTransformer
        arguments:
            $chunkSize: 1000
            $overlap: 200

    # 来源索引器
    Symfony\AI\Store\Indexer\SourceIndexer:
        arguments:
            $loader: '@Symfony\AI\Store\Document\Loader\MarkdownLoader'
            $processor: '@Symfony\AI\Store\Indexer\DocumentProcessor'

    # 检索器
    Symfony\AI\Store\Retriever:
        arguments:
            $store: '@Symfony\AI\Store\Bridge\Postgres\Store'
            $vectorizer: '@Symfony\AI\Store\Document\Vectorizer'
            $eventDispatcher: '@event_dispatcher'
```

 CLI 

```bash
# 初始化存储
php bin/console ai:store:setup

# 索引文档
php bin/console ai:index blog --source=/docs/knowledge-base/

# 测试检索
php bin/console ai:retrieve blog "如何申请退款？"

# 清空存储
php bin/console ai:store:drop
```

---

## 15. RAG 

| | | | |
|------|--------|----------|------|
| `chunkSize` | 1000 | : 500: 2000 | |
| `overlap` | 200 | chunkSize 10~20% | |
| `limit` | — | Reranker: 20~50 | |
| `topK`Reranker | 5 | 3~10 | |
| `semanticRatio` | 0.5 | : 0.7: 0.3 | |
| | 1536 | : 3072: 768 | |

> ****`chunkSize=800` + `overlap=150` + `limit=20`+ `Reranker topK=5` `text-embedding-3-small` + 

---

## 16. 

 Store 

- ****TextDocumentVectorDocumentMetadata 
- ****8 Loader + 6 Transformer 
- ****Vectorizer + DocumentProcessor + SourceIndexer 
- **25+ ** InMemory PostgreSQL 
- ****VectorQueryTextQueryHybridQuery 
- ****Retriever + Reranker 
- ****PreQueryEvent / PostQueryEvent 
- **Agent **SimilaritySearch AI 

 [ 5 Chat ](05-chat.md) AI PlatformAgent Store 
