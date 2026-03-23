# Store 组件

## 概述

Store 组件是 Symfony AI 单体仓库中负责**向量数据库抽象**的核心组件。它提供了一套统一接口，让应用程序能以一致的方式与各种向量数据库和全文搜索引擎交互，从而支持**检索增强生成（RAG）**等 AI 应用模式。

### 主要功能

- **向量存储抽象**：通过统一的 `StoreInterface` 接口操作任意向量数据库
- **文档管理**：完整的文档加载、转换、向量化、存储流水线
- **多种查询类型**：向量查询、全文查询、混合查询
- **重排序支持**：通过交叉编码器模型对检索结果重新排序
- **25+ 存储后端**：涵盖主流向量数据库和搜索引擎
- **RAG 模式**：为检索增强生成提供完整的工具链

---

## 架构

### 目录结构

```
src/store/src/
├── StoreInterface.php              # 核心存储接口
├── ManagedStoreInterface.php       # 可管理存储接口（setup/drop）
├── IndexerInterface.php            # 索引器接口
├── RetrieverInterface.php          # 检索器接口
├── Retriever.php                   # 检索器实现
├── CombinedStore.php               # 组合存储（写入多个后端）
│
├── Document/                       # 文档系统
│   ├── TextDocument.php            # 文本文档
│   ├── VectorDocument.php          # 向量文档
│   ├── Metadata.php                # 文档元数据
│   ├── EmbeddableDocumentInterface.php
│   ├── LoaderInterface.php
│   ├── TransformerInterface.php
│   ├── FilterInterface.php
│   ├── VectorizerInterface.php
│   ├── Vectorizer.php
│   ├── Loader/                     # 文档加载器
│   │   ├── CsvLoader.php
│   │   ├── MarkdownLoader.php
│   │   ├── RssFeedLoader.php
│   │   ├── RstLoader.php
│   │   ├── RstToctreeLoader.php
│   │   ├── TextFileLoader.php
│   │   ├── JsonFileLoader.php
│   │   └── InMemoryLoader.php
│   ├── Transformer/                # 文档转换器
│   │   ├── TextSplitTransformer.php
│   │   ├── SummaryGeneratorTransformer.php
│   │   ├── TextReplaceTransformer.php
│   │   ├── TextTrimTransformer.php
│   │   ├── ChunkDelayTransformer.php
│   │   └── ChainTransformer.php
│   └── Filter/
│       └── TextContainsFilter.php
│
├── Indexer/                        # 索引系统
│   ├── DocumentIndexer.php
│   ├── SourceIndexer.php
│   ├── DocumentProcessor.php
│   └── ConfiguredSourceIndexer.php
│
├── Query/                          # 查询系统
│   ├── QueryInterface.php
│   ├── VectorQuery.php
│   ├── TextQuery.php
│   └── HybridQuery.php
│
├── Reranker/                       # 重排序
│   ├── RerankerInterface.php
│   └── Reranker.php
│
├── EventListener/
│   └── RerankerListener.php
│
├── Event/                          # 事件
│   ├── PreQueryEvent.php
│   └── PostQueryEvent.php
│
├── Distance/                       # 距离计算
│   ├── DistanceStrategy.php
│   └── DistanceCalculator.php
│
├── InMemory/
│   └── Store.php                   # 内存存储
│
├── Command/                        # CLI 命令
│   ├── SetupStoreCommand.php
│   ├── DropStoreCommand.php
│   ├── IndexCommand.php
│   └── RetrieveCommand.php
│
├── Exception/                      # 异常体系
│   ├── ExceptionInterface.php
│   ├── InvalidArgumentException.php
│   ├── RuntimeException.php
│   ├── UnsupportedFeatureException.php
│   └── UnsupportedQueryTypeException.php
│
├── Test/
│   └── AbstractStoreIntegrationTestCase.php
│
└── Bridge/                         # 存储后端桥接器
    ├── AzureSearch/
    ├── Cache/
    ├── ChromaDb/
    ├── ClickHouse/
    ├── Cloudflare/
    ├── Elasticsearch/
    ├── ManticoreSearch/
    ├── MariaDb/
    ├── Meilisearch/
    ├── Milvus/
    ├── MongoDb/
    ├── Neo4j/
    ├── OpenSearch/
    ├── Pinecone/
    ├── Postgres/
    ├── Qdrant/
    ├── Redis/
    ├── S3Vectors/
    ├── Sqlite/
    ├── Supabase/
    ├── SurrealDb/
    ├── Typesense/
    ├── Vektor/
    └── Weaviate/
```

---

## 核心概念

### StoreInterface

`StoreInterface` 是整个组件的核心接口，定义了向量存储的基本操作：

```php
interface StoreInterface
{
    // 添加一个或多个向量文档
    public function add(VectorDocument|array $documents): void;

    // 按 ID 删除文档
    public function remove(string|array $ids, array $options = []): void;

    // 执行查询，返回匹配的向量文档
    public function query(QueryInterface $query, array $options = []): iterable;

    // 检查是否支持某种查询类型
    public function supports(string $queryClass): bool;
}
```

**示例：直接使用 StoreInterface**

```php
use Symfony\AI\Store\Query\VectorQuery;
use Symfony\AI\Platform\Vector\Vector;

// 添加文档
$store->add($vectorDocument);

// 查询相似文档
$query = new VectorQuery(new Vector([0.1, 0.2, 0.3, ...]));
$results = $store->query($query, ['limit' => 10]);

foreach ($results as $document) {
    echo $document->getMetadata()->getText();
    echo $document->getScore();
}
```

