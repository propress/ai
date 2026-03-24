# 企业知识库 RAG 聊天系统（全模块深度整合）

## 业务场景

你在构建一个**企业级知识库助手**，需要具备以下全部能力：

1. **知识库问答（RAG）**：从 Markdown/JSON 文档自动建立索引，通过语义搜索回答用户问题
2. **对话持久化**：用户关闭浏览器后重新打开，完整对话历史依然保留
3. **动态记忆**：根据用户当前问题自动检索相关上下文，而非注入固定文本
4. **混合检索**：结合向量语义搜索和全文关键词搜索，并通过 Reranker 提升结果质量
5. **文档处理管线**：自动分块、向量化、存储，支持从文件批量加载
6. **工具调用**：通过容错工具箱确保生产环境稳定
7. **处理器链**：通过 InputProcessor / OutputProcessor 灵活编排请求管线

本教程整合了 Symfony AI 几乎所有核心模块，是系列教程的"毕业项目"。

**典型应用：** 企业内部知识管理、产品文档问答、技术支持自助服务、合规政策查询

## 涉及模块

| 模块 | Composer 包 | 用途 |
|------|------------|------|
| **Platform** | `symfony/ai-platform` + `symfony/ai-open-ai-platform` | 连接 OpenAI（文本生成 + Embedding） |
| **Agent** | `symfony/ai-agent` | 智能体框架、处理器链、记忆注入 |
| **Chat** | `symfony/ai-chat` + `symfony/ai-doctrine-chat` | 对话管理与 PostgreSQL 持久化 |
| **Store** | `symfony/ai-store` + `symfony/ai-mongo-db-store` | MongoDB Atlas 向量存储 |
| **Agent Bridge** | `symfony/ai-similarity-search-tool` + `symfony/ai-clock-tool` | 工具集（语义搜索、时间） |

> **💡 提示：** 本教程使用 **OpenAI**（`gpt-4o`）作为主要平台。
> [教程 01](./01-basic-chatbot.md) 中用 OpenAI 实现了基础聊天，本教程则展示 OpenAI 在完整 RAG 系统中的深度集成——
> 包括 Embedding 模型选择、Reranker 配合、动态记忆等进阶用法。
> 如果你更熟悉其他平台（Anthropic、Mistral、Gemini），只需替换 `PlatformFactory` 一行代码即可。

---

## 整体架构

```
┌──────────────────────────────────────────────────────────────────────────┐
│                              Chat 层                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ Chat：initiate() 初始化会话 / submit() 提交消息                     │ │
│  │ 自动流程：加载历史 → 追加消息 → 调用 Agent → 保存回复               │ │
│  └──────────────────────────┬──────────────────────────────────────────┘ │
│                             │                                            │
│  ┌──────────────────────────▼──────────────────────────────────────────┐ │
│  │ MessageStore（对话持久化后端）                                       │ │
│  │ · Doctrine DBAL (PostgreSQL) ── 本教程主选                          │ │
│  │ · Bridge\Redis\MessageStore ── 高性能替代                           │ │
│  │ · Bridge\MongoDb\MessageStore ── 文档数据库替代                     │ │
│  │ · Bridge\Session\MessageStore ── 轻量级 HTTP 会话                   │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
┌────────────────────────────▼─────────────────────────────────────────────┐
│                           Agent 层                                        │
│                                                                           │
│  ┌────────────── InputProcessor 链（按顺序执行）───────────────────────┐ │
│  │ ① SystemPromptInputProcessor → 注入系统提示词                       │ │
│  │ ② MemoryInputProcessor ──────→ EmbeddingProvider 动态检索相关记忆   │ │
│  │ ③ AgentProcessor（输入阶段）──→ 注册可用工具列表                    │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                             │                                            │
│                     ┌───────▼────────┐                                   │
│                     │  调用 Platform  │                                   │
│                     └───────┬────────┘                                   │
│                             │                                            │
│  ┌────────────── OutputProcessor 链（处理 AI 输出）────────────────────┐ │
│  │ ① AgentProcessor（输出阶段）──→ 检测并执行工具调用                  │ │
│  │    ├─ SimilaritySearch ──── 向量检索知识库文档                       │ │
│  │    ├─ Clock ────────────── 获取当前时间                              │ │
│  │    └─ 自定义工具 ────────── 业务查询等                               │ │
│  │   (FaultTolerantToolbox 包装：工具异常不会中断对话)                   │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
┌────────────────────────────▼─────────────────────────────────────────────┐
│                         Platform 层                                       │
│  ┌──────────────────────────────────────────────────────────────────┐     │
│  │ OpenAI gpt-4o ── 文本生成                                        │     │
│  │ OpenAI text-embedding-3-small ── 文档/查询向量化                  │     │
│  └──────────────────────────────────────────────────────────────────┘     │
│                                                                           │
│  ┌──────────────────────────────────────────────────────────────────┐     │
│  │ Store 层                                                          │     │
│  │ · MongoDB Atlas Search ── 向量存储（本教程主选）                   │     │
│  │ · CombinedStore ────────── 混合检索（向量 + 全文）                 │     │
│  │ · Retriever + 事件 ─────── 查询扩展 / 结果重排序                  │     │
│  └──────────────────────────────────────────────────────────────────┘     │
│                                                                           │
│  ┌──────────────────────────────────────────────────────────────────┐     │
│  │ 文档索引管线                                                      │     │
│  │ Loader → TextSplitTransformer → Vectorizer → Store               │     │
│  │ (SourceIndexer 自动编排)                                          │     │
│  └──────────────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 前置准备

### 环境要求

- PHP >= 8.2
- Composer
- MongoDB 7+（启用 Atlas Search 或本地向量索引）— 向量存储
- PostgreSQL 15+ — 对话持久化
- Docker（可选，快速启动依赖服务）

### 启动基础设施

```bash
# MongoDB（向量存储后端）
docker run -d --name mongodb \
    -e MONGO_INITDB_ROOT_USERNAME=root \
    -e MONGO_INITDB_ROOT_PASSWORD=secret \
    -p 27017:27017 \
    mongo:7

