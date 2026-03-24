# RAG 知识库问答

## 业务场景

你在做一个企业内部知识库系统。公司有几百篇产品文档（Markdown、CSV、JSON 等格式）、技术手册、FAQ 等。员工可以用自然语言提问，系统从文档中找到最相关的内容，交给 AI 基于这些内容来回答，而不是让 AI 凭空编造。

**典型应用：** 企业知识库、文档问答系统、法律法规查询、技术文档助手、产品客服机器人

## 涉及模块

| 模块 | 包名 | 用途 |
|------|------|------|
| **Platform** | `symfony/ai-platform` + `symfony/ai-open-ai-platform` | 连接 AI 平台（文本生成 + Embedding 向量化） |
| **Store** | `symfony/ai-store` | 文档加载、转换、向量化、存储、检索的完整管线 |
| **Store Bridge** | `symfony/ai-postgres-store` | 向量数据库的具体实现（本教程使用 PostgreSQL pgvector） |
| **Agent** | `symfony/ai-agent` | 将 `SimilaritySearch` 作为工具，让 AI 自动检索相关文档 |

## 项目流程图

RAG（Retrieval-Augmented Generation）的核心是两个阶段：**索引阶段**和**查询阶段**。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     索引阶段（离线，一次性或定期执行）                      │
└─────────────────────────────────────────────────────────────────────────┘

  文件系统 / 数据库 / API
       │
       ▼
┌──────────────┐    ┌───────────────────┐    ┌──────────────┐    ┌───────────────┐
│   Loader      │──▶│   Transformer      │──▶│  Vectorizer   │──▶│    Store       │
│ 加载原始文档   │    │ 分块 + 清洗文本     │    │ 生成 Embedding │    │ 持久化到向量库  │
│ (Markdown,    │    │ (TextSplit,        │    │ (OpenAI       │    │ (PostgreSQL   │
│  CSV, JSON…)  │    │  TextTrim…)        │    │  text-embed)  │    │  pgvector)    │
└──────────────┘    └───────────────────┘    └──────────────┘    └───────────────┘


┌─────────────────────────────────────────────────────────────────────────┐
│                     查询阶段（在线，每次用户提问时执行）                    │
└─────────────────────────────────────────────────────────────────────────┘

  用户提问
       │
       ▼
┌──────────────┐    ┌───────────────┐    ┌──────────────┐    ┌───────────────┐
│  Vectorizer   │──▶│    Store       │──▶│    Agent      │──▶│   AI 回复      │
│ 向量化问题     │    │ 相似度检索     │    │ 结合上下文     │    │ 基于文档内容    │
│              │    │ (pgvector)     │    │ 调用大模型     │    │ 生成准确回答    │
└──────────────┘    └───────────────┘    └──────────────┘    └───────────────┘
```

> **💡 提示：** Store 模块提供了完整的 **加载 → 转换 → 向量化 → 存储** 管线，你无需手动实现文档分块或向量化逻辑。整条管线由 `DocumentProcessor` 自动编排。

---

## 前置准备

### 环境要求

- PHP >= 8.2
- Composer
- PostgreSQL 15+（启用 pgvector 扩展）
- Docker（可选，用于快速启动 PostgreSQL）

### 启动 PostgreSQL + pgvector

```bash
# 使用 Docker 快速启动带 pgvector 的 PostgreSQL
docker run -d --name pgvector \
    -e POSTGRES_PASSWORD=secret \
    -e POSTGRES_DB=knowledge_base \
    -p 5432:5432 \
    pgvector/pgvector:pg17
```

### 安装依赖

```bash
composer require symfony/ai-platform symfony/ai-open-ai-platform \
    symfony/ai-store symfony/ai-postgres-store \
    symfony/ai-agent