### ManagedStoreInterface

`ManagedStoreInterface` 扩展存储接口，允许对底层基础设施进行管理操作：

```php
interface ManagedStoreInterface
{
    // 创建/初始化存储基础设施（如创建数据库表、集合、索引等）
    public function setup(array $options = []): void;

    // 删除/清空存储基础设施
    public function drop(array $options = []): void;
}
```

大多数桥接器实现都同时实现了 `StoreInterface` 和 `ManagedStoreInterface`。

### IndexerInterface

`IndexerInterface` 定义了文档索引的入口点：

```php
interface IndexerInterface
{
    /**
     * 处理输入，通过完整的文档流水线：
     * 加载/接受 → 过滤 → 转换 → 向量化 → 存储
     */
    public function index(string|iterable|object $input, array $options = []): void;
}
```

### RetrieverInterface

`RetrieverInterface` 是索引器的对立面，负责从存储中检索相似文档：

```php
interface RetrieverInterface
{
    /**
     * 对查询字符串进行向量化，然后检索相似文档
     */
    public function retrieve(string $query, array $options = []): iterable;
}
```

---

## 文档系统

### TextDocument

`TextDocument` 表示一个可嵌入的文本文档，是向量化之前的原始文档形式：

```php
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Metadata;

$document = new TextDocument(
    id: 'doc-001',          // 文档 ID（字符串或整数）
    content: '这是文档内容', // 文本内容（不能为空）
    metadata: new Metadata(['source' => 'example.txt']),
);

// 创建带新内容的副本
$newDocument = $document->withContent('更新后的内容');

echo $document->getId();      // 'doc-001'
echo $document->getContent(); // '这是文档内容'
```

`TextDocument` 实现了 `EmbeddableDocumentInterface`，可以被 `Vectorizer` 转换为 `VectorDocument`。

### VectorDocument

`VectorDocument` 包含文档 ID、嵌入向量和元数据，是实际存入向量数据库的数据结构：

```php
use Symfony\AI\Store\Document\VectorDocument;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Platform\Vector\Vector;

$document = new VectorDocument(
    id: 'doc-001',
    vector: new Vector([0.1, 0.2, 0.3, ...]), // 嵌入向量
    metadata: new Metadata(['_text' => '文档内容']),
    score: null, // 检索得分，查询后填充
);

// 创建带得分的副本（不可变模式）
$scored = $document->withScore(0.95);

echo $scored->getScore(); // 0.95
```

### Metadata

`Metadata` 继承自 `\ArrayObject`，提供类型安全的元数据访问：

```php
use Symfony\AI\Store\Document\Metadata;

$metadata = new Metadata();

// 预定义键的便捷方法
$metadata->setText('原始文本内容');           // _text
$metadata->setSource('/path/to/file.txt');   // _source
$metadata->setSummary('文档摘要');            // _summary
$metadata->setTitle('文档标题');              // _title
$metadata->setParentId('parent-doc-001');    // _parent_id（分块时使用）
$metadata->setDepth(2);                      // _depth（目录树深度）

// 检查和读取
if ($metadata->hasText()) {
    echo $metadata->getText();
}

// 预定义的元数据键常量
Metadata::KEY_TEXT;       // '_text'
Metadata::KEY_SOURCE;     // '_source'
Metadata::KEY_SUMMARY;    // '_summary'
Metadata::KEY_TITLE;      // '_title'
Metadata::KEY_PARENT_ID;  // '_parent_id'
Metadata::KEY_DEPTH;      // '_depth'
```

### Vectorizer（向量化器）

`Vectorizer` 是 `VectorizerInterface` 的具体实现，通过 Platform 组件调用 AI 模型生成嵌入向量：

```php
use Symfony\AI\Store\Document\Vectorizer;

$vectorizer = new Vectorizer(
    platform: $platform,       // PlatformInterface 实例
    model: 'text-embedding-3-small',
    logger: $logger,           // 可选 PSR-3 日志记录器
);

// 向量化单个字符串
$vector = $vectorizer->vectorize('这是要向量化的文本');

// 向量化单个文档
$vectorDocument = $vectorizer->vectorize($textDocument);

// 批量向量化（支持批处理的模型会自动使用批量 API）
$vectorDocuments = $vectorizer->vectorize([$doc1, $doc2, $doc3]);
```

向量化器会自动在元数据中保留原始文本（`_text` 键），以便后续的重排序等操作使用。

---

## 文档加载器

文档加载器实现 `LoaderInterface`，负责从各种来源加载文档：

```php
interface LoaderInterface
{
    public function load(?string $source = null, array $options = []): iterable;
}
```

### CsvLoader（CSV 加载器）

从 CSV 文件加载文档，支持灵活的列映射：

```php
use Symfony\AI\Store\Document\Loader\CsvLoader;

$loader = new CsvLoader(
    contentColumn: 'description',  // 内容列（列名或索引）
    idColumn: 'id',                // ID 列（可选）
    metadataColumns: ['category', 'date'], // 元数据列
    delimiter: ',',
    hasHeader: true,
);

foreach ($loader->load('/path/to/data.csv') as $document) {
    echo $document->getContent();
    echo $document->getMetadata()['category'];
}
```