# PostgreSQL（对话持久化后端）
docker run -d --name postgres \
    -e POSTGRES_PASSWORD=secret \
    -e POSTGRES_DB=chat_app \
    -p 5432:5432 \
    postgres:17
```

### 安装依赖

```bash
composer require symfony/ai-platform symfony/ai-open-ai-platform \
    symfony/ai-agent symfony/ai-similarity-search-tool symfony/ai-clock-tool \
    symfony/ai-chat symfony/ai-doctrine-chat \
    symfony/ai-store symfony/ai-mongo-db-store \
    symfony/uid symfony/http-client symfony/clock \
    doctrine/dbal
```

### 设置 API 密钥

```bash
export OPENAI_API_KEY="your-openai-api-key"
```

> **🔒 安全建议：** 永远不要将 API 密钥硬编码在源代码中。在 Symfony 项目中，应使用 `.env.local` 文件或
> 服务器环境变量来管理密钥；在 CI/CD 中使用 Secrets 管理。

---

## Step 1：初始化 Platform 与 MongoDB 向量存储

```php
<?php

require 'vendor/autoload.php';

use MongoDB\Client;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Store\Bridge\MongoDb\Store as VectorStore;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\Component\HttpClient\HttpClient;

// 1. 创建 Platform 实例（连接 OpenAI）
$platform = PlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    HttpClient::create(),
);

// 2. 初始化 MongoDB 向量存储
$mongoClient = new Client('mongodb://root:secret@localhost:27017');

$vectorStore = new VectorStore(
    client: $mongoClient,
    databaseName: 'knowledge_base',
    collectionName: 'documents',
    indexName: 'vector_index',
    vectorFieldName: 'vector',
    embeddingsDimension: 1536,  // OpenAI text-embedding-3-small 输出 1536 维
);

// ManagedStoreInterface：setup() 创建必要的索引结构
$vectorStore->setup();

// 3. 初始化向量化器（使用 OpenAI Embedding 模型）
$vectorizer = new Vectorizer($platform, 'text-embedding-3-small');
```

### `ManagedStoreInterface` 生命周期

向量存储和对话存储都实现了 `ManagedStoreInterface`，它定义了两个生命周期方法：

- **`setup(array $options = [])`**：创建存储所需的表、索引或集合。在首次使用前调用一次。
- **`drop()`**：清除所有数据并删除存储结构。用于重置或清理。

```php
// 首次部署时调用
$vectorStore->setup();

// 需要完全重建时
$vectorStore->drop();
$vectorStore->setup();
```

> **⚠️ 注意：** `drop()` 会**永久删除**所有数据，仅在开发环境或确认需要重建时使用。
> 生产环境建议通过数据库迁移脚本管理 schema 变更，而非直接调用 `drop()`。

### Embedding 模型对比

| 模型 | 维度 | 适用场景 | 费用参考 |
|------|------|---------|---------|
| `text-embedding-3-small` | 1536 | 通用场景，性价比最优 | $0.02/1M tokens |
| `text-embedding-3-large` | 3072 | 高精度需求，大规模检索 | $0.13/1M tokens |
| `mistral-embed`（Mistral） | 1024 | Mistral 生态，多语言 | 按量计费 |

> **💡 提示：** `text-embedding-3-small` 是大多数项目的最佳起步选择。
> 如果发现检索质量不够好，再升级到 `text-embedding-3-large`——它的维度更高，语义区分度更细，
> 但存储开销和计算成本也会翻倍。`embeddingsDimension` 参数必须与所选模型的输出维度匹配。

**替代方案：使用 PostgreSQL pgvector**

```php
use Symfony\AI\Store\Bridge\Postgres\Store as PostgresVectorStore;