```

> **💡 提示：** `symfony/ai-store` 提供文档加载器（Loader）、转换器（Transformer）、向量化器（Vectorizer）等核心类。`symfony/ai-postgres-store` 是 PostgreSQL pgvector 的 Bridge 实现，适合生产环境。

### 设置 API 密钥

```bash
export OPENAI_API_KEY="sk-your-api-key-here"
```

> **🔒 安全建议：** 永远不要将 API 密钥硬编码在源代码中。使用环境变量或 Symfony Secrets 管理敏感信息。

---

## Step 1：用 Loader 加载文档

Store 模块提供了多种 `LoaderInterface` 实现，可以从不同数据源加载文档。每种 Loader 会将原始数据转换为 `TextDocument` 对象。

假设我们的知识库以 Markdown 文件存放在 `docs/` 目录下：

```
docs/
├── refund-policy.md
├── pricing.md
├── security.md
├── api-guide.md
└── collaboration.md
```

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Store\Document\Loader\MarkdownLoader;

// MarkdownLoader 能解析 Markdown 文件，自动提取标题并清理格式
$loader = new MarkdownLoader();

// 加载单个文件 —— 返回 TextDocument 的可迭代集合
$documents = $loader->load('docs/refund-policy.md');

foreach ($documents as $doc) {
    echo $doc->getMetadata()->getTitle() . "\n"; // 从 # 标题中自动提取
    echo $doc->getContent() . "\n";
}
```

> **💡 提示：** Store 内置了丰富的加载器，你可以根据数据源选择合适的 Loader：
>
> | 加载器 | 用途 | 示例 |
> |--------|------|------|
> | `TextFileLoader` | 纯文本文件 | `.txt` 日志、配置文件 |
> | `MarkdownLoader` | Markdown 文档 | 技术文档、README |
> | `CsvLoader` | CSV 表格数据 | 产品目录、FAQ 数据库 |
> | `JsonFileLoader` | JSON 数据文件 | API 响应数据、配置 |
> | `RssFeedLoader` | RSS 订阅源 | 新闻、博客文章 |
> | `InMemoryLoader` | 内存中的文档 | 测试、动态生成的数据 |

### 加载 CSV 格式的 FAQ

```php
use Symfony\AI\Store\Document\Loader\CsvLoader;

// 指定内容列和元数据列
$loader = new CsvLoader(
    contentColumn: 'answer',
    idColumn: 'id',
    metadataColumns: ['question', 'category'],
);

// faq.csv 格式: id,question,answer,category
$documents = $loader->load('docs/faq.csv');
```

### 加载 JSON 格式的产品数据

```php
use Symfony\AI\Store\Document\Loader\JsonFileLoader;

// 使用 JsonPath 表达式定位字段
$loader = new JsonFileLoader(
    id: '$.products[*].id',
    content: '$.products[*].description',
    metadata: [
        'name' => '$.products[*].name',
        'category' => '$.products[*].category',
    ],
);

$documents = $loader->load('docs/products.json');
```

---

## Step 2：用 Transformer 分块处理文档

真实文档往往很长（几千甚至上万字）。直接存储整篇文档会导致检索不精确——用户问一个具体问题，返回的却是一整篇文章。解决方案是将文档分成小块（Chunk）。

Store 模块提供了 `TextSplitTransformer` 来自动分块：

```php
<?php

use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;

// 将文档分割成 1000 字符的块，相邻块之间重叠 200 字符
$splitter = new TextSplitTransformer(
    chunkSize: 1000,
    overlap: 200,
);

// 加载并分块
$loader = new MarkdownLoader();
$documents = $loader->load('docs/api-guide.md');
$chunks = $splitter->transform($documents);

foreach ($chunks as $chunk) {
    echo "块内容（前 50 字）：" . mb_substr($chunk->getContent(), 0, 50) . "...\n";
    echo "来源文档 ID：" . $chunk->getMetadata()->getParentId() . "\n\n";
}
```

> **⚠️ 注意：** 分块大小（`chunkSize`）的选择很关键。太大会降低检索精度，太小会丢失上下文。通常 500-1500 字符是合理范围。`overlap`（重叠）确保块边界处的内容不会丢失上下文，建议设为 `chunkSize` 的 10%-20%。

### 组合多个 Transformer

如果需要先清理文本再分块，可以使用 `ChainTransformer`：

```php
use Symfony\AI\Store\Document\Transformer\ChainTransformer;
use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;
use Symfony\AI\Store\Document\Transformer\TextTrimTransformer;

$transformer = new ChainTransformer([
    new TextTrimTransformer(),                        // 先清除多余空白
    new TextSplitTransformer(chunkSize: 800, overlap: 150),  // 再分块
]);

$chunks = $transformer->transform($documents);
```

---

## Step 3：创建向量存储并索引文档