### MarkdownLoader（Markdown 加载器）

从 Markdown 文件加载文档，自动提取标题作为元数据：

```php
use Symfony\AI\Store\Document\Loader\MarkdownLoader;

$loader = new MarkdownLoader();

// strip_formatting: true 会剥离 Markdown 语法，只保留纯文本
foreach ($loader->load('/docs/guide.md', ['strip_formatting' => true]) as $document) {
    echo $document->getMetadata()->getTitle(); // 从 # 标题提取
    echo $document->getContent();
}
```

### RssFeedLoader（RSS 订阅加载器）

从 RSS/Atom 订阅 URL 加载文章：

```php
use Symfony\AI\Store\Document\Loader\RssFeedLoader;

$loader = new RssFeedLoader(
    httpClient: $httpClient,
    uuidNamespace: '6ba7b810-9dad-11d1-80b4-00c04fd430c8', // UUID v5 命名空间
);

foreach ($loader->load('https://example.com/feed.rss') as $document) {
    echo $document->getMetadata()['title'];
    echo $document->getMetadata()['link'];
}
```

### RstLoader（reStructuredText 加载器）

从 RST 文件加载文档：

```php
use Symfony\AI\Store\Document\Loader\RstLoader;

$loader = new RstLoader();
foreach ($loader->load('/docs/chapter.rst') as $document) {
    echo $document->getContent();
}
```

### RstToctreeLoader（RST 目录树加载器）

递归加载 RST 文档目录树：

```php
use Symfony\AI\Store\Document\Loader\RstToctreeLoader;

$loader = new RstToctreeLoader();
// 从 index.rst 递归加载所有引用的文档
foreach ($loader->load('/docs/index.rst') as $document) {
    echo $document->getMetadata()->getDepth();  // 文档在目录树中的深度
    echo $document->getMetadata()->getTitle();
}
```

### TextFileLoader（文本文件加载器）

从纯文本文件加载文档：

```php
use Symfony\AI\Store\Document\Loader\TextFileLoader;

$loader = new TextFileLoader();
foreach ($loader->load('/path/to/document.txt') as $document) {
    echo $document->getContent();
}
```

### JsonFileLoader（JSON 文件加载器）

从 JSON 文件加载文档：

```php
use Symfony\AI\Store\Document\Loader\JsonFileLoader;

$loader = new JsonFileLoader();
foreach ($loader->load('/path/to/data.json') as $document) {
    echo $document->getContent();
}
```

### InMemoryLoader（内存加载器）

直接从内存中的文档数组加载，适合测试或动态内容场景：

```php
use Symfony\AI\Store\Document\Loader\InMemoryLoader;
use Symfony\AI\Store\Document\TextDocument;

$loader = new InMemoryLoader([$document1, $document2]);
foreach ($loader->load() as $document) {
    echo $document->getContent();
}
```

---

## 文档转换器

文档转换器实现 `TransformerInterface`，接受文档可迭代对象并返回新的可迭代对象：

```php
interface TransformerInterface
{
    public function transform(iterable $documents, array $options = []): iterable;
}
```

### TextSplitTransformer（文本分割转换器）

将长文档切分为带有重叠的小块，适合处理超过模型上下文长度的文档：

```php
use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;

$transformer = new TextSplitTransformer(
    chunkSize: 1000, // 每块字符数
    overlap: 200,    // 块间重叠字符数（防止信息丢失）
);

$chunks = iterator_to_array($transformer->transform($documents));
// 每个 chunk 的元数据中包含 _parent_id（原文档 ID）
```

运行时也可以通过 `$options` 覆盖参数：

```php
$chunks = $transformer->transform($documents, [
    TextSplitTransformer::OPTION_CHUNK_SIZE => 500,
    TextSplitTransformer::OPTION_OVERLAP => 100,
]);
```

### SummaryGeneratorTransformer（摘要生成转换器）

使用 LLM 为每个文档生成摘要，并存入元数据的 `_summary` 字段：

```php
use Symfony\AI\Store\Document\Transformer\SummaryGeneratorTransformer;

$transformer = new SummaryGeneratorTransformer(
    platform: $platform,
    model: 'gpt-4o-mini',
    yieldSummaryDocuments: true,   // 同时生成独立的摘要文档（双重索引模式）
    systemPrompt: '用2-3句话总结以下文本。',
);

// 启用双重索引时，每个原文档会额外生成一个摘要文档
foreach ($transformer->transform($documents) as $doc) {
    echo $doc->getMetadata()->getSummary();
}
```

### TextReplaceTransformer（文本替换转换器）

对文档内容进行字符串替换：

```php
use Symfony\AI\Store\Document\Transformer\TextReplaceTransformer;

$transformer = new TextReplaceTransformer(
    search: '旧内容',
    replace: '新内容',
);
```

### TextTrimTransformer（文本修剪转换器）

去除文档内容首尾空白字符：

```php
use Symfony\AI\Store\Document\Transformer\TextTrimTransformer;

$transformer = new TextTrimTransformer();
```

### ChunkDelayTransformer（分块延迟转换器）

在文档块之间添加延迟，避免触发 API 速率限制：

```php
use Symfony\AI\Store\Document\Transformer\ChunkDelayTransformer;

$transformer = new ChunkDelayTransformer(delayMs: 1000); // 每块延迟 1 秒
```

### ChainTransformer（链式转换器）