$pdo = new \PDO(
    'pgsql:host=localhost;port=5432;dbname=knowledge_base',
    'postgres', 'secret',
);
$vectorStore = new PostgresVectorStore($pdo, 'kb_documents');
$vectorStore->setup(['vector_size' => 1536]);
```

> **📝 补充：** Symfony AI Store 模块支持 **24 种**向量存储后端，
> 包括 Qdrant、Pinecone、Milvus、Weaviate、Elasticsearch、Redis 等。
> 所有后端实现统一的 `StoreInterface`，切换只需替换实例化代码。

---

## Step 2：构建文档索引管线

实际项目中，知识库文档通常存放在文件系统中（Markdown、JSON 等格式）。
Symfony AI 提供了完整的管线：**Loader → Transformer → Vectorizer → Store**。

### 2.1 使用 SourceIndexer 从文件加载

`SourceIndexer` 将文件加载和文档处理一步到位：

```php
use Symfony\AI\Store\Document\Loader\MarkdownLoader;
use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;
use Symfony\AI\Store\Indexer\DocumentProcessor;
use Symfony\AI\Store\Indexer\SourceIndexer;

// 1. 创建 Markdown 文件加载器
$loader = new MarkdownLoader();

// 2. 配置文档分块：每块 800 字符，重叠 150 字符
$splitter = new TextSplitTransformer(
    chunkSize: 800,
    overlap: 150,
);

// 3. 创建文档处理器（过滤 → 分块 → 向量化 → 存储）
$processor = new DocumentProcessor(
    vectorizer: $vectorizer,
    store: $vectorStore,
    transformers: [$splitter],
);

// 4. 创建 SourceIndexer 并索引文件
$sourceIndexer = new SourceIndexer($loader, $processor);

// 索引单个文件
$sourceIndexer->index('docs/refund-policy.md');

// 索引多个文件
$sourceIndexer->index([
    'docs/refund-policy.md',
    'docs/vip-membership.md',
    'docs/shipping-guide.md',
]);

echo "✅ 文档索引完成\n";
```

> **💡 提示：** `TextSplitTransformer` 将长文档切分成多个小块（chunks），每块独立向量化和存储。
> `overlap` 参数确保切分边界处的语义不丢失——比如一句话被切到两块中间时，重叠区域可以保留完整语义。
> 一般建议 `overlap` 设为 `chunkSize` 的 15-25%。

### 2.2 使用 JsonFileLoader 加载结构化数据

```php
use Symfony\AI\Store\Document\Loader\JsonFileLoader;

// FAQ 数据示例（faq.json）：
// [{"id": "faq-001", "question": "如何退货？", "answer": "30天内可退...", "category": "售后"}]

$jsonLoader = new JsonFileLoader(
    id: '$[*].id',           // JsonPath：文档 ID
    content: '$[*].answer',   // JsonPath：文档内容
    metadata: [
        'title' => '$[*].question',    // 映射为 metadata
        'category' => '$[*].category',
    ],
);

$jsonSourceIndexer = new SourceIndexer($jsonLoader, $processor);
$jsonSourceIndexer->index('data/faq.json');
```

### 2.3 手动创建文档（适合小量数据或测试）

```php
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Indexer\DocumentIndexer;
use Symfony\Component\Uid\Uuid;

$indexer = new DocumentIndexer($processor);

$documents = [
    new TextDocument(
        id: Uuid::v4()->toString(),
        content: '我们的退货政策：购买后 30 天内可免费退货，需保持商品原包装完好。'
            . '电子产品需附带完整配件。退款将在 5-7 个工作日内原路返回。',
        metadata: new Metadata([
            Metadata::KEY_SOURCE => 'policy/refund.md',
            Metadata::KEY_TITLE => '退货退款政策',
        ]),
    ),
    new TextDocument(
        id: Uuid::v4()->toString(),
        content: 'VIP 会员权益：享受全场 9 折、优先客服通道、每月专属优惠券、生日双倍积分、'
            . '免费快递升级。年费 299 元，连续会员享 8 折续费优惠。',
        metadata: new Metadata([
            Metadata::KEY_SOURCE => 'policy/vip.md',
            Metadata::KEY_TITLE => 'VIP 会员说明',
        ]),
    ),
    new TextDocument(
        id: Uuid::v4()->toString(),
        content: '物流配送说明：标准配送 3-5 工作日，快速配送 1-2 工作日（加收 15 元）。'
            . '偏远地区可能额外需要 2-3 天。订单满 99 元免标准运费。',
        metadata: new Metadata([
            Metadata::KEY_SOURCE => 'policy/shipping.md',
            Metadata::KEY_TITLE => '物流配送说明',
        ]),
    ),
];

$indexer->index($documents);

echo "✅ 已索引 " . count($documents) . " 个知识库文档\n";
```

### `Metadata` 系统

`Metadata` 继承自 `\ArrayObject`，提供了一组预定义 key，框架内部组件会自动识别和使用：

| 常量 | 值 | 说明 |
|------|------|------|
| `Metadata::KEY_TEXT` | `_text` | 文档原始文本（分块后保留） |
| `Metadata::KEY_SOURCE` | `_source` | 来源标识（文件路径、URL 等） |
| `Metadata::KEY_TITLE` | `_title` | 文档标题 |
| `Metadata::KEY_SUMMARY` | `_summary` | 文档摘要 |
| `Metadata::KEY_PARENT_ID` | `_parent_id` | 父文档 ID（分块时自动设置） |
| `Metadata::KEY_DEPTH` | `_depth` | 分块深度层级 |

> **📝 补充：** `TextSplitTransformer` 分块时会自动设置 `KEY_PARENT_ID` 指向原始文档，
> 使得你可以在检索后追溯到完整文档。你也可以添加任意自定义 key，例如
> `'department' => 'support'`、`'version' => '2024-01'`。

---

## Step 3：使用 Retriever 进行高级检索

`Retriever` 是检索的高层封装，自动选择最优查询策略，并支持通过事件系统扩展检索行为。

### 3.1 基本用法

```php
use Symfony\AI\Store\Retriever;