现在把文档块通过 `Vectorizer` 转换为 Embedding 向量，存入 PostgreSQL pgvector。

`DocumentProcessor` 是核心管线编排器，它将 **过滤 → 转换 → 向量化 → 存储** 四个步骤串联起来，一行代码完成整个索引流程。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Store\Bridge\Postgres\Store;
use Symfony\AI\Store\Document\Loader\MarkdownLoader;
use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentProcessor;
use Symfony\AI\Store\Indexer\SourceIndexer;
use Symfony\Component\HttpClient\HttpClient;

// 1. 创建 Platform（用于生成 Embedding 向量）
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());

// 2. 创建 PostgreSQL pgvector 存储
$pdo = new \PDO('pgsql:host=localhost;dbname=knowledge_base', 'postgres', 'secret');
$store = Store::fromPdo($pdo, 'document_embeddings');
$store->setup(); // 自动创建表和 pgvector 索引

// 3. 创建 Vectorizer（将文本转为 Embedding 向量）
$vectorizer = new Vectorizer($platform, 'text-embedding-3-small');

// 4. 构建完整的索引管线：Loader → Transformer → Vectorizer → Store
$processor = new DocumentProcessor(
    vectorizer: $vectorizer,
    store: $store,
    transformers: [
        new TextSplitTransformer(chunkSize: 1000, overlap: 200),
    ],
);

$indexer = new SourceIndexer(
    loader: new MarkdownLoader(),
    processor: $processor,
);

// 5. 索引所有文档
$sources = glob('docs/*.md');

foreach ($sources as $source) {
    $indexer->index($source);
    echo "✅ 已索引：{$source}\n";
}

echo "\n全部文档索引完成！\n";
```

> **💡 提示：** `SourceIndexer` 接受文件路径，自动调用 Loader 加载文档。如果你已经有了 `TextDocument` 对象（比如从数据库查询的结果），可以使用 `DocumentIndexer` 直接索引。

> **🏭 生产建议：** 索引阶段通常是离线批处理任务，可以放在 Symfony Command 中定时执行。PostgreSQL pgvector 存储的数据是持久化的，应用重启后不会丢失。

### 理解 DocumentProcessor 管线

`DocumentProcessor` 是索引流程的核心，它按以下顺序处理文档：

```
输入文档 ──▶ Filter（过滤） ──▶ Transformer（分块/清洗） ──▶ Vectorizer（向量化） ──▶ Store（存储）
```

```php
// 完整的 DocumentProcessor 配置示例
$processor = new DocumentProcessor(
    vectorizer: $vectorizer,       // 必需：将文本转为向量
    store: $store,                 // 必需：存储向量文档
    filters: [],                   // 可选：过滤不需要的文档
    transformers: [                // 可选：文档转换器链
        new TextTrimTransformer(),
        new TextSplitTransformer(chunkSize: 1000, overlap: 200),
    ],
);
```

---

## Step 4：使用 Retriever 检索文档

Store 模块提供了 `Retriever` 类作为高层检索接口。它封装了 Vectorizer + Store 的查询流程，你只需传入自然语言查询即可获得最相关的文档。

```php
<?php

use Symfony\AI\Store\Retriever;

// 创建检索器
$retriever = new Retriever(
    store: $store,
    vectorizer: $vectorizer,
);

// 自然语言检索 —— 自动将查询文本向量化并搜索
$results = $retriever->retrieve('你们的退款政策是什么？');

foreach ($results as $doc) {
    echo "相关度：" . round($doc->getScore(), 4) . "\n";
    echo "内容：" . $doc->getMetadata()->getText() . "\n\n";
}
```

### 三种查询类型

Store 模块支持三种查询策略，适用于不同的检索场景：

```php
use Symfony\AI\Store\Query\TextQuery;
use Symfony\AI\Store\Query\VectorQuery;
use Symfony\AI\Store\Query\HybridQuery;

// 1. TextQuery —— 关键词全文检索
//    适合精确匹配关键词，比如产品名称、错误代码
$results = $store->query(new TextQuery('CloudFlow API 限额'));

// 2. VectorQuery —— 语义向量检索
//    适合模糊的自然语言问题，理解语义相似度
$vector = $vectorizer->vectorize('如何申请退款？');
$results = $store->query(new VectorQuery($vector));