将多个转换器串联在一起：

```php
use Symfony\AI\Store\Document\Transformer\ChainTransformer;

$transformer = new ChainTransformer(
    new TextTrimTransformer(),
    new TextReplaceTransformer('旧', '新'),
    new TextSplitTransformer(1000, 200),
);
```

---

## 文档过滤器

过滤器实现 `FilterInterface`，从文档流中移除不符合条件的文档：

```php
interface FilterInterface
{
    public function filter(iterable $documents): iterable;
}
```

### TextContainsFilter（文本包含过滤器）

只保留内容中包含特定文本的文档：

```php
use Symfony\AI\Store\Document\Filter\TextContainsFilter;

$filter = new TextContainsFilter('关键词');
// 只有包含"关键词"的文档才会通过
```

---

## 索引系统

### DocumentIndexer（文档索引器）

`DocumentIndexer` 直接接受文档对象并通过处理流水线进行处理：

```php
use Symfony\AI\Store\Indexer\DocumentIndexer;
use Symfony\AI\Store\Indexer\DocumentProcessor;

$processor = new DocumentProcessor(
    vectorizer: $vectorizer,
    store: $store,
    filters: [$textContainsFilter],
    transformers: [$textSplitTransformer],
    logger: $logger,
);

$indexer = new DocumentIndexer($processor);

// 索引单个文档
$indexer->index($textDocument);

// 批量索引
$indexer->index([$doc1, $doc2, $doc3]);

// 带处理选项
$indexer->index($documents, [
    'chunk_size' => 50,                     // 每次向量化的文档数
    'platform_options' => ['timeout' => 30], // 传递给 Platform 的选项
]);
```

### SourceIndexer（来源索引器）

`SourceIndexer` 接受文件路径或 URL 等来源标识符，先通过加载器加载文档，再进行处理：

```php
use Symfony\AI\Store\Indexer\SourceIndexer;

$indexer = new SourceIndexer(
    loader: $csvLoader,
    processor: $processor,
);

// 索引单个来源
$indexer->index('/path/to/documents.csv');

// 批量索引多个来源
$indexer->index([
    '/path/to/docs/chapter1.md',
    '/path/to/docs/chapter2.md',
]);
```

### ConfiguredSourceIndexer（预配置来源索引器）

装饰器模式，允许在 Symfony 容器中预配置默认来源：

```php
use Symfony\AI\Store\Indexer\ConfiguredSourceIndexer;

$indexer = new ConfiguredSourceIndexer(
    indexer: $sourceIndexer,
    defaultSource: '/path/to/documents/', // 默认来源
);

// 使用默认来源
$indexer->index();

// 运行时覆盖来源
$indexer->index('/path/to/other/documents/');
```

### DocumentProcessor（文档处理器）

`DocumentProcessor` 是索引系统内部使用的共享处理流水线，负责 **过滤 → 转换 → 向量化 → 存储** 的完整链路：

```php
use Symfony\AI\Store\Indexer\DocumentProcessor;

$processor = new DocumentProcessor(
    vectorizer: $vectorizer,
    store: $store,
    filters: [new TextContainsFilter('关键词')],
    transformers: [
        new TextTrimTransformer(),
        new TextSplitTransformer(1000, 200),
    ],
    logger: $logger,
);

// 直接调用处理器
$processor->process($documents, ['chunk_size' => 100]);
```

---

## 查询系统

### QueryInterface

所有查询类型都实现 `QueryInterface` 标记接口：

```php
interface QueryInterface {}
```

### VectorQuery（向量查询）

经典的语义相似性搜索，基于向量余弦距离等距离度量：

```php
use Symfony\AI\Store\Query\VectorQuery;
use Symfony\AI\Platform\Vector\Vector;

$query = new VectorQuery(
    vector: new Vector([0.1, 0.2, 0.3, ...])
);

$results = $store->query($query, ['limit' => 5]);
```

### TextQuery（文本查询）

全文关键词搜索，适用于不支持向量的后端，或支持内部向量化的后端（如 ChromaDB）：

```php
use Symfony\AI\Store\Query\TextQuery;

// 单个文本搜索
$query = new TextQuery('Symfony AI 向量存储');

// 多文本搜索（OR 逻辑）
$query = new TextQuery(['Symfony AI', '向量存储', 'RAG']);

$results = $store->query($query, ['limit' => 10]);
```

### HybridQuery（混合查询）

结合语义搜索和关键词搜索，提供更精准的检索结果：

```php
use Symfony\AI\Store\Query\HybridQuery;

$query = new HybridQuery(
    vector: new Vector([0.1, 0.2, ...]),
    text: 'Symfony AI',
    semanticRatio: 0.7,  // 70% 语义权重，30% 关键词权重
);

echo $query->getSemanticRatio();  // 0.7
echo $query->getKeywordRatio();   // 0.3
```

---

## 检索器

`Retriever` 是 `RetrieverInterface` 的实现，将查询字符串向量化后执行语义搜索，并通过事件系统支持重排序等扩展：

```php
use Symfony\AI\Store\Retriever;

$retriever = new Retriever(
    store: $store,
    vectorizer: $vectorizer,
    eventDispatcher: $dispatcher, // 可选，用于 PreQueryEvent/PostQueryEvent
);

// 检索相似文档
$documents = $retriever->retrieve('如何配置 Symfony AI？', ['limit' => 5]);

foreach ($documents as $document) {
    echo $document->getMetadata()->getText();  // 原始文本
    echo $document->getScore();                // 相似度得分
}
```