$retriever = new Retriever(
    store: $vectorStore,
    vectorizer: $vectorizer,
);

$results = $retriever->retrieve('退货怎么操作', ['max_results' => 3]);

foreach ($results as $document) {
    $title = $document->metadata[Metadata::KEY_TITLE] ?? '未知';
    echo "📄 [{$title}] {$document->content}\n";
}
```

### 3.2 查询策略自动选择

`Retriever` 内部根据配置和存储能力自动选择最优查询类型：

```
Retriever::createQuery() 决策逻辑：
│
├─ 无向量化器 → TextQuery（关键词搜索）
│
├─ Store 不支持 VectorQuery → TextQuery（回退）
│
├─ Store 支持 HybridQuery → HybridQuery（语义 + 关键词混合）
│
└─ Store 支持 VectorQuery → VectorQuery（纯语义搜索）
```

三种查询类型的差异：

```php
use Symfony\AI\Store\Query\VectorQuery;
use Symfony\AI\Store\Query\TextQuery;
use Symfony\AI\Store\Query\HybridQuery;

// 语义搜索：将查询文本向量化，按向量相似度匹配
$vectorQuery = new VectorQuery($vectorizer->vectorize('退货流程'));

// 关键词搜索：按文本关键词匹配
$textQuery = new TextQuery('退货 流程 政策');
// 也支持数组形式（多关键词 OR 逻辑）
$textQuery = new TextQuery(['退货', '流程', '政策']);

// 混合搜索：同时使用语义和关键词，semanticRatio 控制权重
$hybridQuery = new HybridQuery(
    vector: $vectorizer->vectorize('退货流程'),
    text: ['退货', '流程'],
    semanticRatio: 0.7,  // 70% 语义 + 30% 关键词
);
```

> **💡 提示：** `semanticRatio` 的最优值取决于数据特点：
> - 文档用词一致、查询规范 → 关键词权重高（0.3-0.4）
> - 用户提问口语化、用词多变 → 语义权重高（0.7-0.8）
> - 不确定时从 0.5 开始，根据实际效果调整

### 3.3 使用事件扩展检索（查询扩展 / 结果重排序）

`Retriever` 在检索前后分别派发 `PreQueryEvent` 和 `PostQueryEvent`，
你可以通过事件监听器实现查询扩展、同义词替换、结果过滤、重排序等高级功能：

```php
use Symfony\AI\Store\Event\PreQueryEvent;
use Symfony\AI\Store\Event\PostQueryEvent;
use Symfony\AI\Store\Retriever;
use Symfony\Component\EventDispatcher\EventDispatcher;

$eventDispatcher = new EventDispatcher();

// PreQueryEvent：在检索前修改查询（查询扩展）
$eventDispatcher->addListener(PreQueryEvent::class, function (PreQueryEvent $event) {
    $query = $event->getQuery();

    // 同义词扩展：用户搜"退货"时也搜"退款""售后"
    $synonyms = [
        '退货' => '退货 退款 售后',
        '会员' => '会员 VIP 权益',
        '快递' => '快递 物流 配送',
    ];

    foreach ($synonyms as $keyword => $expanded) {
        if (str_contains($query, $keyword)) {
            $event->setQuery($expanded);
            break;
        }
    }
});

// PostQueryEvent：在检索后修改结果（过滤/重排序）
$eventDispatcher->addListener(PostQueryEvent::class, function (PostQueryEvent $event) {
    $documents = iterator_to_array($event->getDocuments());

    // 过滤掉过期文档
    $filtered = array_filter($documents, function ($doc) {
        $version = $doc->metadata['version'] ?? null;
        return null === $version || $version >= '2024-01';
    });

    $event->setDocuments($filtered);
});

// 创建带事件的 Retriever
$retriever = new Retriever(
    store: $vectorStore,
    vectorizer: $vectorizer,
    eventDispatcher: $eventDispatcher,
);

$results = $retriever->retrieve('退货流程是什么');
```

### 3.4 使用 Reranker 提升搜索质量

初始检索可能返回一些"看似相关但实际不够精确"的文档。`Reranker` 使用专门的重排序模型对结果做二次精排：

```php
use Symfony\AI\Store\Reranker\Reranker;

$reranker = new Reranker(
    platform: $platform,
    model: 'gpt-4o-mini',  // 使用轻量模型做重排序
);

// 先检索 10 个候选，再精排取 Top 3
$candidates = iterator_to_array(
    $retriever->retrieve('如何申请退货退款', ['max_results' => 10])
);

$reranked = $reranker->rerank(
    query: '如何申请退货退款',
    documents: $candidates,
    topK: 3,
);