// 3. HybridQuery —— 混合检索（语义 + 关键词）
//    结合两种检索的优势，通过 semanticRatio 调整权重
$vector = $vectorizer->vectorize('SSO 单点登录安全认证');
$results = $store->query(new HybridQuery(
    vector: $vector,
    text: 'SSO 单点登录安全认证',
    semanticRatio: 0.7, // 0.0=纯关键词, 1.0=纯语义, 0.7=偏重语义
));
```

> **💡 提示：** 使用 `Retriever` 时，默认会根据 Store 的能力自动选择最佳查询类型。如果 Store 支持 `HybridQuery`，Retriever 会自动使用混合检索。如果需要精确控制，可以直接调用 `$store->query()` 方法。

> **⚠️ 注意：** 并非所有 Store Bridge 都支持全部查询类型。PostgreSQL pgvector 支持 `TextQuery`、`VectorQuery` 和 `HybridQuery`。可以通过 `$store->supports(HybridQuery::class)` 检查兼容性。

---

## Step 5：让 Agent 自动检索知识库

将 `SimilaritySearch` 注册为 Agent 的工具。Agent 接收到用户问题时，会自动调用该工具检索最相关的文档，然后基于文档内容生成准确回答。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 1. 创建 SimilaritySearch 工具（连接向量化器和存储）
$similaritySearch = new SimilaritySearch($vectorizer, $store);

// 2. 创建 Agent
$toolbox = new Toolbox([$similaritySearch]);
$agentProcessor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'gpt-4o-mini', [$agentProcessor], [$agentProcessor]);

// 3. 设置系统提示词
$systemPrompt = '你是 CloudFlow 产品的客服。只根据 SimilaritySearch 工具检索到的文档内容回答用户问题。'
    . '如果文档中没有相关信息，请告诉用户你无法回答并建议联系人工客服。'
    . '回答时引用相关文档来源，不要编造文档中没有的信息。';

// 4. 用户提问
$questions = [
    '你们的退款政策是什么？多久可以退？',
    '专业版一个月多少钱？有什么功能？',
    '你们的数据安全做得怎么样？',
    'API 每天可以调用多少次？',
    '你们支持 GraphQL API 吗？', // 知识库中没有的信息
];

foreach ($questions as $question) {
    echo "用户：{$question}\n";

    $messages = new MessageBag(
        Message::forSystem($systemPrompt),
        Message::ofUser($question),
    );

    $result = $agent->call($messages);
    echo "客服：" . $result->getContent() . "\n\n";
}
```

**效果：** AI 不会凭空编造，它的每一个回答都来自你索引的文档。对于知识库中不存在的信息（如 GraphQL），AI 会诚实地说"这个问题我需要转接人工客服"。

> **🏭 生产建议：** 在 Symfony 项目中，推荐通过 AI Bundle（`symfony/ai-bundle`）自动注入 `SimilaritySearch` 工具，而不是手动构建 Agent。Bundle 会自动处理依赖注入和配置。

---

## 完整示例：产品文档知识库

以下是一个完整的 RAG 知识库系统，展示从文档加载到问答的全流程：

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Store\Bridge\Postgres\Store;
use Symfony\AI\Store\Document\Loader\InMemoryLoader;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentProcessor;
use Symfony\AI\Store\Indexer\SourceIndexer;
use Symfony\Component\HttpClient\HttpClient;
use Symfony\Component\Uid\Uuid;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());

// ============================
// 索引阶段：加载 → 分块 → 向量化 → 存储
// ============================

// 1. PostgreSQL pgvector 存储
$pdo = new \PDO('pgsql:host=localhost;dbname=knowledge_base', 'postgres', 'secret');
$store = Store::fromPdo($pdo, 'document_embeddings');
$store->setup();

// 2. Vectorizer
$vectorizer = new Vectorizer($platform, 'text-embedding-3-small');