---

## 重排序

重排序使用交叉编码器（Cross-Encoder）模型对初始检索结果重新评分，提升最终排序质量。

### RerankerInterface

```php
interface RerankerInterface
{
    /**
     * @param list<VectorDocument> $documents
     * @return list<VectorDocument> 重排序后的文档，按得分降序排列，限制为 $topK 条
     */
    public function rerank(string $query, array $documents, int $topK = 5): array;
}
```

### Reranker

基于 `PlatformInterface` 的重排序实现：

```php
use Symfony\AI\Store\Reranker\Reranker;

$reranker = new Reranker(
    platform: $platform,
    model: 'rerank-v3.5',  // 如 Cohere 的 rerank 模型
    logger: $logger,
);

$reranked = $reranker->rerank(
    query: '如何使用 Symfony AI？',
    documents: $initialResults,
    topK: 3,
);
```

### RerankerListener（重排序事件监听器）

通过事件系统将重排序无缝集成到查询流程中：

```php
use Symfony\AI\Store\EventListener\RerankerListener;

$listener = new RerankerListener(
    reranker: $reranker,
    topK: 5,
);

// 监听 PostQueryEvent，自动对结果重排序
$dispatcher->addListener(PostQueryEvent::class, $listener);
```

---

## 距离度量

### DistanceStrategy（距离策略枚举）

```php
use Symfony\AI\Store\Distance\DistanceStrategy;

DistanceStrategy::COSINE_DISTANCE;    // 余弦距离（最常用）
DistanceStrategy::ANGULAR_DISTANCE;   // 角距离
DistanceStrategy::EUCLIDEAN_DISTANCE; // 欧几里得距离
DistanceStrategy::MANHATTAN_DISTANCE; // 曼哈顿距离
DistanceStrategy::CHEBYSHEV_DISTANCE; // 切比雪夫距离
```

### DistanceCalculator（距离计算器）

主要用于 InMemory 存储，在内存中计算向量距离：

```php
use Symfony\AI\Store\Distance\DistanceCalculator;
use Symfony\AI\Store\Distance\DistanceStrategy;

$calculator = new DistanceCalculator(
    strategy: DistanceStrategy::COSINE_DISTANCE,
    batchSize: 100, // 大数据集时分批处理
);

// 计算查询向量与文档集合的距离并排序
$sorted = $calculator->calculate(
    documents: $allDocuments,
    vector: $queryVector,
    maxItems: 10, // 只返回前 10 个结果
);
```

---

## 事件系统

Store 组件通过 Symfony 事件调度器支持查询前后的拦截。

### PreQueryEvent（查询前事件）

在查询执行之前触发，监听器可以修改查询字符串和选项：

```php
use Symfony\AI\Store\Event\PreQueryEvent;

// 查询扩展示例：添加同义词
$dispatcher->addListener(PreQueryEvent::class, function (PreQueryEvent $event): void {
    $query = $event->getQuery();
    // 添加同义词
    $event->setQuery($query . ' OR ' . getSynonyms($query));

    // 修改选项
    $options = $event->getOptions();
    $options['semanticRatio'] = 0.8;
    $event->setOptions($options);
});
```

### PostQueryEvent（查询后事件）

在查询执行之后触发，监听器可以修改文档列表（如重排序）：

```php
use Symfony\AI\Store\Event\PostQueryEvent;

$dispatcher->addListener(PostQueryEvent::class, function (PostQueryEvent $event): void {
    $documents = iterator_to_array($event->getDocuments());

    // 自定义过滤：只保留得分高于阈值的文档
    $filtered = array_filter($documents, fn ($doc) => $doc->getScore() > 0.7);

    $event->setDocuments($filtered);
});
```

---

## InMemory 存储

`InMemory\Store` 是一个基于内存的向量存储实现，使用 `DistanceCalculator` 计算文档相似度，主要用于开发、测试和小型数据集场景：

```php
use Symfony\AI\Store\InMemory\Store;
use Symfony\AI\Store\Distance\DistanceStrategy;

$store = new Store(
    strategy: DistanceStrategy::COSINE_DISTANCE,
);

// 添加文档
$store->add($vectorDocument);

// 查询
$results = $store->query(new VectorQuery($queryVector), ['limit' => 5]);

// 清空
$store->drop();

// 所有查询支持：VectorQuery, TextQuery, HybridQuery
echo $store->supports(VectorQuery::class); // true
```

---

## 存储桥接器

Store 组件提供 25+ 个桥接器，覆盖主流向量数据库和搜索引擎。

### Qdrant

专为向量搜索设计的高性能向量数据库，支持异步写入：

```php
use Symfony\AI\Store\Bridge\Qdrant\Store;

$store = new Store(
    httpClient: $httpClient,
    collectionName: 'my_collection',
    embeddingsDimension: 1536,
    embeddingsDistance: 'Cosine', // Cosine, Euclid, Dot
    async: false,
);

$store->setup(); // 创建集合（如不存在）
$store->add($vectorDocuments);
$store->query(new VectorQuery($vector));
$store->drop(); // 删除集合
```

也可以使用 `StoreFactory` 创建：

```php
use Symfony\AI\Store\Bridge\Qdrant\StoreFactory;

$factory = new StoreFactory($httpClient);
$store = $factory->create('my_collection', 1536, 'Cosine');
```