foreach ($reranked as $doc) {
    echo "📄 [{$doc->metadata[Metadata::KEY_TITLE]}]\n";
}
```

> **🏭 生产建议：** Reranker 会对每个候选文档发起一次推理调用，成本和延迟会随候选数线性增长。
> 推荐工作流：先通过 `Retriever` 粗筛 10-20 个候选，再用 `Reranker` 精排取 Top 3-5。
> 重排序模型选择轻量级的（如 `gpt-4o-mini`）以控制延迟。

### 3.5 CombinedStore：混合检索

`CombinedStore` 将一个向量存储和一个全文搜索存储组合，对 `HybridQuery` 执行双路检索，
然后通过 **Reciprocal Rank Fusion（RRF）** 算法合并排序结果：

```php
use Symfony\AI\Store\CombinedStore;

// 假设你有一个支持全文搜索的存储（如 Elasticsearch、Meilisearch）
// $textStore = new \Symfony\AI\Store\Bridge\Elasticsearch\Store(...);

$combinedStore = new CombinedStore(
    vectorStore: $vectorStore,       // 处理 VectorQuery 部分
    textStore: $textStore,           // 处理 TextQuery 部分
    rrfK: 60,                        // RRF 参数，默认 60
);

// HybridQuery 会被分解为两个子查询：
// - VectorQuery → 发送到 $vectorStore
// - TextQuery → 发送到 $textStore
// 两路结果通过 RRF 合并排序
$retriever = new Retriever(
    store: $combinedStore,
    vectorizer: $vectorizer,
);

$results = $retriever->retrieve('退货政策和运费');
```

> **📝 补充：** RRF（Reciprocal Rank Fusion）是一种经典的排序融合算法，公式为
> `score = Σ 1/(k + rank)`。`rrfK` 参数越大，低排名结果的权重越均匀；越小则高排名结果权重越突出。
> 默认值 60 适合大多数场景。

---

## Step 4：创建 Agent 与动态记忆

### 4.1 EmbeddingProvider：上下文感知的动态记忆

与 `StaticMemoryProvider` 注入固定文本不同，`EmbeddingProvider` 会根据用户当前问题
动态检索相关文档作为记忆注入。它的工作原理：

1. 接收用户消息
2. 使用 Embedding 模型将消息向量化
3. 在向量存储中搜索最相关的文档
4. 将匹配的文档内容作为记忆上下文注入到 Agent 输入中

```php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\Clock\Clock;
use Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Memory\EmbeddingProvider;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;
use Symfony\AI\Agent\Memory\StaticMemoryProvider;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\FaultTolerantToolbox;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Model;
use Symfony\Component\Clock\NativeClock;

// 动态记忆：根据用户问题自动检索相关知识
$embeddingProvider = new EmbeddingProvider(
    platform: $platform,
    model: new Model('text-embedding-3-small'),
    vectorStore: $vectorStore,
);

// 静态记忆：始终注入的固定上下文
$staticProvider = new StaticMemoryProvider(
    '本公司名称：TechMart 科技商城',
    '客服工作时间：周一到周五 9:00-18:00',
    '当前促销活动：全场满 200 减 30',
);

// MemoryInputProcessor 支持多个 Provider，按顺序执行
$memoryProcessor = new MemoryInputProcessor([
    $staticProvider,       // 先注入固定上下文
    $embeddingProvider,    // 再注入动态检索的相关文档
]);
```

> **💡 提示：** `EmbeddingProvider` 是实现 RAG 的关键组件之一。它不同于 `SimilaritySearch` 工具——
> 工具是由 LLM 主动决定何时调用的，而 `EmbeddingProvider` 在每次请求时**自动执行**，
> 确保 LLM 总是有相关上下文。两者可以同时使用：EmbeddingProvider 提供"被动"背景知识，
> SimilaritySearch 提供"主动"精确查询能力。

### 4.2 FaultTolerantToolbox：生产级容错

在生产环境中，工具调用可能因网络超时、API 限流等原因失败。
`FaultTolerantToolbox` 包装普通 `Toolbox`，将异常转化为错误消息返回给 LLM，
避免单个工具失败导致整个对话中断：

```php
// 1. 创建原始 Toolbox
$similaritySearch = new SimilaritySearch($vectorizer, $vectorStore);
$clock = new Clock(new NativeClock());

$innerToolbox = new Toolbox([$similaritySearch, $clock]);

// 2. 包装为 FaultTolerantToolbox
$toolbox = new FaultTolerantToolbox($innerToolbox);
```

当工具执行抛出异常时，`FaultTolerantToolbox` 的行为：

- **`ToolExecutionExceptionInterface`**：返回异常中的错误描述，LLM 可以据此调整策略
- **`ToolNotFoundException`**：返回可用工具列表，LLM 可以选择正确的工具重试
- **其他异常**：不捕获，仍会向上抛出

> **🏭 生产建议：** 始终在生产环境使用 `FaultTolerantToolbox`。例如：当 `SimilaritySearch` 因
> 数据库连接超时失败时，LLM 会收到 "工具执行出错" 的消息，可以跳过知识库查询直接基于已有上下文回答，
> 而非让整个请求返回 500 错误。

### 4.3 组装完整 Agent

```php
// 系统提示
$systemPrompt = new SystemPromptInputProcessor(
    systemPrompt: <<<PROMPT
    你是 TechMart 科技商城的智能客服助手。请遵循以下规则：
    1. 优先使用知识库工具（similarity_search）查找相关文档来回答问题
    2. 如果知识库没有找到相关信息，坦诚告知用户
    3. 回答要简洁、专业、友好
    4. 涉及时间相关的问题使用 clock 工具获取当前时间
    5. 引用知识库内容时，注明来源文档
    PROMPT,
    toolbox: $toolbox,
);

