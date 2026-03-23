# Store 模块分析报告

> **包名**：`symfony/ai-store`
> **路径**：`src/store/`
> **许可证**：MIT
> **PHP 要求**：>= 8.2

---

## 目录

1. [模块概述](#1-模块概述)
2. [核心接口与输入输出](#2-核心接口与输入输出)
3. [参数不同带来的结果差异（重点章节）](#3-参数不同带来的结果差异重点章节)
4. [实际应用场景](#4-实际应用场景)
5. [文档处理流水线详解](#5-文档处理流水线详解)
6. [存储后端对比（25+ 实现）](#6-存储后端对比25-实现)
7. [Reranker 重排序](#7-reranker-重排序)
8. [事件系统与扩展机制](#8-事件系统与扩展机制)
9. [控制台命令](#9-控制台命令)
10. [异常体系](#10-异常体系)
11. [架构总结与最佳实践](#11-架构总结与最佳实践)

---

## 1. 模块概述

`symfony/ai-store` 是 Symfony AI 单体仓库中的向量数据库抽象层，专为 **RAG（检索增强生成，Retrieval-Augmented Generation）** 应用设计。它提供了统一的接口，支持将文档进行向量化存储，并通过语义搜索、全文检索或混合搜索方式高效检索相关文档，从而在 LLM 生成回答时注入外部知识。

### 1.1 核心价值

- **后端无关性**：通过统一接口 `StoreInterface` 抽象所有向量数据库，业务代码无需关心底层存储实现。
- **完整 RAG 流水线**：内置文档加载、过滤、转换、向量化、存储的完整流水线，并有对应的检索流水线。
- **多查询类型**：支持纯向量查询（`VectorQuery`）、纯文本查询（`TextQuery`）和混合查询（`HybridQuery`），适应不同场景。
- **可扩展设计**：每个环节均有对应接口，可以轻松替换或扩展任意组件。
- **事件驱动**：查询前后均触发事件，支持查询改写、结果重排序等高级功能。

### 1.2 模块依赖

```
symfony/ai-platform   ← 统一 AI 平台接口（调用嵌入模型、重排序模型）
symfony/http-client   ← 远程存储 HTTP 通信
symfony/uid           ← 文档 UUID 生成
symfony/event-dispatcher-contracts ← 事件分发
psr/log               ← 日志
```

### 1.3 整体架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         RAG 应用层                                   │
└──────────────┬──────────────────────────────┬───────────────────────┘
               │                              │
    ┌──────────▼──────────┐       ┌───────────▼──────────┐
    │   IndexerInterface  │       │  RetrieverInterface  │
    │  (文档索引流水线)    │       │  (文档检索流水线)    │
    └──────────┬──────────┘       └───────────┬──────────┘
               │                              │
    ┌──────────▼──────────┐       ┌───────────▼──────────┐
    │  DocumentProcessor  │       │      Retriever        │
    │  filter→transform   │       │  vectorize→query      │
    │  →vectorize→store   │       │  →PreQuery→PostQuery  │
    └──────────┬──────────┘       └───────────┬──────────┘
               │                              │
    ┌──────────▼──────────────────────────────▼──────────┐
    │                   StoreInterface                    │
    │   add() / remove() / query() / supports()           │
    └──────────┬──────────────────────────────────────────┘
               │
    ┌──────────▼──────────────────────────────────────────┐
    │              Bridge 存储后端（25+）                  │
    │  Postgres / Qdrant / Weaviate / Redis / ...          │
    └─────────────────────────────────────────────────────┘
```

---

## 2. 核心接口与输入输出

### 2.1 StoreInterface

位于 `src/StoreInterface.php`，是整个模块的核心契约：

```php
interface StoreInterface
{
    /**
     * 添加向量文档到存储
     * @param VectorDocument|VectorDocument[] $documents
     */
    public function add(VectorDocument|array $documents): void;

    /**
     * 从存储中删除文档
     * @param string|array<string> $ids
     * @param array<string, mixed> $options
     */
    public function remove(string|array $ids, array $options = []): void;

    /**
     * 根据查询检索文档
     * @param array<string, mixed> $options
     * @return iterable<VectorDocument>
     * @throws UnsupportedQueryTypeException 当查询类型不受支持时
     */
    public function query(QueryInterface $query, array $options = []): iterable;

    /**
     * 检查存储是否支持某种查询类型
     * @param class-string<QueryInterface> $queryClass
     */
    public function supports(string $queryClass): bool;
}
```

**关键设计特点**：
- `add()` 只接受已向量化的 `VectorDocument`，原始文档需先经过 `Vectorizer` 处理。
- `query()` 返回 `iterable<VectorDocument>`，支持惰性加载（Generator），避免大结果集的内存压力。
- `supports()` 方法允许运行时查询能力检测，`Retriever` 会据此自动选择最优查询类型。

### 2.2 ManagedStoreInterface

位于 `src/ManagedStoreInterface.php`，扩展 Store 的生命周期管理：

```php
interface ManagedStoreInterface
{
    public function setup(array $options = []): void; // 创建索引/集合/表
    public function drop(array $options = []): void;  // 删除所有数据
}
```

大多数 Bridge 实现（Postgres、Qdrant、Weaviate 等）都同时实现了此接口，支持通过控制台命令 `ai:store:setup` 和 `ai:store:drop` 进行管理。

### 2.3 VectorDocument — 存储与返回的核心数据结构

```php
final class VectorDocument
{
    public function __construct(
        private readonly int|string $id,        // 文档唯一标识
        private readonly VectorInterface $vector, // 嵌入向量（高维浮点数组）
        private readonly Metadata $metadata = new Metadata(), // 元数据键值对
        private readonly ?float $score = null,  // 查询相似度分数（存储时为 null）
    ) {}

    public function withScore(float $score): self; // 不可变更新分数
}
```

**score 分数的含义**：

score 的含义取决于所使用的距离度量和后端实现：

| 距离策略 | score 范围 | 含义 | score 越高 |
|---------|-----------|------|-----------|
| 余弦距离（Cosine Distance） | [0.0, 2.0] | 1 - 余弦相似度 | 越不相似 |
| 余弦相似度（Cosine Similarity） | [-1.0, 1.0] | 向量方向夹角余弦 | 越相似 |
| 欧氏距离（L2） | [0.0, +∞) | 空间中两点直线距离 | 越不相似 |
| 内积（Inner Product） | (-∞, +∞) | 点积（归一化向量等价余弦） | 越相似 |
| RRF 分数（CombinedStore） | (0.0, 1.0] | 互易排名融合分数 | 越相关 |

> **注意**：InMemory Store 使用余弦**距离**（越小越相似），而 Qdrant、Weaviate 等外部存储通常返回余弦**相似度**（越大越相似）。使用时需结合具体后端文档确认。

### 2.4 TextDocument — 待索引的原始文档

```php
final class TextDocument implements EmbeddableDocumentInterface
{
    public function __construct(
        private readonly int|string $id,     // 唯一标识（可由 Uuid::v4() 生成）
        private readonly string $content,    // 文本内容（不能为空字符串）
        private readonly Metadata $metadata = new Metadata(),
    ) {}
}
```

`TextDocument` 是索引流水线的**输入**，经过 `Vectorizer` 后变为 `VectorDocument`，才能存入 Store。

### 2.5 Metadata — 元数据体系

`Metadata` 继承自 `ArrayObject`，支持任意键值，同时定义了一组具有特殊语义的保留键：

```php
final class Metadata extends \ArrayObject
{
    const KEY_PARENT_ID = '_parent_id'; // 分块后关联原始文档 ID
    const KEY_TEXT      = '_text';      // 原始文本（向量化后保留，供检索时展示）
    const KEY_SOURCE    = '_source';    // 来源（文件路径、URL 等）
    const KEY_SUMMARY   = '_summary';   // AI 生成的摘要
    const KEY_TITLE     = '_title';     // 文档标题
    const KEY_DEPTH     = '_depth';     // 文档层级深度（如章节嵌套）
}
```

**Metadata 对过滤能力的影响示例**：

```php
// 场景：企业知识库按部门过滤
$metadata = new Metadata([
    Metadata::KEY_SOURCE   => 'hr/policy-2024.pdf',
    Metadata::KEY_TITLE    => '2024年员工手册',
    'department'           => 'HR',
    'doc_type'             => 'policy',
    'year'                 => 2024,
    'access_level'         => 'all',
]);

// InMemory Store 使用 filter 回调进行元数据过滤
$documents = $store->query($vectorQuery, [
    'filter' => fn(VectorDocument $doc): bool =>
        $doc->getMetadata()['department'] === 'HR'
        && $doc->getMetadata()['year'] >= 2023,
    'maxItems' => 10,
]);
```

---

## 3. 参数不同带来的结果差异（重点章节）

### 3.1 topK / maxItems 参数

`topK` 或 `maxItems` 控制返回的文档数量。在 `StoreInterface::query()` 中通过 `$options` 传递，在 `RerankerInterface::rerank()` 中作为显式参数。

```php
// 精准场景：直接返回 3 个最相关结果
$store->query($vectorQuery, ['maxItems' => 3]);

// Reranker 场景：先取 50 个候选，重排序后返回 5 个
$candidates = $store->query($vectorQuery, ['maxItems' => 50]);
$reranked   = $reranker->rerank($queryText, iterator_to_array($candidates), topK: 5);
```

| topK 值 | 适用场景 | 优点 | 缺点 |
|--------|---------|------|------|
| 1-3 | 明确问答、代码定位 | 精准、响应快 | 可能遗漏次优答案 |
| 5-10 | 通用 RAG 问答 | 覆盖充分、综合质量高 | 上下文窗口占用较多 |
| 20-50 | 配合 Reranker 使用 | 召回率高 | 需要额外重排序成本 |
| 100+ | 数据分析、去重检测 | 全面覆盖 | 性能开销大，LLM 上下文装不下 |

**最佳实践**：在生产 RAG 应用中，推荐 `maxItems=20`（粗召回）+ `Reranker topK=5`（精排序）的两阶段策略。

### 3.2 距离度量参数

#### 3.2.1 InMemory Store 的 DistanceStrategy

`InMemory\Store` 使用 `DistanceCalculator`，支持 5 种距离策略：

```php
enum DistanceStrategy: string
{
    case COSINE_DISTANCE   = 'cosine';     // 余弦距离（默认）
    case ANGULAR_DISTANCE  = 'angular';    // 角距离（弧度）
    case EUCLIDEAN_DISTANCE = 'euclidean'; // 欧氏距离
    case MANHATTAN_DISTANCE = 'manhattan'; // 曼哈顿距离
    case CHEBYSHEV_DISTANCE = 'chebyshev'; // 切比雪夫距离
}

// 使用欧氏距离创建 InMemory Store
$store = new InMemory\Store(
    new DistanceCalculator(DistanceStrategy::EUCLIDEAN_DISTANCE, batchSize: 100)
);
```

#### 3.2.2 Postgres Bridge 的 Distance

```php
enum Distance: string
{
    case Cosine       = 'cosine';        // <=> 运算符
    case InnerProduct = 'inner_product'; // <#> 运算符
    case L1           = 'l1';            // <+> 运算符（曼哈顿）
    case L2           = 'l2';            // <-> 运算符（欧氏）
}
```

#### 3.2.3 Redis Bridge 的 Distance

```php
enum Distance: string
{
    case Cosine = 'COSINE'; // 余弦相似度
    case L2     = 'L2';     // 欧氏距离
    case Ip     = 'IP';     // 内积
}
```

#### 3.2.4 距离度量对比表

| 距离度量 | 数学公式 | 适用场景 | 不适用场景 | 向量是否需要归一化 |
|--------|---------|---------|----------|---------------|
| 余弦距离 | `1 - (A·B)/(‖A‖‖B‖)` | 文本语义相似度（最常用） | 需要考虑量级差异时 | 不需要 |
| 欧氏距离（L2） | `√Σ(aᵢ-bᵢ)²` | 图像特征、数值特征 | 高维稀疏向量 | 不需要 |
| 内积（IP） | `Σ(aᵢ·bᵢ)` | 归一化向量等效余弦，推荐系统 | 未归一化向量 | 需要归一化 |
| 曼哈顿（L1） | `Σ|aᵢ-bᵢ|` | 对异常值鲁棒、稀疏特征 | 要求旋转不变性时 | 不需要 |
| 切比雪夫 | `max|aᵢ-bᵢ|` | 棋盘游戏距离、最大偏差敏感 | 语义相似度 | 不需要 |

**实际选择建议**：
- 使用 OpenAI 嵌入模型（text-embedding-ada-002 / text-embedding-3-*）：推荐**余弦相似度**，这是其训练优化的目标。
- 使用 CLIP 图像嵌入：推荐**内积**（模型输出已归一化）。
- 需要考虑向量量级的场景（如用户行为强度）：推荐**欧氏距离**。

### 3.3 semanticRatio / alpha 参数（HybridQuery）

`HybridQuery` 中的 `semanticRatio` 参数（取值 0.0～1.0）控制语义搜索与关键词搜索的权重比例：

```php
// 纯语义搜索
$query = new HybridQuery($vector, $texts, semanticRatio: 1.0);

// 平衡搜索（默认）
$query = new HybridQuery($vector, $texts, semanticRatio: 0.5);

// 纯关键词搜索
$query = new HybridQuery($vector, $texts, semanticRatio: 0.0);
```

通过 `Retriever` 传递 `semanticRatio` 选项：

```php
$retriever->retrieve('深度学习框架', options: ['semanticRatio' => 0.7]);
```

| semanticRatio | 语义权重 | 关键词权重 | 效果描述 | 适用场景 |
|--------------|--------|----------|---------|---------|
| 0.0 | 0% | 100% | 精确关键词匹配，无语义理解 | 代码搜索、型号搜索、精确词查询 |
| 0.3 | 30% | 70% | 以关键词为主，兼顾语义扩展 | 技术文档、产品规格搜索 |
| 0.5 | 50% | 50% | 完全平衡，通用场景 | 通用 FAQ、知识库 |
| 0.7 | 70% | 30% | 以语义为主，允许同义词匹配 | 企业知识库、自然语言问答 |
| 1.0 | 100% | 0% | 纯语义，理解意图但可能丢失精确关键词 | 推荐系统、意图理解 |

**`CombinedStore` 的 RRF（互易排名融合）**：

`CombinedStore` 将 `HybridQuery` 分解为独立的向量查询和文本查询，分别发给不同的子存储，最后用 RRF 算法合并排名：

```
RRF_score(doc) = Σ 1 / (k + rank_i)
```

其中 `k=60`（可配置），`rank_i` 是文档在第 i 个结果列表中的排名（从 0 开始）。k 值越大，头部结果优势越小，排名更加均衡。

### 3.4 嵌入向量维度

嵌入维度由所选模型决定，对存储和搜索精度均有显著影响：

| 模型 | 维度 | 存储（每文档） | 搜索速度 | 语义精度 | 适用场景 |
|-----|-----|------------|--------|--------|---------|
| text-embedding-ada-002 | 1536 | ~6KB | 快 | 中等 | 通用 RAG，成本敏感 |
| text-embedding-3-small | 1536 | ~6KB | 快 | 中高 | 通用 RAG，替代 ada-002 |
| text-embedding-3-large | 3072 | ~12KB | 较慢 | 高 | 高精度检索，预算充足 |
| nomic-embed-text | 768 | ~3KB | 很快 | 中等 | 本地部署，Ollama |
| mxbai-embed-large | 1024 | ~4KB | 快 | 中高 | 本地高精度 |

**维度对存储空间的影响估算**（100 万文档）：

```
1536 维 × 4 字节/float × 1,000,000 文档 = ~6 GB（纯向量数据）
3072 维 × 4 字节/float × 1,000,000 文档 = ~12 GB（纯向量数据）
```

### 3.5 文档分块策略（TextSplitTransformer）

`TextSplitTransformer` 将长文档切割为适合嵌入模型的小块：

```php
final class TextSplitTransformer implements TransformerInterface
{
    public function __construct(
        private readonly int $chunkSize = 1000, // 每块最大字符数
        private readonly int $overlap = 200,    // 相邻块的重叠字符数
    ) {}
}
```

块大小与重叠对 RAG 效果的影响：

| chunkSize | overlap | 效果描述 | 适用场景 |
|----------|---------|---------|---------|
| 256 | 32 | 粒度细，精确定位短句 | 代码注释、短问答知识库 |
| 500 | 50 | 平衡粒度，适合段落级检索 | 通用 FAQ、Wiki |
| 1000 | 200 | 保留更多上下文（默认值） | 技术文档、长篇文章 |
| 2000 | 400 | 粗粒度，丢失细节 | 概念性搜索、摘要生成 |

**overlap（重叠）的作用**：

防止关键信息被切割在两块的边界处。例如，`chunkSize=1000, overlap=200` 时，相邻块共享 200 字符，确保跨块的句子上下文完整性：

```
块 1: [0    ......    1000]
块 2:          [800  ........  1800]
块 3:                   [1600 ........ 2600]
```

**对 RAG 检索精度的影响**：
- 块太小（< 200）：向量化信息不足，语义表达不完整，召回率下降。
- 块太大（> 2000）：一个块包含多个主题，向量被"稀释"，精确度下降。
- overlap 为 0：边界切割风险高，句子可能被截断，影响语义完整性。
- overlap 过大（> chunkSize/2）：大量冗余，存储空间浪费，检索速度下降。

---

## 4. 实际应用场景

### 4.1 企业知识库（参考 Notion AI、Confluence AI）

**目标**：员工可用自然语言查询公司内部文档、政策、流程。

**索引流程**：

```php
use Symfony\AI\Store\Document\Loader\MarkdownLoader;
use Symfony\AI\Store\Document\Transformer\{TextSplitTransformer, TextTrimTransformer, ChainTransformer};
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\SourceIndexer;
use Symfony\AI\Store\Indexer\DocumentProcessor;

$processor = new DocumentProcessor(
    vectorizer: new Vectorizer($platform, 'text-embedding-3-small'),
    store: $store,
    transformers: [
        new TextTrimTransformer(),
        new TextSplitTransformer(chunkSize: 800, overlap: 150),
    ],
);

$indexer = new SourceIndexer(new MarkdownLoader(), $processor);
$indexer->index('/var/knowledge-base/hr/');
```

**查询流程**：

```php
$retriever = new Retriever(
    store: $postgresStore,
    vectorizer: new Vectorizer($platform, 'text-embedding-3-small'),
    eventDispatcher: $dispatcher,
);

// HybridQuery，语义优先（semanticRatio=0.7）
$docs = $retriever->retrieve(
    '员工年假政策是什么？',
    options: ['semanticRatio' => 0.7, 'maxItems' => 10]
);
```

**metadata 过滤示例**：

```php
// 通过 InMemory Store 的 filter 选项过滤特定部门文档
$docs = $store->query($vectorQuery, [
    'maxItems' => 10,
    'filter' => fn(VectorDocument $doc): bool =>
        $doc->getMetadata()['department'] === 'HR',
]);
```

**推荐配置**：
- 存储后端：Postgres（pgvector）或 Qdrant
- 查询类型：`HybridQuery`，`semanticRatio=0.7`
- 块大小：800 字符，重叠 150 字符
- topK：10 个粗召回 + Reranker 精排到 3 个

---

### 4.2 代码文档搜索（参考 Sourcegraph Cody）

**目标**：开发者通过自然语言查找函数定义、注释、API 用法。

**特殊要求**：
- 代码关键词精确匹配（函数名、类名）同样重要
- 需要记录 `language`、`filename`、`function_name` 等元数据

```php
// 加载 PHP 源码文件（每个文件一个 TextDocument）
$loader = new TextFileLoader();

// 自定义分块：按函数边界分块，chunkSize=500 保持函数完整性
$processor = new DocumentProcessor(
    vectorizer: $vectorizer,
    store: $store,
    transformers: [new TextSplitTransformer(chunkSize: 500, overlap: 80)],
);

// 代码搜索使用 HybridQuery，关键词权重更高
$query = new HybridQuery($vector, ['UserRepository', 'findByEmail'], semanticRatio: 0.4);
$results = $store->query($query, ['maxItems' => 10]);
```

**推荐配置**：
- 查询类型：`HybridQuery`，`semanticRatio=0.3~0.5`（关键词权重更高）
- 块大小：300~500 字符（保持函数完整）
- Metadata：`language`, `filename`, `class_name`, `method_name`

---

### 4.3 电商商品搜索（参考 Algolia Neural Search）

**目标**：用户搜索商品，支持语义理解（"适合夏天穿的鞋子"）和精确过滤（brand=Nike, size=42）。

```php
// 商品文档结构
$metadata = new Metadata([
    'category'    => 'shoes',
    'brand'       => 'Nike',
    'price'       => 599.0,
    'in_stock'    => true,
    'size_range'  => [38, 39, 40, 41, 42, 43],
]);

$product = new TextDocument(
    id: 'prod-001',
    content: 'Nike Air Max 270 轻量跑鞋，透气网面，适合夏季日常穿着',
    metadata: $metadata,
);

// 混合查询：语义理解 + 关键词匹配，同时过滤库存
$results = $store->query(
    new HybridQuery($vector, ['夏天', '凉鞋', '透气'], semanticRatio: 0.5),
    options: [
        'maxItems' => 20,
        'filter' => fn(VectorDocument $d): bool =>
            $d->getMetadata()['in_stock'] === true
            && $d->getMetadata()['price'] <= 1000.0,
    ]
);
```

**推荐配置**：
- 查询类型：`HybridQuery`，`semanticRatio=0.5`（平衡）
- 块大小：商品描述通常较短，无需分块
- Metadata：`category`, `brand`, `price`, `in_stock`, `size_range`

---

### 4.4 医疗文献检索（参考 PubMed AI、Semantic Scholar）

**目标**：研究人员搜索与研究主题相关的医学论文，需要高精度两阶段检索。

```php
// 两阶段检索：粗召回 50 篇 + Reranker 精排到 10 篇
$candidates = iterator_to_array(
    $retriever->retrieve('COVID-19 mRNA 疫苗免疫应答机制', ['maxItems' => 50])
);

$reranker = new Reranker($platform, 'cohere-rerank-v3');
$topDocs  = $reranker->rerank('COVID-19 mRNA 疫苗免疫应答机制', $candidates, topK: 10);

// topDocs 的 score 为交叉编码器分数，比双编码器相似度更准确
foreach ($topDocs as $doc) {
    echo $doc->getMetadata()->getTitle() . ' (score: ' . $doc->getScore() . ')' . PHP_EOL;
}
```

**推荐配置**：
- 查询类型：`VectorQuery`（语义优先）
- topK：50 粗召回 → Reranker → 10 精排
- 嵌入模型：`text-embedding-3-large`（高精度）
- Metadata：`pmid`, `journal`, `year`, `authors`, `mesh_terms`

---

### 4.5 法律条文检索（参考 Harvey AI、CaseText）

**目标**：律师搜索特定司法管辖区的相关法条，要求精确条文引用。

```php
// 法律文档分块：保持条款完整，chunkSize=1000
$processor = new DocumentProcessor(
    vectorizer: $vectorizer,
    store: $store,
    transformers: [new TextSplitTransformer(chunkSize: 1000, overlap: 100)],
);

// 元数据结构
$metadata = new Metadata([
    'jurisdiction' => 'CN',      // 司法管辖区
    'law_code'     => '劳动法',
    'chapter'      => '第三章',
    'article'      => '第二十一条',
    'year'         => 1994,
    'effective'    => true,
]);

// 精确过滤中国劳动法 + 语义搜索
$docs = $store->query(
    new VectorQuery($vector),
    options: [
        'maxItems' => 20,
        'filter' => fn(VectorDocument $d): bool =>
            $d->getMetadata()['jurisdiction'] === 'CN'
            && $d->getMetadata()['effective'] === true,
    ]
);
```

**推荐配置**：
- 查询类型：`VectorQuery` + 元数据过滤
- 块大小：1000 字符（保持完整条款）
- Reranker：是（法律精度要求高）

---

### 4.6 多语言客服知识库（参考 Zendesk AI）

**目标**：支持多语言用户提问，从统一知识库中检索答案，跨语言语义对齐。

```php
// 使用多语言嵌入模型，相同语义的不同语言文本在向量空间相近
// 如：multilingual-e5-large、text-multilingual-embedding-002

$vectorizer = new Vectorizer($platform, 'multilingual-e5-large');

// 索引中文和英文文档
$chineseDoc = new TextDocument('faq-001-zh', '如何申请退款？提交工单后3-5个工作日处理。', new Metadata([
    'language' => 'zh',
    'product' => 'order',
    'category' => 'refund',
]));

$englishDoc = new TextDocument('faq-001-en', 'How to apply for a refund? Submit a ticket and we\'ll process within 3-5 business days.', new Metadata([
    'language' => 'en',
    'product' => 'order',
    'category' => 'refund',
]));

// 用中文查询，也能召回英文文档（跨语言检索）
$docs = $retriever->retrieve('退款怎么处理', ['maxItems' => 5]);
```

**推荐配置**：
- 嵌入模型：多语言模型（multilingual-e5、paraphrase-multilingual）
- 查询类型：`VectorQuery` 或 `HybridQuery`（semanticRatio=0.8）
- Metadata：`language`, `product`, `category`

---

### 4.7 新闻聚合与去重（参考 Feedly）

**目标**：自动检测新文章是否与已有文章高度相似，去除重复内容。

```php
// 使用 RssFeedLoader 加载新闻 RSS
$loader = new RssFeedLoader($httpClient);

// 新文章索引前检测相似性
function isNewArticle(string $title, StoreInterface $store, Vectorizer $vectorizer): bool
{
    $vector = $vectorizer->vectorize($title);
    $similar = iterator_to_array($store->query(new VectorQuery($vector), ['maxItems' => 1]));

    if ([] === $similar) {
        return true;
    }

    // 余弦距离 < 0.05 视为重复（非常相似）
    return $similar[0]->getScore() > 0.05;
}
```

**推荐配置**：
- 查询类型：`VectorQuery`，`maxItems=1`（只取最相似的一篇）
- 相似度阈值：余弦距离 < 0.05 视为重复（InMemory Store）
- 块大小：不分块（新闻标题/摘要本身较短）

---

### 4.8 RAG 聊天机器人完整示例（参考 ChatPDF、Perplexity）

这是最典型的 RAG 场景，展示从文档加载到问答的完整流程：

```php
// === 索引阶段（一次性）===

// 1. 加载 PDF/Markdown 文档
$loader     = new MarkdownLoader();
$documents  = $loader->load('/path/to/documentation/');

// 2. 文档转换流水线
$transformer = new ChainTransformer([
    new TextTrimTransformer(),                          // 清理空白
    new TextSplitTransformer(chunkSize: 800, overlap: 150), // 分块
]);

// 3. 可选：AI 生成摘要并双索引
// $transformer = new ChainTransformer([..., new SummaryGeneratorTransformer($platform, 'gpt-4o-mini', yieldSummaryDocuments: true)]);

// 4. 向量化并存储
$processor = new DocumentProcessor(
    vectorizer: new Vectorizer($platform, 'text-embedding-3-small'),
    store: $qdrantStore,
    transformers: [$transformer],
);
$processor->process($documents);

// === 查询阶段（每次对话）===

// 5. 检索相关文档
$retriever = new Retriever($qdrantStore, $vectorizer, $eventDispatcher);
$docs = $retriever->retrieve($userQuestion, ['maxItems' => 10]);

// 6. 构建 RAG Prompt
$context = implode("\n\n", array_map(
    fn(VectorDocument $d): string => $d->getMetadata()->getText() ?? '',
    iterator_to_array($docs)
));

$prompt = "根据以下文档内容回答问题：\n\n{$context}\n\n问题：{$userQuestion}";

// 7. 调用 LLM 生成答案
$answer = $platform->invoke('gpt-4o', Message::ofUser($prompt))->asText();
```

---

### 4.9 个性化推荐系统（参考 Netflix、Spotify）

```php
// 用户行为向量化：将用户历史操作聚合为向量
$userProfile = averageVectors(
    array_map(
        fn(string $itemId) => $itemStore->getVector($itemId),
        $user->getViewHistory()
    )
);

// 基于用户画像向量查找相似物品
$recommended = $itemStore->query(
    new VectorQuery(new Vector($userProfile)),
    options: [
        'maxItems' => 20,
        'filter' => fn(VectorDocument $d): bool =>
            !in_array($d->getId(), $user->getViewHistory()), // 过滤已看过的
    ]
);
```

---

### 4.10 学术论文推荐（参考 Semantic Scholar）

```php
// 基于已读论文列表，找相似论文
$readPaperVectors = array_map(
    fn(string $paperId) => $paperStore->getVector($paperId),
    $user->getReadPapers()
);

// 取平均向量作为用户学术兴趣画像
$interestVector = new Vector(
    array_map(
        fn(int $i) => array_sum(array_column($readPaperVectors, $i)) / count($readPaperVectors),
        range(0, 1535) // 1536 维
    )
);

$suggestions = $paperStore->query(
    new VectorQuery($interestVector),
    ['maxItems' => 10]
);
```

---

## 5. 文档处理流水线详解

### 5.1 完整流水线示意

```
数据源（文件/URL/数据库）
    ↓
LoaderInterface::load()
    ↓ iterable<TextDocument>
FilterInterface::filter()       ← 过滤不符合要求的文档（如内容太短）
    ↓ iterable<TextDocument>
TransformerInterface::transform() ← 转换/分块/清理/生成摘要
    ↓ iterable<TextDocument>
VectorizerInterface::vectorize()  ← 调用嵌入模型生成向量
    ↓ array<VectorDocument>
StoreInterface::add()             ← 存入向量数据库
```

### 5.2 加载器（Loader）详解

| 加载器 | 来源 | 输出 | 特殊选项 |
|-------|------|------|---------|
| `TextFileLoader` | 本地文本文件 | 1 个 TextDocument/文件 | 无 |
| `MarkdownLoader` | `.md` 文件 | 1 个 TextDocument/文件 | `strip_formatting: bool` |
| `CsvLoader` | CSV 文件 | 每行 1 个 TextDocument | `content_column`, `id_column`, `metadata_columns`, `delimiter` |
| `JsonFileLoader` | JSON 文件 | 每条记录 1 个 TextDocument | `id`(JsonPath), `content`(JsonPath), `metadata`(JsonPath 映射) |
| `RssFeedLoader` | RSS URL | 每条新闻 1 个 TextDocument | `uuid_namespace` |
| `RstLoader` | RST 文件 | 1 个 TextDocument/文件 | — |
| `RstToctreeLoader` | RST 目录树 | 多个 TextDocument | — |
| `InMemoryLoader` | 内存中的文档 | 传入的文档 | `source: null` |

**CsvLoader 高级用法**：

```php
// 从数据库导出的 CSV 加载产品数据
$loader = new CsvLoader(
    contentColumn: 'description',  // 哪列作为文档内容
    idColumn: 'product_id',        // 哪列作为文档 ID
    metadataColumns: ['category', 'price', 'brand'], // 哪些列存入 metadata
    delimiter: ',',
    hasHeader: true,
);
foreach ($loader->load('/export/products.csv') as $doc) {
    // $doc->getMetadata()['category'], ['price'], ['brand'] 均可用
}
```

**JsonFileLoader 使用 JsonPath**：

```php
// 从 API 响应的 JSON 文件加载
$loader = new JsonFileLoader(
    id: '$.articles[*].id',                   // 文章 ID 的 JsonPath
    content: '$.articles[*].body',             // 文章正文的 JsonPath
    metadata: [
        'title'  => '$.articles[*].title',
        'author' => '$.articles[*].author.name',
        'date'   => '$.articles[*].published_at',
    ]
);
```

### 5.3 转换器（Transformer）详解

#### TextSplitTransformer — 核心分块器

```php
$splitter = new TextSplitTransformer(chunkSize: 1000, overlap: 200);
// 产生的每个 chunk 的 metadata 包含：
// - _parent_id: 原始文档 ID
// - _text: 块内容
// - 以及所有原始文档的 metadata（继承）
```

#### TextTrimTransformer — 清理空白

```php
$trimmer = new TextTrimTransformer();
// 对所有文档执行 trim()，去除首尾空白
// 适合：从 HTML 提取文本后清理多余空行
```

#### TextReplaceTransformer — 文本替换

```php
// 清理模板占位符或特殊格式
$replacer = new TextReplaceTransformer(search: '{{INTERNAL}}', replace: '');

// 支持运行时覆盖
$transformer->transform($docs, [
    'search' => '<!-- comment -->',
    'replace' => '',
]);
```

#### SummaryGeneratorTransformer — AI 摘要生成（双索引）

```php
$summarizer = new SummaryGeneratorTransformer(
    platform: $platform,
    model: 'gpt-4o-mini',
    yieldSummaryDocuments: true,  // 同时索引摘要文档（双索引）
    systemPrompt: '用 2-3 句话概括以下文本的核心内容和关键技术术语。',
);
// 效果：原文档 + 摘要文档都进入索引
// 好处：摘要向量化更聚焦于主题，可提升检索精度
```

**双索引（Dual Indexing）策略**：

同时存储原文块的向量和 AI 摘要的向量，查询时优先命中摘要向量（更聚焦），再通过 `_parent_id` 取回完整原文作为 LLM 上下文。

#### ChunkDelayTransformer — 速率限制保护

```php
// 处理大量文档时避免触发 API 速率限制
$delayer = new ChunkDelayTransformer(
    clock: new SystemClock(),
    chunkSize: 50,  // 每 50 个文档后暂停
    delay: 10,      // 暂停 10 秒
);
```

#### ChainTransformer — 流水线组合

```php
$pipeline = new ChainTransformer([
    new TextTrimTransformer(),
    new TextReplaceTransformer('{{draft}}', ''),
    new TextSplitTransformer(chunkSize: 800, overlap: 100),
    new ChunkDelayTransformer($clock, chunkSize: 100, delay: 5),
]);
```

### 5.4 过滤器（Filter）

#### TextContainsFilter — 关键词过滤

```php
// 过滤掉包含"草稿"字样的文档（不索引草稿）
$filter = new TextContainsFilter(needle: '草稿', caseSensitive: false);

// 运行时覆盖
$filter->filter($documents, ['needle' => 'TODO', 'case_sensitive' => true]);
```

### 5.5 向量化器（Vectorizer）

```php
final class Vectorizer implements VectorizerInterface
{
    // 支持批量向量化（利用模型的 INPUT_MULTIPLE 能力）
    // 单个字符串 → Vector
    public function vectorize(string $text): Vector;

    // 单个文档 → VectorDocument（自动将原文保存到 metadata._text）
    public function vectorize(EmbeddableDocumentInterface $doc): VectorDocument;

    // 批量文档 → VectorDocument[]（自动检测是否支持批量 API）
    public function vectorize(array $docs): array;
}
```

**批量 vs 逐条向量化**：

`Vectorizer` 会自动检测嵌入模型是否支持 `Capability::INPUT_MULTIPLE`（批量输入）。若支持，则一次 API 调用处理整批文档，大幅降低延迟和成本（OpenAI text-embedding 系列均支持批量）。

---

## 6. 存储后端对比（25+ 实现）

### 6.1 查询类型支持矩阵

| 存储后端 | VectorQuery | TextQuery | HybridQuery | ManagedStore | 部署方式 |
|--------|:-----------:|:---------:|:-----------:|:------------:|---------|
| **InMemory** | ✅ | ✅ | ✅ | ✅ | 进程内存 |
| **Postgres** (pgvector) | ✅ | ✅ | ✅ | ✅ | 自托管/云 |
| **Sqlite** | ✅ | ✅ | ✅ | ✅ | 本地文件 |
| **Cache** (PSR-6) | ✅ | ✅ | ✅ | ✅ | 各种缓存 |
| **ChromaDb** | ✅ | ✅ | ❌ | ✅ | 自托管 |
| **Meilisearch** | ✅ | ❌ | ❌ | ✅ | 自托管/云 |
| **Elasticsearch** | ✅ | ❌ | ❌ | ✅ | 自托管/云 |
| **OpenSearch** | ✅ | ❌ | ❌ | ✅ | 自托管/云 |
| **ManticoreSearch** | ✅ | ❌ | ❌ | ✅ | 自托管 |
| **Qdrant** | ✅ | ❌ | ❌ | ✅ | 自托管/云 |
| **Weaviate** | ✅ | ❌ | ❌ | ✅ | 自托管/云 |
| **Milvus** | ✅ | ❌ | ❌ | ✅ | 自托管/云 |
| **Redis** (RedisSearch) | ✅ | ❌ | ❌ | ✅ | 自托管/云 |
| **MongoDb** (Atlas) | ✅ | ❌ | ❌ | ✅ | 云服务 |
| **Pinecone** | ✅ | ❌ | ❌ | ✅ | 云服务 |
| **Supabase** | ✅ | ❌ | ❌ | ❌ | 云服务 |
| **Cloudflare** | ✅ | ❌ | ❌ | ✅ | 边缘云 |
| **S3Vectors** | ✅ | ❌ | ❌ | ✅ | AWS 云 |
| **Neo4j** | ✅ | ❌ | ❌ | ✅ | 自托管/云 |
| **SurrealDb** | ✅ | ❌ | ❌ | ✅ | 自托管 |
| **ClickHouse** | ✅ | ❌ | ❌ | ✅ | 自托管/云 |
| **Typesense** | ✅ | ❌ | ❌ | ✅ | 自托管/云 |
| **MariaDb** | ✅ | ❌ | ❌ | ✅ | 自托管/云 |
| **Vektor** | ✅ | ❌ | ❌ | ✅ | 自托管 |
| **AzureSearch** | ✅ | ❌ | ❌ | ❌ | Azure 云 |
| **CombinedStore** | ✅ | ✅ | ✅（RRF）| ❌ | 组合 |

> **注意**：仅 InMemory、Postgres、Sqlite、Cache 原生支持 HybridQuery。其他存储若需混合查询，可使用 `CombinedStore` 将向量存储与文本存储组合，通过 RRF 算法融合结果。

### 6.2 各后端特点详解

#### InMemory Store

```php
use Symfony\AI\Store\InMemory\Store;
use Symfony\AI\Store\Distance\{DistanceCalculator, DistanceStrategy};

$store = new Store(
    new DistanceCalculator(DistanceStrategy::COSINE_DISTANCE, batchSize: 100)
);
```

- **优点**：零依赖、零配置，适合单元测试、原型开发
- **缺点**：数据不持久化，内存限制，不适合生产大数据量
- **特殊能力**：支持 `callable` 过滤器，最灵活的运行时过滤
- **适用场景**：测试、开发环境、数据量 < 10 万条

#### Postgres (pgvector)

```php
use Symfony\AI\Store\Bridge\Postgres\{Store, Distance};

$store = new Store(
    connection: $pdo,
    distance: Distance::Cosine,     // 余弦距离（默认）
    tableName: 'documents',
    vectorDimension: 1536,
);
```

- **优点**：基于成熟 PostgreSQL，ACID 事务，原生支持 SQL 复杂过滤
- **原生 Hybrid**：支持 `VectorQuery + TextQuery + HybridQuery`（pgvector + pg_trgm）
- **距离支持**：Cosine、InnerProduct、L1、L2（通过 `<=>`, `<#>`, `<+>`, `<->` 运算符）
- **适用场景**：已有 Postgres 基础设施的企业、中小规模 RAG（< 500 万文档）

#### Qdrant

- **优点**：专为向量搜索设计，原生过滤能力强，支持 payload 过滤
- **部署**：REST API，支持 Docker 本地部署 + Qdrant Cloud
- **特殊功能**：支持命名向量（Named Vectors）、多向量、量化
- **适用场景**：高性能向量搜索，复杂元数据过滤

#### Weaviate

- **优点**：内置多模态支持，Schema 定义清晰，支持 GraphQL
- **缺点**：配置相对复杂
- **适用场景**：多模态 RAG（文本+图像）、需要图结构关系的场景

#### ChromaDb

```php
use Symfony\AI\Store\Bridge\ChromaDb\Store;
// 依赖：codewithkyrian/chromadb-php
```

- **优点**：Python 生态最流行的向量数据库，适合与 Python AI 栈混合使用
- **支持**：`VectorQuery` + `TextQuery`（通过 `queryTexts` 内部向量化）
- **适用场景**：LangChain 迁移项目、Python/PHP 混合团队

#### Redis (RedisSearch)

```php
use Symfony\AI\Store\Bridge\Redis\{Store, Distance};
// 依赖：ext-redis

$store = new Store(
    redis: $redis,
    indexName: 'documents',
    distance: Distance::Cosine,
    vectorDimension: 1536,
);
```

- **优点**：超低延迟（< 1ms），已有 Redis 基础设施可直接复用
- **缺点**：内存成本高，持久化需要 RDB/AOF 配置
- **适用场景**：实时推荐、低延迟聊天机器人、缓存层向量搜索

#### Sqlite

- **优点**：无服务器、本地文件，适合桌面应用和轻量级部署
- **原生 Hybrid**：支持 FTS5 全文索引
- **适用场景**：桌面工具、离线 AI 助手、边缘设备

#### Cache Store

```php
use Symfony\AI\Store\Bridge\Cache\Store;
// 基于 PSR-6 CacheItemPoolInterface

$store = new Store($cachePool, ttl: 3600);
```

- **优点**：利用现有缓存基础设施（Redis/Memcached/File），自动 TTL 过期
- **支持**：`VectorQuery + TextQuery + HybridQuery`
- **适用场景**：短期向量缓存、API 结果缓存、会话级检索

#### Elasticsearch / OpenSearch

- **优点**：企业级搜索基础设施，丰富的分析功能，大规模文本索引
- **适用场景**：已有 ELK 栈的企业、全文搜索+向量搜索混合（CombinedStore）

#### Milvus

- **优点**：专业向量数据库，千亿级向量支持，GPU 加速
- **适用场景**：超大规模向量检索（> 1 亿条）

#### Pinecone

- **优点**：全托管云服务，零运维，自动扩展
- **缺点**：成本较高，数据在云端
- **适用场景**：快速原型、中小团队不想运维数据库

#### S3Vectors

- **优点**：利用 AWS S3 存储向量，成本极低，适合冷数据
- **适用场景**：归档场景、低频查询的大规模向量存储

#### AzureSearch（注意：类名为 `SearchStore`）

```php
use Symfony\AI\Store\Bridge\AzureSearch\SearchStore;

$store = new SearchStore(
    httpClient: $client,
    endpointUrl: 'https://xxx.search.windows.net',
    apiKey: '...',
    indexName: 'documents',
    apiVersion: '2024-05-01-preview',
    vectorFieldName: 'vector',
);
```

- **特点**：Azure 云服务，与 Azure OpenAI 天然集成
- **注意**：`SearchStore` 不实现 `ManagedStoreInterface`，需手动在 Azure Portal 创建索引

### 6.3 CombinedStore — 混合存储组合

```php
use Symfony\AI\Store\CombinedStore;

// 用 Qdrant 做向量搜索，Elasticsearch 做全文搜索，RRF 融合
$combined = new CombinedStore(
    vectorStore: $qdrantStore,
    textStore: $elasticsearchStore,
    rrfK: 60, // RRF 常数，默认 60
);

// CombinedStore 自动将 HybridQuery 分解并融合结果
$results = $combined->query(
    new HybridQuery($vector, $textTerms, semanticRatio: 0.6),
    ['maxItems' => 10]
);
```

**RRF 算法原理**：

```
RRF_score(doc) = Σᵢ 1 / (k + rankᵢ)
```

- k=60 是经验常数，防止排名靠前的文档获得过度加成
- 文档在向量搜索中排名第 1：1/(60+1) ≈ 0.0164
- 文档在文本搜索中排名第 1：1/(60+1) ≈ 0.0164
- 两个列表都排名第 1：0.0164 + 0.0164 = 0.0328（得分翻倍，优先返回）

---

## 7. Reranker 重排序

### 7.1 为什么需要 Reranker

向量搜索使用**双编码器（Bi-Encoder）**架构：查询和文档分别独立编码为向量，通过向量相似度排序。这种方式速度快，但精度有限，因为查询和文档向量在编码时没有"互相看到对方"。

**交叉编码器（Cross-Encoder）**：查询和文档拼接后一起送入模型，模型可以做细粒度的语义匹配，精度远高于双编码器，但速度慢（O(n) 次模型调用）。

两阶段检索策略：

```
向量搜索（快，topK=50）→ Reranker（慢但准，topK=5）→ LLM 生成（用 5 条精排文档）
```

### 7.2 RerankerInterface

```php
interface RerankerInterface
{
    /**
     * @param list<VectorDocument> $documents
     * @return list<VectorDocument> 重排序后的文档，按相关性从高到低排列，限制为 $topK 条
     */
    public function rerank(string $query, array $documents, int $topK = 5): array;
}
```

### 7.3 Reranker 实现

`Reranker` 委托 `PlatformInterface` 调用重排序模型（如 Cohere Rerank、混元、Jina Rerank）：

```php
final class Reranker implements RerankerInterface
{
    public function __construct(
        private readonly PlatformInterface $platform,
        private readonly string $model, // 如 'cohere-rerank-v3', 'jina-reranker-v2-base-multilingual'
    ) {}

    public function rerank(string $query, array $documents, int $topK = 5): array
    {
        // 1. 从 metadata._text 或 metadata._source 提取文本
        $texts = array_map(fn($d) => $d->getMetadata()->getText() ?? '', $documents);

        // 2. 调用 Platform 的重排序 API
        $entries = $this->platform->invoke($this->model, ['query' => $query, 'texts' => $texts])->asReranking();

        // 3. 按 cross-encoder 分数排序，取 topK
        usort($entries, fn($a, $b) => $b->getScore() <=> $a->getScore());
        $entries = array_slice($entries, 0, $topK);

        // 4. 返回重排序后的 VectorDocument，携带新 score
        return array_map(fn($e) => $documents[$e->getIndex()]->withScore($e->getScore()), $entries);
    }
}
```

**重要前提**：`Reranker` 从 `Metadata::getText()` 获取文档文本。`Vectorizer` 在向量化时会自动调用 `$metadata->setText($document->getContent())` 保存原文，因此只要通过 `Vectorizer` 索引的文档，均支持 Reranker。

### 7.4 RerankerListener — 事件驱动重排序

通过事件监听器，可以在 `Retriever` 的查询后自动触发 Reranker，无需修改任何业务代码：

```php
use Symfony\AI\Store\EventListener\RerankerListener;

$listener = new RerankerListener(
    reranker: new Reranker($platform, 'cohere-rerank-v3'),
    topK: 5,  // 默认返回 5 条
);

// 注册到事件分发器（监听 PostQueryEvent）
$dispatcher->addListener(PostQueryEvent::class, $listener);

// 之后所有 Retriever::retrieve() 调用都自动经过 Reranker
$docs = $retriever->retrieve('搜索关键词'); // 自动 Reranker 处理
```

### 7.5 不同 Reranker 模型对比

| 模型 | 提供商 | 语言支持 | 特点 | 适用场景 |
|-----|-------|---------|------|---------|
| cohere-rerank-v3 | Cohere | 多语言 | 精度高，业界标杆 | 通用 RAG |
| cohere-rerank-english-v3.0 | Cohere | 英文 | 英文精度最优 | 英文文档库 |
| jina-reranker-v2-base-multilingual | Jina AI | 多语言 | 开源可本地部署 | 中文/多语言 |
| BAAI/bge-reranker-large | 智源 | 中英文 | 中文效果好 | 中文知识库 |

---

## 8. 事件系统与扩展机制

### 8.1 PreQueryEvent — 查询前干预

在向量化查询之前触发，可以修改查询字符串和选项：

```php
use Symfony\AI\Store\Event\PreQueryEvent;

// 监听器：查询拼写纠正
$dispatcher->addListener(PreQueryEvent::class, function(PreQueryEvent $event): void {
    $corrected = $spellingCorrector->correct($event->getQuery());
    $event->setQuery($corrected);
});

// 监听器：同义词扩展
$dispatcher->addListener(PreQueryEvent::class, function(PreQueryEvent $event): void {
    $synonyms = $synonymDict->expand($event->getQuery());
    $options = $event->getOptions();
    $options['synonyms'] = $synonyms;
    $event->setOptions($options);
});
```

**应用场景**：
- 拼写纠正（"苹果手机" → "iPhone"）
- 查询扩展（增加同义词）
- 查询改写（口语 → 专业术语）
- A/B 测试（修改 `semanticRatio`）

### 8.2 PostQueryEvent — 结果后处理

在存储返回结果后、业务代码接收前触发，可以修改结果集：

```php
use Symfony\AI\Store\Event\PostQueryEvent;

// 监听器：过滤低分结果
$dispatcher->addListener(PostQueryEvent::class, function(PostQueryEvent $event): void {
    $docs = iterator_to_array($event->getDocuments());
    $filtered = array_filter($docs, fn($d) => $d->getScore() < 0.3); // 余弦距离<0.3
    $event->setDocuments(array_values($filtered));
});

// 监听器：结果追踪日志
$dispatcher->addListener(PostQueryEvent::class, function(PostQueryEvent $event): void {
    $analytics->track('search', [
        'query' => $event->getQuery(),
        'result_count' => count(iterator_to_array($event->getDocuments())),
    ]);
});
```

**应用场景**：
- Reranker 重排序（内置 `RerankerListener`）
- 结果过滤（过滤低相关性结果）
- 结果去重（合并相似文档）
- 搜索日志/埋点

---

## 9. 控制台命令

Store 模块提供 4 个 Symfony Console 命令：

### 9.1 ai:store:index — 文档索引

```bash
# 使用配置的默认源索引文档
php bin/console ai:store:index blog

# 指定单个源
php bin/console ai:store:index blog --source=/path/to/file.txt

# 指定多个源
php bin/console ai:store:index blog \
  --source=/docs/page1.md \
  --source=/docs/page2.md
```

**实现细节**：

- 支持 `SourceIndexer`（配合 `LoaderInterface`）和 `ConfiguredSourceIndexer`（预配置默认源）
- `ConfiguredSourceIndexer` 适合在 YAML 配置中定义默认数据源，运行时可用 `--source` 覆盖

### 9.2 ai:store:retrieve — 文档检索

```bash
# 交互式搜索（会提示输入查询）
php bin/console ai:store:retrieve blog

# 直接搜索
php bin/console ai:store:retrieve blog "如何配置 Nginx"

# 限制返回数量
php bin/console ai:store:retrieve blog "Nginx 配置" --limit=5
```

输出格式：

```
 Result #1
 +--------+--------------------------------------+
 | ID     | 3a4b5c6d-...                         |
 | Score  | 0.0234                               |
 | Source | /docs/nginx-config.md                |
 | Text   | Nginx 配置文件位于 /etc/nginx/...     |
 +--------+--------------------------------------+
```

### 9.3 ai:store:setup — 初始化存储

```bash
php bin/console ai:store:setup
```

调用 `ManagedStoreInterface::setup()`，创建数据库表/索引/集合。

### 9.4 ai:store:drop — 清空存储

```bash
php bin/console ai:store:drop
```

调用 `ManagedStoreInterface::drop()`，删除所有数据（危险操作）。

---

## 10. 异常体系

```
ExceptionInterface (接口)
    ├── InvalidArgumentException    ← 参数校验失败（如空 content、overlap >= chunkSize）
    ├── RuntimeException            ← 运行时错误（文件不存在、HTTP 请求失败）
    ├── UnsupportedFeatureException ← 功能不支持
    └── UnsupportedQueryTypeException ← Store 不支持某查询类型
```

**UnsupportedQueryTypeException** 的使用模式：

```php
// Retriever 在查询前调用 supports()，避免异常
if ($store->supports(HybridQuery::class)) {
    $query = new HybridQuery($vector, $texts, semanticRatio: 0.7);
} elseif ($store->supports(VectorQuery::class)) {
    $query = new VectorQuery($vector);
} else {
    $query = new TextQuery($texts);
}

// 若直接调用不支持的查询类型，Store 会抛出 UnsupportedQueryTypeException
try {
    $store->query(new HybridQuery($vector, $texts), $options);
} catch (UnsupportedQueryTypeException $e) {
    // 降级处理
}
```

---

## 11. 架构总结与最佳实践

### 11.1 接口层次总结

```
StoreInterface          ← 核心存储接口（所有 Bridge 实现）
ManagedStoreInterface   ← 生命周期管理（大多数 Bridge 实现）
IndexerInterface        ← 索引流水线入口
RetrieverInterface      ← 检索流水线入口
LoaderInterface         ← 文档加载
FilterInterface         ← 文档过滤
TransformerInterface    ← 文档转换
VectorizerInterface     ← 文档向量化
RerankerInterface       ← 结果重排序
```

### 11.2 选型决策树

```
是否需要持久化？
├── 否 → InMemory Store（测试/原型）
└── 是
    ├── 已有 Postgres？ → Postgres Bridge（pgvector）
    ├── 已有 Redis？ → Redis Bridge（RedisSearch）
    ├── 需要超大规模（> 1亿）？ → Milvus / Qdrant
    ├── 需要全托管云服务？
    │   ├── AWS → S3Vectors / Pinecone
    │   ├── Azure → AzureSearch
    │   └── 通用 → Pinecone / Qdrant Cloud
    ├── 本地轻量级？ → Sqlite
    └── 需要混合搜索？
        ├── 单存储原生支持 → Postgres / Sqlite / Cache
        └── 组合两个存储 → CombinedStore（向量 + 全文）
```

### 11.3 RAG 参数调优清单

| 参数 | 默认值 | 推荐调整方向 |
|-----|-------|------------|
| `chunkSize` | 1000 | 短问答：500，长文档：1000~2000 |
| `overlap` | 200 | 一般为 chunkSize 的 10~20% |
| `maxItems`（粗召回） | 未设定 | 配合 Reranker：20~50；直接使用：5~10 |
| `topK`（Reranker） | 5 | 生产环境：3~5；搜索展示：10 |
| `semanticRatio` | 0.5 | 语义理解场景：0.7，精确匹配场景：0.3 |
| 嵌入模型维度 | 1536 | 高精度需求：3072；性能优先：768 |
| `rrfK`（CombinedStore） | 60 | 一般无需调整，可微调 40~80 |

### 11.4 性能优化建议

1. **批量索引**：`DocumentProcessor` 默认 `chunk_size=50` 批量调用 `StoreInterface::add()`，减少存储 I/O 次数。
2. **批量向量化**：`Vectorizer` 自动检测 `Capability::INPUT_MULTIPLE`，支持批量 API 调用，OpenAI 批量可降低 50% 成本。
3. **速率限制保护**：大批量索引时使用 `ChunkDelayTransformer`，避免触发 API 速率限制。
4. **DistanceCalculator 批处理**：设置 `batchSize=100`，对大数据集分批计算距离，控制内存峰值。
5. **Reranker 按需使用**：Reranker 有额外 API 成本，仅在精度敏感场景使用；简单问答可直接用向量搜索 topK=5。

### 11.5 关键设计模式总结

- **策略模式**：`DistanceStrategy` 枚举 + `DistanceCalculator` 可替换距离策略
- **管道模式**：`Filter → Transformer → Vectorizer → Store` 的文档处理管道
- **工厂方法**：`Retriever::createQuery()` 根据 Store 能力自动选择查询类型
- **装饰器模式**：`ConfiguredSourceIndexer` 装饰 `SourceIndexer`，注入默认源
- **观察者模式**：`PreQueryEvent / PostQueryEvent` 允许外部逻辑干预查询流程
- **组合模式**：`CombinedStore` 组合多个 Store，`ChainTransformer` 组合多个 Transformer
- **不可变对象**：`VectorDocument::withScore()` 返回新实例，保持不可变性

---

*报告生成时间：基于 symfony/ai-store 源码分析*
*源码路径：`/home/runner/work/ai/ai/src/store/`*