### PostgreSQL（pgvector）

使用 pgvector 扩展的 PostgreSQL 向量存储，支持 VectorQuery、TextQuery 和 HybridQuery：

```php
use Symfony\AI\Store\Bridge\Postgres\Store;
use Symfony\AI\Store\Bridge\Postgres\Distance;

$store = new Store(
    connection: $pdo,
    tableName: 'embeddings',
    vectorFieldName: 'embedding',
    distance: Distance::L2, // L2, Cosine, InnerProduct, HalfvecL2
);

// 为 Gemini 嵌入向量优化的配置
$store->setup([
    'vector_type' => 'halfvec',
    'vector_size' => 3072,
    'index_method' => 'hnsw',
    'index_opclass' => 'halfvec_cosine_ops',
]);
```

### Redis（RedisSearch）

使用 RedisSearch 模块的向量存储，支持 FLAT 和 HNSW 索引：

```php
use Symfony\AI\Store\Bridge\Redis\Store;
use Symfony\AI\Store\Bridge\Redis\Distance;

$store = new Store(
    redis: $redis,
    indexName: 'vector_index',
    keyPrefix: 'vector:',
    distance: Distance::Cosine,
);

// 为不同嵌入模型配置向量维度
$store->setup([
    'vector_size' => 1536,      // OpenAI: 1536, Mistral: 1024, Gemini: 3072
    'index_method' => 'HNSW',   // FLAT 或 HNSW
]);
```

### Meilisearch

Meilisearch 向量存储，支持 VectorQuery 和 TextQuery：

```php
use Symfony\AI\Store\Bridge\Meilisearch\Store;

$store = new Store(
    httpClient: $httpClient,
    endpointUrl: 'http://localhost:7700',
    apiKey: 'masterKey',
    indexName: 'documents',
);
```

### ChromaDB

ChromaDB 向量存储，支持使用文本直接查询（内部自动向量化）：

```php
use Symfony\AI\Store\Bridge\ChromaDb\Store;

$store = new Store(
    httpClient: $httpClient,
    baseUrl: 'http://localhost:8000',
    collectionName: 'my_collection',
);
```

### MongoDB

使用 MongoDB Atlas Vector Search 的存储实现：

```php
use Symfony\AI\Store\Bridge\MongoDb\Store;

$store = new Store(
    client: $mongoClient,
    database: 'my_database',
    collection: 'embeddings',
    indexName: 'vector_index',
);
```

### Elasticsearch

Elasticsearch 向量存储，支持 kNN 搜索：

```php
use Symfony\AI\Store\Bridge\Elasticsearch\Store;

$store = new Store(
    httpClient: $httpClient,
    host: 'https://localhost:9200',
    apiKey: 'your-api-key',
    indexName: 'documents',
);
```

### Pinecone

Pinecone 托管向量数据库：

```php
use Symfony\AI\Store\Bridge\Pinecone\Store;

$store = new Store(
    httpClient: $httpClient,
    apiKey: 'your-pinecone-key',
    indexHost: 'https://your-index-host.pinecone.io',
);
```

### Milvus

开源高性能向量数据库：

```php
use Symfony\AI\Store\Bridge\Milvus\Store;

$store = new Store(
    httpClient: $httpClient,
    baseUrl: 'http://localhost:19530',
    collectionName: 'my_collection',
);
```

### Weaviate

Weaviate 向量搜索引擎：

```php
use Symfony\AI\Store\Bridge\Weaviate\Store;

$store = new Store(
    httpClient: $httpClient,
    baseUrl: 'http://localhost:8080',
    className: 'Document',
);
```

### Neo4j

图数据库 Neo4j 的向量存储，适用于需要知识图谱的 RAG 场景：

```php
use Symfony\AI\Store\Bridge\Neo4j\Store;

$store = new Store(
    httpClient: $httpClient,
    baseUrl: 'http://localhost:7474',
    username: 'neo4j',
    password: 'password',
    indexName: 'vector_index',
);
```

### SQLite（sqlite-vec）

轻量级的 SQLite 向量存储，使用 sqlite-vec 扩展，适合本地开发：

```php
use Symfony\AI\Store\Bridge\Sqlite\Store;

$store = new Store(
    pdo: $pdo,
    tableName: 'embeddings',
);
```

### Supabase

Supabase（基于 PostgreSQL + pgvector）的 REST API 向量存储：

```php
use Symfony\AI\Store\Bridge\Supabase\Store;

$store = new Store(
    httpClient: $httpClient,
    url: 'https://your-project.supabase.co',
    apiKey: 'your-anon-key',
    table: 'documents',
    vectorFieldName: 'embedding',
    vectorDimension: 1536,
    functionName: 'match_documents', // 需要手动创建的 PostgreSQL 函数
);
```

> **注意**：Supabase 存储需要手动在 PostgreSQL 中创建向量搜索函数。

### Azure AI Search

微软 Azure AI Search 服务：

```php
use Symfony\AI\Store\Bridge\AzureSearch\SearchStore;

$store = new SearchStore(
    httpClient: $httpClient,
    endpointUrl: 'https://your-service.search.windows.net',
    apiKey: 'your-api-key',
    indexName: 'documents',
);
```

### Cloudflare Vectorize

Cloudflare 托管向量存储服务：