$agentProcessor = new AgentProcessor($toolbox);

// 创建 Agent
$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o',
    inputProcessors: [$systemPrompt, $memoryProcessor, $agentProcessor],
    outputProcessors: [$agentProcessor],
);
```

> **📝 补充：** 处理器的顺序很重要。`InputProcessor` 按数组顺序依次执行：先注入系统提示词，
> 再注入记忆上下文（静态 + 动态），最后由 `AgentProcessor` 注册工具列表。
> `OutputProcessor` 中的 `AgentProcessor` 负责检测 AI 返回的工具调用请求并执行。

---

## Step 5：添加 Chat 对话持久化

使用 Doctrine DBAL (PostgreSQL) 作为对话存储后端：

```php
use Doctrine\DBAL\DriverManager;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\Bridge\Doctrine\DoctrineDbalMessageStore;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\Message\SystemMessage;
use Symfony\AI\Platform\Message\UserMessage;

// 1. 连接 PostgreSQL
$dbalConnection = DriverManager::getConnection([
    'url' => 'pgsql://postgres:secret@localhost:5432/chat_app',
]);

// 2. 创建 MessageStore（Doctrine DBAL 后端）
$chatStore = new DoctrineDbalMessageStore(
    tableName: 'chat_user123_session_abc',  // 按用户+会话隔离
    dbalConnection: $dbalConnection,
);

// 3. 初始化表结构（首次使用前调用一次）
$chatStore->setup();

// 4. 创建 Chat 实例
$chat = new Chat(
    agent: $agent,
    store: $chatStore,
);

// 5. 初始化对话（清除旧消息，设置初始消息）
$chat->initiate(new MessageBag(
    new SystemMessage('你是 TechMart 科技商城的智能客服助手。'),
));
```

> **⚠️ 注意：** `Chat::initiate()` 会**清除**该 MessageStore 中的所有历史消息并保存初始 MessageBag。
> 只在创建新会话时调用。后续对话使用 `submit()` 追加消息——它会自动加载历史、调用 Agent、保存回复。

### 对话存储后端对比

Symfony AI Chat 模块提供 **10 种** MessageStore 后端，以下是最常用的几种：

**Redis（高性能缓存）：**

```php
use Symfony\AI\Chat\Bridge\Redis\MessageStore as RedisMessageStore;

$redis = new \Redis();
$redis->connect('127.0.0.1', 6379);

$chatStore = new RedisMessageStore(
    redis: $redis,
    indexName: 'chat:user123:session_abc',
);
$chatStore->setup();
```

**MongoDB（文档数据库）：**

```php
use Symfony\AI\Chat\Bridge\MongoDb\MessageStore as MongoDbMessageStore;

$chatStore = new MongoDbMessageStore(
    client: $mongoClient,
    databaseName: 'chat_app',
    collectionName: 'conversations',
);
$chatStore->setup();
```

**HTTP Session（轻量级，适合简单 Web 应用）：**

```php
use Symfony\AI\Chat\Bridge\Session\MessageStore as SessionMessageStore;
use Symfony\Component\HttpFoundation\RequestStack;

$chatStore = new SessionMessageStore(
    requestStack: $requestStack,  // 由 Symfony 注入
    sessionKey: 'ai_chat_messages',
);
$chatStore->setup();
```

**PSR-6 Cache（通用缓存适配）：**

```php
use Symfony\AI\Chat\Bridge\Cache\MessageStore as CacheMessageStore;

$chatStore = new CacheMessageStore(
    cache: $cachePool,          // 任何 PSR-6 CacheItemPoolInterface 实现
    cacheKey: 'chat_session_abc',
    ttl: 86400,                 // 24 小时过期
);
$chatStore->setup();
```

**InMemory（开发/测试）：**

```php
use Symfony\AI\Chat\InMemory\Store as InMemoryChatStore;

$chatStore = new InMemoryChatStore('dev_session');
```

| 后端 | 持久化 | 性能 | 适用场景 |
|------|--------|------|---------|
| **Doctrine DBAL** | ✅ 持久 | 中等 | 需要事务、审计、与现有 DB 集成 |
| **Redis** | ✅ 持久（需配置） | 极高 | 高并发、低延迟场景 |
| **MongoDB** | ✅ 持久 | 高 | 与 MongoDB 生态统一 |
| **Session** | ❌ 随会话 | 高 | 简单 Web 应用、无需跨设备 |
| **Cache** | ⏱ TTL 控制 | 高 | 利用现有缓存基础设施 |
| **InMemory** | ❌ 进程内 | 极高 | 开发、测试 |

> **🏭 生产建议：** 选择 Doctrine DBAL 的优势是可以与应用的其他数据共用同一个数据库连接和事务管理。
> 如果应用已经使用 PostgreSQL，这是最自然的选择。对于高并发场景（如实时客服），Redis 是更好的选择。

---

## Step 6：完整对话流程演示

```php
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\Message\SystemMessage;
use Symfony\AI\Platform\Message\UserMessage;