// 3. 准备知识库文档（生产中请使用 MarkdownLoader / CsvLoader 等加载文件）
$knowledgeBase = [
    new TextDocument(
        id: Uuid::v4(),
        content: 'CloudFlow 提供 14 天无理由退款。用户在购买后 14 天内可以申请全额退款。'
            . '退款将在 3-5 个工作日内退回原支付方式。企业版用户需联系客户经理处理退款。',
        metadata: new Metadata(['category' => 'billing', '_title' => '退款政策']),
    ),
    new TextDocument(
        id: Uuid::v4(),
        content: 'CloudFlow 有三个版本：基础版 ¥99/月（5 用户、10GB 存储、基础报表）；'
            . '专业版 ¥299/月（20 用户、100GB 存储、高级报表、API 访问）；'
            . '企业版按需定价（无限用户、无限存储、专属客户经理、SLA 保障、私有部署选项）。',
        metadata: new Metadata(['category' => 'billing', '_title' => '定价方案']),
    ),
    new TextDocument(
        id: Uuid::v4(),
        content: 'CloudFlow 所有数据在传输和存储时都进行 AES-256 加密。'
            . '我们通过了 SOC 2 Type II 和 ISO 27001 认证。数据中心位于 AWS 中国区域。'
            . '支持 SSO（SAML 2.0）和双因素认证。企业版支持私有部署。',
        metadata: new Metadata(['category' => 'security', '_title' => '数据安全']),
    ),
    new TextDocument(
        id: Uuid::v4(),
        content: 'CloudFlow API 使用 REST 架构，支持 JSON 格式。认证使用 Bearer Token。'
            . '基础版 API 限额：1000 次/天。专业版：10000 次/天。企业版：无限制。'
            . 'API 文档地址：https://api.cloudflow.example.com/docs',
        metadata: new Metadata(['category' => 'technical', '_title' => 'API 使用指南']),
    ),
    new TextDocument(
        id: Uuid::v4(),
        content: 'CloudFlow 支持实时协作编辑、评论和 @提及通知。项目看板支持看板视图和甘特图。'
            . '集成 Slack、企业微信、钉钉等通讯工具。支持自定义工作流和自动化触发器。',
        metadata: new Metadata(['category' => 'features', '_title' => '团队协作功能']),
    ),
];

// 4. 构建索引管线并执行
$processor = new DocumentProcessor(
    vectorizer: $vectorizer,
    store: $store,
    transformers: [
        new TextSplitTransformer(chunkSize: 800, overlap: 150),
    ],
);

$indexer = new SourceIndexer(
    loader: new InMemoryLoader($knowledgeBase),
    processor: $processor,
);

$indexer->index();
echo "✅ 索引了 " . count($knowledgeBase) . " 篇文档\n\n";

// ============================
// 查询阶段：用户提问 → 检索 → AI 回答
// ============================

$similaritySearch = new SimilaritySearch($vectorizer, $store);
$toolbox = new Toolbox([$similaritySearch]);
$agentProcessor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'gpt-4o-mini', [$agentProcessor], [$agentProcessor]);

$systemPrompt = '你是 CloudFlow 客服。只根据 SimilaritySearch 工具检索到的内容回答。'
    . '如果没有找到相关信息，请如实告知用户。';

$questions = [
    '你们支持退款吗？',
    '最便宜的方案是哪个？',
    '你们通过了哪些安全认证？',
    '可以和钉钉集成吗？',
    '你们支持 GraphQL API 吗？', // 知识库中没有的信息
];

echo "=== 知识库问答 ===\n\n";

foreach ($questions as $question) {
    echo "用户：{$question}\n";

    $result = $agent->call(new MessageBag(
        Message::forSystem($systemPrompt),
        Message::ofUser($question),
    ));

    echo "客服：" . $result->getContent() . "\n\n";
}
```

---

## 其他实现方案：使用不同的向量数据库

Symfony AI Store 提供了 20+ 种向量数据库 Bridge。以下是几种常见的替代方案：

### 方案 A：ChromaDB（轻量级，适合开发和中小规模）

```bash
composer require symfony/ai-chroma-db-store

# 启动 ChromaDB
docker run -d -p 8000:8000 chromadb/chroma
```

```php
use Codewithkyrian\ChromaDB\ChromaDB;
use Symfony\AI\Store\Bridge\ChromaDb\Store;

$client = ChromaDB::factory()
    ->withHost('http://localhost')
    ->withPort(8000)
    ->connect();

$store = new Store($client, 'knowledge_base');
$store->setup();
```

### 方案 B：Qdrant（高性能，适合大规模向量检索）

```bash
composer require symfony/ai-qdrant-store