```php
use Symfony\AI\Store\Bridge\Cloudflare\Store;

$store = new Store(
    httpClient: $httpClient,
    accountId: 'your-account-id',
    apiKey: 'your-api-key',
    index: 'my-index',
    dimensions: 1536,
    metric: 'cosine',
);
```

### AWS S3 Vectors

AWS S3 向量存储（使用 AsyncAws S3Vectors 客户端）：

```php
use Symfony\AI\Store\Bridge\S3Vectors\Store;

$store = new Store(
    client: $s3VectorsClient,
    vectorBucketName: 'my-vector-bucket',
    indexName: 'my-index',
    filter: [],
    topK: 3,
);

$store->setup([
    'dimension' => 1536,
    'distanceMetric' => DistanceMetric::COSINE,
]);
```

### SurrealDB

SurrealDB 多模型数据库的向量存储：

```php
use Symfony\AI\Store\Bridge\SurrealDb\Store;

$store = new Store(
    httpClient: $httpClient,
    baseUrl: 'http://localhost:8000',
    namespace: 'my_namespace',
    database: 'my_database',
    tableName: 'documents',
);
```

### ClickHouse

ClickHouse 列式数据库的向量存储，适合大规模数据：

```php
use Symfony\AI\Store\Bridge\ClickHouse\Store;

$store = new Store(
    httpClient: $httpClient,
    baseUrl: 'http://localhost:8123',
    database: 'default',
    tableName: 'embeddings',
);
```

### OpenSearch

OpenSearch（Elasticsearch 的开源分支）向量存储：

```php
use Symfony\AI\Store\Bridge\OpenSearch\Store;

$store = new Store(
    httpClient: $httpClient,
    host: 'https://localhost:9200',
    indexName: 'documents',
);
```

### Typesense

Typesense 搜索引擎的向量存储：

```php
use Symfony\AI\Store\Bridge\Typesense\Store;

$store = new Store(
    httpClient: $httpClient,
    apiKey: 'your-api-key',
    host: 'https://your-cluster.typesense.net',
    collectionName: 'documents',
);
```

### ManticoreSearch

ManticoreSearch 全文搜索引擎的向量存储：

```php
use Symfony\AI\Store\Bridge\ManticoreSearch\Store;

$store = new Store(
    httpClient: $httpClient,
    host: 'http://localhost:9308',
    indexName: 'documents',
);
```

### MariaDB（MariaDB Vector）

MariaDB 向量存储，使用内置向量功能：

```php
use Symfony\AI\Store\Bridge\MariaDb\Store;

$store = new Store(
    pdo: $pdo,
    tableName: 'embeddings',
);
```

### Vektor

Vektor 向量数据库：

```php
use Symfony\AI\Store\Bridge\Vektor\Store;

$store = new Store(
    httpClient: $httpClient,
    baseUrl: 'http://localhost:8080',
    collectionName: 'documents',
);
```

### Cache（Symfony Cache）

使用 Symfony Cache 组件实现的简单向量存储，适合开发和测试：

```php
use Symfony\AI\Store\Bridge\Cache\Store;

$store = new Store(
    cache: $cachePool,
    distanceStrategy: DistanceStrategy::COSINE_DISTANCE,
);
```

---

## CombinedStore（组合存储）

`CombinedStore` 允许同时向多个存储后端写入数据，实现数据冗余或多后端同步：

```php
use Symfony\AI\Store\CombinedStore;

$combined = new CombinedStore(
    stores: [$qdrantStore, $redisStore],
    queryStore: $qdrantStore, // 指定查询时使用的主存储
);

// 写入操作会同步到所有后端
$combined->add($vectorDocuments);

// 查询只在主存储上执行
$results = $combined->query(new VectorQuery($vector));
```

---

## CLI 命令

Store 组件提供以下 CLI 命令：

### SetupStoreCommand

初始化存储基础设施：

```bash
php bin/console ai:store:setup <store-name>
```

### DropStoreCommand

删除存储基础设施：

```bash
php bin/console ai:store:drop <store-name>
```

### IndexCommand

执行文档索引：

```bash
php bin/console ai:index <indexer-name> [source]
```

### RetrieveCommand

测试文档检索：

```bash
php bin/console ai:retrieve <retriever-name> "查询文本"
```

---

## 测试工具

`AbstractStoreIntegrationTestCase` 提供了测试存储实现的基础测试用例：

```php
use Symfony\AI\Store\Test\AbstractStoreIntegrationTestCase;

class MyStoreIntegrationTest extends AbstractStoreIntegrationTestCase
{
    protected function createStore(): StoreInterface
    {
        return new MyStore(/* ... */);
    }

    protected function createVectorDocument(string $id, float ...$values): VectorDocument
    {
        return new VectorDocument($id, new Vector($values));
    }
}
```

---

## RAG 模式实现示例

以下是一个完整的 RAG（检索增强生成）实现示例：

### 1. 搭建索引流水线

```php
use Symfony\AI\Store\Document\Loader\MarkdownLoader;
use Symfony\AI\Store\Document\Transformer\TextTrimTransformer;
use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\SourceIndexer;
use Symfony\AI\Store\Indexer\DocumentProcessor;

// 创建向量化器
$vectorizer = new Vectorizer($platform, 'text-embedding-3-small');

// 创建处理器（过滤 → 转换 → 向量化 → 存储）
$processor = new DocumentProcessor(
    vectorizer: $vectorizer,
    store: $qdrantStore,
    filters: [],
    transformers: [
        new TextTrimTransformer(),
        new TextSplitTransformer(chunkSize: 1000, overlap: 200),
    ],
);

// 创建索引器
$indexer = new SourceIndexer(new MarkdownLoader(), $processor);

// 索引文档
$indexer->index('/path/to/docs/');
```