// 创建新会话
$chat->initiate(new MessageBag(
    new SystemMessage('你是 TechMart 科技商城的智能客服助手。用简洁友好的语气回答。'),
));

// 模拟多轮对话
$questions = [
    '你好，我想问一下退货的流程是怎样的？',
    '退款一般多久到账？',
    '你们有 VIP 会员服务吗？',
    '包邮条件是什么？',
    '现在几点了？',
];

foreach ($questions as $question) {
    echo "👤 用户：{$question}\n";
    $response = $chat->submit(new UserMessage($question));
    echo "🤖 客服：{$response->getContent()}\n\n";
}
```

执行结果示例：

```
👤 用户：你好，我想问一下退货的流程是怎样的？
🤖 客服：您好！根据我们的退货政策（来源：policy/refund.md），
   购买后 30 天内可免费退货，需保持商品原包装完好。
   电子产品需附带完整配件。退款将在审核通过后 5-7 个工作日内原路返回。

👤 用户：退款一般多久到账？
🤖 客服：退款在审核通过后 5-7 个工作日内原路返回到您的支付账户。

👤 用户：你们有 VIP 会员服务吗？
🤖 客服：有的！VIP 会员年费 299 元，享受全场 9 折、优先客服、
   每月专属优惠券、生日双倍积分、免费快递升级等权益。
   连续会员还可享 8 折续费优惠。

👤 用户：包邮条件是什么？
🤖 客服：订单满 99 元即可免标准运费（3-5 工作日）。
   如需快速配送（1-2 工作日），加收 15 元。

👤 用户：现在几点了？
🤖 客服：当前时间是 2025 年 1 月 15 日 14:32（北京时间）。
```

### 完整一次请求的数据流

```
1. 用户发送消息
   ↓
2. Chat 层：从 DoctrineDbalMessageStore 加载对话历史（MessageBag）
   ↓
3. Chat 层：将用户消息追加到 MessageBag
   ↓
4. Agent 层 - InputProcessor 链：
   a. SystemPromptInputProcessor → 注入系统提示词
   b. MemoryInputProcessor：
      ├─ StaticMemoryProvider → 注入固定信息（公司名、工作时间）
      └─ EmbeddingProvider → 向量化用户消息 → 检索 MongoDB → 注入相关文档
   c. AgentProcessor（输入）→ 从 FaultTolerantToolbox 读取工具定义
   ↓
5. Platform 层：将完整 MessageBag + options 发送给 OpenAI gpt-4o
   ↓
6. LLM 返回响应，可能包含 tool_calls
   ↓
7. Agent 层 - OutputProcessor 链：
   a. AgentProcessor（输出）检查响应：
      ├─ 包含 tool_calls → FaultTolerantToolbox 执行：
      │   ├─ SimilaritySearch → 向量化搜索词 → MongoDB 检索 → 返回相关文档
      │   ├─ Clock → 获取系统时间
      │   └─ 工具异常 → 返回错误消息（不中断对话）
      ├─ 将工具结果追加到 MessageBag
      ├─ 再次调用 Platform（LLM 基于工具结果生成回答）
      └─ 重复直到 LLM 返回纯文本
   ↓
8. Chat 层：将用户消息 + AI 回复保存到 DoctrineDbalMessageStore
   ↓
9. 返回 AssistantMessage 给用户
```

---

## Step 7：在 Symfony 控制器中集成

```php
namespace App\Controller;

use Symfony\AI\Chat\Chat;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\Message\SystemMessage;
use Symfony\AI\Platform\Message\UserMessage;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Routing\Attribute\Route;
use Symfony\Component\Uid\Uuid;

final class ChatController extends AbstractController
{
    public function __construct(
        private Chat $chat,
    ) {
    }

    #[Route('/chat/start', methods: ['POST'])]
    public function startChat(): JsonResponse
    {
        $sessionId = Uuid::v4()->toString();

        $this->chat->initiate(new MessageBag(
            new SystemMessage('你是 TechMart 智能客服。'),
        ));

        return $this->json(['session_id' => $sessionId]);
    }

