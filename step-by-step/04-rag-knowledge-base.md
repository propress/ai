# RAG 知识库问答

## 业务场景

你在做一个企业内部知识库系统。公司有几百篇产品文档、技术手册、FAQ 等。员工可以用自然语言提问，系统从文档中找到最相关的内容，交给 AI 基于这些内容来回答，而不是让 AI 凭空编造。

**典型应用：** 企业知识库、文档问答系统、法律法规查询、技术文档助手

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台（用于文本生成和生成 Embedding） |
| **Store** | 向量数据库抽象层，存储和检索文档向量 |
| **Agent** | 将 SimilaritySearch 作为工具，让 AI 自动检索相关文档 |

## RAG 的工作原理

```
【索引阶段（一次性）】
文档 → 分块 → 生成 Embedding 向量 → 存入向量数据库

【查询阶段（每次用户提问）】
用户提问 → 生成问题的 Embedding → 在向量数据库中找最相似的文档块 → 把文档块作为上下文交给 AI → AI 基于上下文生成回答
```

---

## Step 1：准备文档数据

在真实场景中，文档可能来自数据库、文件系统、CMS 等。这里用数组模拟一组产品 FAQ 文档。

```php
<?php

// 模拟企业知识库文档
$documents = [
    [
        'title' => '退款政策',
        'content' => 'CloudFlow 提供 14 天无理由退款。用户在购买后 14 天内可以申请全额退款。'
            . '退款将在 3-5 个工作日内退回原支付方式。企业版用户需联系客户经理处理退款。',
        'category' => 'billing',
    ],
    [
        'title' => '定价方案',
        'content' => 'CloudFlow 有三个版本：基础版 ¥99/月（5 用户、10GB 存储、基础报表）；'
            . '专业版 ¥299/月（20 用户、100GB 存储、高级报表、API 访问）；'
            . '企业版按需定价（无限用户、无限存储、专属客户经理、SLA 保障、私有部署选项）。',
        'category' => 'billing',
    ],
    [
        'title' => '数据安全',
        'content' => 'CloudFlow 所有数据在传输和存储时都进行 AES-256 加密。'
            . '我们通过了 SOC 2 Type II 和 ISO 27001 认证。数据中心位于 AWS 中国区域。'
            . '支持 SSO（SAML 2.0）和双因素认证。企业版支持私有部署。',
        'category' => 'security',
    ],
    [
        'title' => 'API 使用指南',
        'content' => 'CloudFlow API 使用 REST 架构，支持 JSON 格式。认证使用 Bearer Token。'
            . '基础版 API 限额：1000 次/天。专业版：10000 次/天。企业版：无限制。'
            . 'API 文档地址：https://api.cloudflow.example.com/docs',
        'category' => 'technical',
    ],
    [
        'title' => '团队协作功能',
        'content' => 'CloudFlow 支持实时协作编辑、评论和 @提及通知。项目看板支持看板视图和甘特图。'
            . '集成 Slack、企业微信、钉钉等通讯工具。支持自定义工作流和自动化触发器。',
        'category' => 'features',
    ],
];
```

---

## Step 2：创建向量存储并索引文档

将文档转换为 Embedding 向量并存入向量数据库。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Store\Bridge\InMemory\Store;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentIndexer;
use Symfony\AI\Store\Indexer\DocumentProcessor;
use Symfony\Component\HttpClient\HttpClient;
use Symfony\Component\Uid\Uuid;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());

// 1. 创建向量存储（此处用内存存储演示，生产中用 Qdrant/ChromaDB/PostgreSQL 等）
$store = new Store();

// 2. 将文档转换为 TextDocument 对象
$textDocuments = [];
foreach ($documents as $doc) {
    $textDocuments[] = new TextDocument(
        id: Uuid::v4(),
        content: "标题：{$doc['title']}\n内容：{$doc['content']}",
        metadata: new Metadata([
            'title' => $doc['title'],
            'category' => $doc['category'],
        ]),
    );
}

// 3. 创建向量化器（使用 OpenAI 的 Embedding 模型）
$vectorizer = new Vectorizer($platform, 'text-embedding-3-small');

// 4. 创建索引器并执行索引
$indexer = new DocumentIndexer(new DocumentProcessor($vectorizer, $store));
$indexer->index($textDocuments);

echo "成功索引 " . count($textDocuments) . " 篇文档\n";
```

---

## Step 3：基于知识库回答用户问题

使用 `SimilaritySearch` 作为 Agent 的工具。当用户提问时，Agent 自动搜索最相关的文档，然后基于文档内容回答。

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
$processor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'gpt-4o-mini', [$processor], [$processor]);

// 3. 用户提问
$systemPrompt = '你是 CloudFlow 产品的客服。只根据检索到的文档内容回答用户问题。'
    . '如果文档中没有相关信息，请告诉用户你无法回答并建议联系人工客服。'
    . '回答时不要编造文档中没有的信息。';

$questions = [
    '你们的退款政策是什么？多久可以退？',
    '专业版一个月多少钱？有什么功能？',
    '你们的数据安全做得怎么样？',
    'API 每天可以调用多少次？',
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

**效果：** AI 不会凭空编造，它的每一个回答都来自你索引的文档。如果用户问的问题文档里没有，AI 会诚实地说"这个问题我需要转接人工客服"。

---

## Step 4：使用真实的向量数据库

在生产环境中，你需要使用持久化的向量数据库。以下是几种常见选择：

### 使用 ChromaDB

```bash
# 安装
composer require symfony/ai-store-chromadb