### 2. 构建检索流水线

```php
use Symfony\AI\Store\Retriever;
use Symfony\AI\Store\EventListener\RerankerListener;
use Symfony\AI\Store\Reranker\Reranker;

// 创建重排序器
$reranker = new Reranker($platform, 'rerank-v3.5');
$dispatcher->addListener(PostQueryEvent::class, new RerankerListener($reranker, topK: 3));

// 创建检索器
$retriever = new Retriever($qdrantStore, $vectorizer, $dispatcher);

// 检索相关文档
$documents = $retriever->retrieve('如何配置向量存储？', ['limit' => 10]);
```

### 3. 将文档注入 Agent 上下文

```php
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 构建包含检索文档的上下文
$context = '';
foreach ($documents as $document) {
    $context .= $document->getMetadata()->getText() . "\n\n";
}

$messages = new MessageBag(
    Message::forSystem(
        "你是一个知识助手。请根据以下上下文回答问题：\n\n{$context}"
    ),
    Message::ofUser('如何配置 Symfony AI 的向量存储？'),
);

$response = $agent->call($messages);
echo $response->getContent();
```

### 4. 完整的 Symfony 服务配置

```yaml
# config/services.yaml
services:
    # 向量化器
    Symfony\AI\Store\Document\Vectorizer:
        arguments:
            $platform: '@Symfony\AI\Platform\PlatformInterface'
            $model: 'text-embedding-3-small'

    # Qdrant 存储
    Symfony\AI\Store\Bridge\Qdrant\Store:
        arguments:
            $httpClient: '@http_client'
            $collectionName: 'documents'
            $embeddingsDimension: 1536

    # 文档处理器
    Symfony\AI\Store\Indexer\DocumentProcessor:
        arguments:
            $vectorizer: '@Symfony\AI\Store\Document\Vectorizer'
            $store: '@Symfony\AI\Store\Bridge\Qdrant\Store'
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
            $store: '@Symfony\AI\Store\Bridge\Qdrant\Store'
            $vectorizer: '@Symfony\AI\Store\Document\Vectorizer'
            $dispatcher: '@event_dispatcher'
```

---

## 异常体系

Store 组件使用项目特定的异常类：

```php
use Symfony\AI\Store\Exception\ExceptionInterface;           // 基础接口
use Symfony\AI\Store\Exception\InvalidArgumentException;     // 无效参数
use Symfony\AI\Store\Exception\RuntimeException;             // 运行时错误
use Symfony\AI\Store\Exception\UnsupportedFeatureException;  // 不支持的功能
use Symfony\AI\Store\Exception\UnsupportedQueryTypeException; // 不支持的查询类型
```

`UnsupportedQueryTypeException` 由 `StoreInterface::query()` 在传入不支持的查询类型时抛出：

```php
try {
    $results = $store->query(new HybridQuery($vector, 'text'));
} catch (UnsupportedQueryTypeException $e) {
    // 该存储不支持混合查询，降级到向量查询
    $results = $store->query(new VectorQuery($vector));
}
```

---

## 桥接器查询支持一览

| 桥接器 | VectorQuery | TextQuery | HybridQuery | ManagedStore |
|--------|:-----------:|:---------:|:-----------:|:------------:|
| Qdrant | ✅ | ❌ | ❌ | ✅ |
| PostgreSQL | ✅ | ✅ | ✅ | ✅ |
| Redis | ✅ | ❌ | ❌ | ✅ |
| Meilisearch | ✅ | ✅ | ❌ | ✅ |
| ChromaDB | ✅ | ✅ | ❌ | ✅ |
| MongoDB | ✅ | ❌ | ❌ | ✅ |
| Elasticsearch | ✅ | ✅ | ✅ | ✅ |
| Pinecone | ✅ | ❌ | ❌ | ✅ |
| Milvus | ✅ | ❌ | ❌ | ✅ |
| Weaviate | ✅ | ❌ | ❌ | ✅ |
| Neo4j | ✅ | ❌ | ❌ | ✅ |
| SQLite | ✅ | ❌ | ❌ | ✅ |
| Supabase | ✅ | ❌ | ❌ | ❌ |
| AzureSearch | ✅ | ✅ | ❌ | ✅ |
| Cloudflare | ✅ | ❌ | ❌ | ✅ |
| S3Vectors | ✅ | ❌ | ❌ | ✅ |
| SurrealDB | ✅ | ❌ | ❌ | ✅ |
| ClickHouse | ✅ | ❌ | ❌ | ✅ |
| OpenSearch | ✅ | ✅ | ✅ | ✅ |
| Typesense | ✅ | ✅ | ❌ | ✅ |
| ManticoreSearch | ✅ | ✅ | ❌ | ✅ |
| MariaDB | ✅ | ❌ | ❌ | ✅ |
| Vektor | ✅ | ❌ | ❌ | ✅ |
| Cache | ✅ | ❌ | ❌ | ✅ |
| InMemory | ✅ | ✅ | ✅ | ✅ |