# 启动 Qdrant
docker run -d -p 6333:6333 qdrant/qdrant
```

```php
use Symfony\AI\Store\Bridge\Qdrant\StoreFactory;

$store = StoreFactory::create(
    collectionName: 'knowledge_base',
    endpoint: 'http://localhost:6333',
    embeddingsDimension: 1536,
    embeddingsDistance: 'Cosine',
);
$store->setup();
```

### 方案 C：Elasticsearch（适合已有 ES 集群的团队）

```bash
composer require symfony/ai-elasticsearch-store
```

```php
use Symfony\AI\Store\Bridge\Elasticsearch\Store;
use Symfony\Component\HttpClient\HttpClient;

$store = new Store(
    httpClient: HttpClient::create(),
    endpoint: 'http://localhost:9200',
    indexName: 'knowledge_base',
    dimensions: 1536,
    similarity: 'cosine',
);
$store->setup();
```

### 方案 D：Meilisearch（适合同时需要全文搜索和向量检索的场景）

```bash
composer require symfony/ai-meilisearch-store
```

```php
use Symfony\AI\Store\Bridge\Meilisearch\Store;
use Symfony\Component\HttpClient\HttpClient;

$store = new Store(
    httpClient: HttpClient::create(),
    endpointUrl: 'http://localhost:7700',
    apiKey: $_ENV['MEILISEARCH_API_KEY'],
    indexName: 'knowledge_base',
);
$store->setup();
```

> **💡 提示：** 选择向量数据库时的参考建议：
> - **已有 PostgreSQL**：直接用 pgvector，零额外运维成本
> - **开发原型 / 中小规模**：ChromaDB 或 SQLite Bridge，开箱即用
> - **大规模生产（百万级文档）**：Qdrant 或 Milvus，专为向量检索优化
> - **已有搜索基础设施**：Elasticsearch / Meilisearch Bridge

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| **文档加载** | |
| `LoaderInterface` | 文档加载器接口，`load()` 方法返回 `TextDocument` 集合 |
| `MarkdownLoader` | 加载 Markdown 文件，自动提取标题和清理格式 |
| `CsvLoader` | 加载 CSV 文件，支持指定内容列和元数据列 |
| `JsonFileLoader` | 加载 JSON 文件，使用 JsonPath 表达式定位字段 |
| `InMemoryLoader` | 直接从内存中的 `TextDocument` 数组加载 |
| **文档转换** | |
| `TransformerInterface` | 文档转换器接口，`transform()` 方法处理文档集合 |
| `TextSplitTransformer` | 将长文档分块，支持 `chunkSize` 和 `overlap` 参数 |
| `ChainTransformer` | 串联多个转换器，依次执行 |
| **向量化与存储** | |
| `Vectorizer` | 使用 AI 平台将文本转为 Embedding 向量 |
| `StoreInterface` | 向量存储接口，提供 `add()`、`query()`、`remove()` 方法 |
| `DocumentProcessor` | 核心管线编排器：过滤 → 转换 → 向量化 → 存储 |
| `SourceIndexer` | 从文件路径索引：Loader → DocumentProcessor |
| `DocumentIndexer` | 从 `TextDocument` 对象索引：直接进入 DocumentProcessor |
| **检索与查询** | |
| `Retriever` | 高层检索接口，自动向量化查询并搜索 |
| `TextQuery` | 关键词全文检索 |
| `VectorQuery` | 语义向量相似度检索 |
| `HybridQuery` | 混合检索（语义 + 关键词），通过 `semanticRatio` 调整权重 |
| **Agent 集成** | |
| `SimilaritySearch` | Agent 工具，让 AI 自动调用向量检索并基于结果回答 |
| `TextDocument` | 文本文档对象，包含 ID、内容和元数据 |
| `VectorDocument` | 向量文档对象，包含 ID、向量、元数据和相关度分数 |
| `Metadata` | 文档元数据，支持 `_title`、`_source`、`_parent_id` 等内置字段 |

## 下一步

当系统变复杂时，不同类型的问题可能需要不同的处理方式。比如技术问题转技术团队，账单问题转财务团队。请看 [05-customer-service-multi-agent.md](./05-customer-service-multi-agent.md) 学习多智能体路由。