# 启动 ChromaDB（Docker）
docker run -p 8000:8000 chromadb/chroma
```

```php
use Symfony\AI\Store\Bridge\ChromaDb\StoreFactory;

$store = StoreFactory::create('knowledge_base', 'http://localhost:8000');
$store->setup(); // 创建 collection
```

### 使用 PostgreSQL + pgvector

```bash
composer require symfony/ai-store-postgresql
```

```php
use Symfony\AI\Store\Bridge\PostgreSql\StoreFactory;

$store = StoreFactory::create(
    tableName: 'document_embeddings',
    connection: $dbalConnection, // Doctrine DBAL 连接
);
$store->setup(); // 创建表和索引
```

### 使用 Qdrant

```bash
composer require symfony/ai-store-qdrant

# 启动 Qdrant
docker run -p 6333:6333 qdrant/qdrant
```

```php
use Symfony\AI\Store\Bridge\Qdrant\StoreFactory;

$store = StoreFactory::create('knowledge_base', 'http://localhost:6333');
$store->setup();
```

> **选择建议：**
> - 小规模（< 10 万文档）：ChromaDB 或 PostgreSQL pgvector
> - 中大规模：Qdrant 或 Milvus
> - 已有 Elasticsearch：使用 Elasticsearch Bridge

---

## Step 5：文档分块处理

真实的文档可能很长。需要将长文档分成小块再索引，这样检索更精确。

```php
// 对于长文档，手动分块
function splitDocument(string $content, int $chunkSize = 500, int $overlap = 50): array
{
    $chunks = [];
    $length = mb_strlen($content);
    $start = 0;

    while ($start < $length) {
        $chunk = mb_substr($content, $start, $chunkSize);
        $chunks[] = $chunk;
        $start += $chunkSize - $overlap; // 重叠部分保证上下文连贯
    }

    return $chunks;
}

// 使用分块
$longDocument = '这是一篇很长的技术文档...（假设有几千字）';
$chunks = splitDocument($longDocument);

$textDocuments = [];
foreach ($chunks as $i => $chunk) {
    $textDocuments[] = new TextDocument(
        id: Uuid::v4(),
        content: $chunk,
        metadata: new Metadata([
            'source' => '技术手册',
            'chunk_index' => $i,
        ]),
    );
}

$indexer->index($textDocuments);
```

---

## 完整示例：产品文档知识库

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
use Symfony\AI\Store\Bridge\InMemory\Store;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentIndexer;
use Symfony\AI\Store\Indexer\DocumentProcessor;
use Symfony\Component\HttpClient\HttpClient;
use Symfony\Component\Uid\Uuid;

$httpClient = HttpClient::create();
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], $httpClient);

// ===== 索引阶段 =====
$store = new Store();
$vectorizer = new Vectorizer($platform, 'text-embedding-3-small');

// 知识库文档（生产中从数据库/文件系统加载）
$knowledgeBase = [
    '退款政策：CloudFlow 提供 14 天无理由退款。退款将在 3-5 个工作日内退回原支付方式。',
    '定价：基础版 ¥99/月（5用户），专业版 ¥299/月（20用户），企业版按需定价。',
    '安全：AES-256 加密，SOC 2 + ISO 27001 认证，支持 SSO 和双因素认证。',
    'API：REST 架构，Bearer Token 认证。基础版 1000次/天，专业版 10000次/天。',
    '协作：实时编辑、评论、看板视图、甘特图、集成 Slack/企业微信/钉钉。',
];

$textDocuments = [];
foreach ($knowledgeBase as $content) {
    $textDocuments[] = new TextDocument(
        id: Uuid::v4(),
        content: $content,
        metadata: new Metadata(['source' => 'knowledge_base']),
    );
}

$indexer = new DocumentIndexer(new DocumentProcessor($vectorizer, $store));
$indexer->index($textDocuments);
echo "✅ 索引了 " . count($textDocuments) . " 篇文档\n\n";

// ===== 查询阶段 =====
$similaritySearch = new SimilaritySearch($vectorizer, $store);
$toolbox = new Toolbox([$similaritySearch]);
$processor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'gpt-4o-mini', [$processor], [$processor]);

// 模拟用户连续提问
$questions = [
    '你们支持退款吗？',
    '最便宜的方案是哪个？',
    '你们通过了哪些安全认证？',
    '可以和钉钉集成吗？',
    '你们支持 GraphQL API 吗？', // 知识库中没有的信息
];

$systemPrompt = '你是 CloudFlow 客服。只根据 SimilaritySearch 工具检索到的内容回答。'
    . '如果没有找到相关信息，请如实告知用户。';

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

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `TextDocument` | 文本文档对象，包含内容、ID 和元数据 |
| `Vectorizer` | 将文本转为 Embedding 向量 |
| `DocumentIndexer` | 批量索引文档到向量存储 |
| `Store`（各 Bridge） | 向量数据库抽象（InMemory / ChromaDB / Qdrant / PostgreSQL 等） |
| `SimilaritySearch` | Agent 工具，根据用户查询搜索最相似的文档 |
| `Metadata` | 文档元数据，可用于过滤和展示来源 |

## 下一步

当系统变复杂时，不同类型的问题可能需要不同的处理方式。比如技术问题转技术团队，账单问题转财务团队。请看 [05-customer-service-multi-agent.md](./05-customer-service-multi-agent.md) 学习多智能体路由。