    #[Route('/chat/message', methods: ['POST'])]
    public function sendMessage(Request $request): JsonResponse
    {
        $message = $request->getPayload()->getString('message');
        $response = $this->chat->submit(new UserMessage($message));

        return $this->json([
            'reply' => $response->getContent(),
        ]);
    }
}
```

> **🔒 安全建议：** MessageStore 的 key（如表名或 indexName）中的用户 ID 必须来自服务端认证（如
> `$this->getUser()->getId()`），而非客户端提交。始终使用 `{userId}:{sessionId}` 格式构建 key，
> 确保用户只能访问自己的会话。

> **🏭 生产建议：** 在 Symfony 项目中推荐通过 AI Bundle（`symfony/ai-bundle`）来配置 Agent、Chat Store 等服务，
> 通过依赖注入获取实例，而不是手动创建。参见 AI Bundle 文档了解 YAML 配置方式。

---

## 从开发到生产：存储后端切换

| 组件 | 开发/测试 | 生产推荐 | 切换方式 |
|------|----------|---------|---------|
| **向量存储** | `InMemory\Store` | `Bridge\MongoDb\Store` | 替换 `$vectorStore` 实例 |
| **对话存储** | `InMemory\Store` | `Bridge\Doctrine\DoctrineDbalMessageStore` | 替换 `$chatStore` 实例 |
| **AI 平台** | 任意 Bridge | 同左 | 替换 `PlatformFactory` |

> **🏭 生产建议：** MongoDB Atlas Search 对向量索引有自动优化。对于本地 MongoDB 部署，
> 确保 `setup()` 创建的向量索引类型正确。如果使用 PostgreSQL pgvector，
> 大规模数据集（百万级文档）建议配置 HNSW 索引以加速查询。

---

## 关键知识点总结

| 模块组合 | 用途 |
|---------|------|
| Platform 单独 | 简单的一问一答 |
| Platform + Chat | 多轮持久化对话 |
| Platform + Agent + Toolbox | 带工具的智能助手 |
| Platform + Store + Agent | RAG 知识库问答 |
| Platform + Agent + Memory | 带记忆的个性化助手 |
| **全部组合** | **企业级知识库助手（本教程）** |

| 概念 | 说明 |
|------|------|
| `DocumentProcessor` 管线 | 过滤 → 分块（`TextSplitTransformer`）→ 向量化 → 存储，一条龙处理 |
| `SourceIndexer` | 结合 `LoaderInterface`（如 `MarkdownLoader`）和 `DocumentProcessor`，从文件直接索引 |
| `DocumentIndexer` | 手动创建 `TextDocument` 后交给 `DocumentProcessor` 索引 |
| `Metadata` 系统 | 预定义 key（`KEY_SOURCE`、`KEY_TITLE`、`KEY_PARENT_ID` 等）+ 自定义扩展 |
| `Retriever` 自动策略 | 根据向量化器和存储能力自动选择 `HybridQuery` > `VectorQuery` > `TextQuery` |
| `PreQueryEvent` / `PostQueryEvent` | 检索前修改查询（扩展/重写），检索后修改结果（过滤/重排序） |
| `Reranker` | 对初始检索结果做二次精排，提升 Top-K 质量 |
| `CombinedStore` | 组合向量存储和全文存储，通过 RRF 算法融合双路结果 |
| `EmbeddingProvider` | 动态记忆——自动将用户消息向量化并检索相关文档作为上下文注入 |
| `FaultTolerantToolbox` | 包装 Toolbox，将工具异常转为错误消息，避免对话中断 |
| `ManagedStoreInterface` | `setup()` 创建存储结构，`drop()` 清除。向量存储和对话存储都实现此接口 |
| InputProcessor 链 | 按顺序执行：系统提示 → 记忆注入 → 工具注册 |
| OutputProcessor 链 | `AgentProcessor` 检测 tool_calls → 执行 → 回传结果 → 循环直到纯文本 |

## 系列文档总结

| 文档 | 场景 | 核心模块 | 复杂度 |
|------|------|---------|--------|
| [01-basic-chatbot](./01-basic-chatbot.md) | 基础问答 | Platform | ⭐ |
| [02-multi-turn-conversation](./02-multi-turn-conversation.md) | 持久化对话 | Platform + Chat | ⭐⭐ |
| [03-tool-augmented-assistant](./03-tool-augmented-assistant.md) | 工具调用 | Platform + Agent | ⭐⭐ |
| [04-rag-knowledge-base](./04-rag-knowledge-base.md) | 知识库 RAG | Platform + Agent + Store | ⭐⭐⭐ |
| [05-customer-service-multi-agent](./05-customer-service-multi-agent.md) | 多智能体路由 | Platform + Agent MultiAgent | ⭐⭐⭐ |
| [06-structured-data-extraction](./06-structured-data-extraction.md) | 结构化提取 | Platform + StructuredOutput | ⭐⭐ |
| [07-multimodal-content-understanding](./07-multimodal-content-understanding.md) | 多模态处理 | Platform + Content 类 | ⭐⭐ |
| [08-agent-with-memory](./08-agent-with-memory.md) | 长期记忆 | Platform + Agent + Store | ⭐⭐⭐ |
| [09-web-research-assistant](./09-web-research-assistant.md) | 联网研究 | Platform + Agent + 工具桥接 | ⭐⭐⭐ |
| [10-rag-chat-with-persistence](./10-rag-chat-with-persistence.md) | **全模块深度整合** | **全部** | ⭐⭐⭐⭐ |

## 下一步

如果你需要对海量邮件进行智能分类、自动分发和草稿回复，请看 [11-smart-email-triage-and-auto-reply.md](./11-smart-email-triage-and-auto-reply.md)。
