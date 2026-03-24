# RSS 新闻聚合与智能摘要

## 业务场景

你是一个行业分析师，需要每天跟踪 10+ 个技术博客、新闻网站和行业媒体的 RSS 订阅源。每天新文章有几十到上百篇，不可能全部读完。你需要 AI 自动抓取 RSS 源、分块处理长文、筛选与你关注领域相关的文章、用向量存储支持语义检索、生成每篇文章的结构化摘要、最后汇总为一份每日简报。

**典型应用：** 行业新闻监控、技术趋势追踪、舆情监控、投资情报、内容策展、竞品信息聚合

## 涉及模块

| 模块 | 包名 | 用途 |
|------|------|------|
| **Platform** | `symfony/ai-platform` | 统一 AI 平台接口 |
| **Platform Bridge** | `symfony/ai-anthropic-platform` | Claude 内容分析（主） |
| **Platform Bridge** | `symfony/ai-open-ai-platform` | OpenAI Embedding（向量化） |
| **Store** | `symfony/ai-store` | 向量存储核心 + 文档处理管线 |
| **Store Bridge** | `symfony/ai-store-mongodb` | MongoDB 向量存储 |
| **Store Bridge** | `symfony/ai-store-meilisearch` | Meilisearch 全文+向量检索 |
| **Agent** | `symfony/ai-agent` | 工具调用 + 流程编排 |
| **Agent Bridge** | `symfony/ai-firecrawl-tool` | 全文抓取（可选） |

> 💡 **提示：** `RssFeedLoader` 是 Store 模块自带的 RSS 加载器，不需要额外安装。它使用 `HttpClientInterface` 抓取 RSS/Atom 源，并为每篇文章生成 UUIDv5 以实现天然去重。

## 架构概述

RSS 聚合系统的完整数据流：从 RSS 源抓取 → 文档处理管线（分块、过滤、摘要） → 向量化存储 → 语义检索 → 日报生成。

```
RSS 订阅源（多个 URL）
    │
    ▼
┌─────────────────────────────────────────────┐
│          RssFeedLoader                       │
│  HTTP 抓取 → 解析 RSS/Atom → TextDocument   │
│  UUIDv5 去重（基于 <link> 或 <guid>）        │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│        DocumentProcessor 管线                │
│                                             │
│  Filter → Transformer → Vectorizer → Store  │
│    │           │             │          │   │
│    │     TextSplit +     Embedding    MongoDB│
│    │     Summary         (OpenAI)    /Meili │
│    │                                        │
│  TextContains        chunk_size=800         │
│  Filter              overlap=150            │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│            Retriever                         │
│  VectorQuery / TextQuery / HybridQuery      │
│  PreQueryEvent → 查询扩展                    │
│  PostQueryEvent → 结果过滤/排序              │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│         AI 分析 + 日报生成                    │
│  StructuredOutput → ArticleSummary          │
│  StructuredOutput → DailyBriefing           │
└─────────────────────────────────────────────┘
```

📝 **知识扩展：** `DocumentProcessor` 是 Store 模块的核心管线，将文档处理分为四个阶段：**Filter**（过滤不需要的文档）→ **Transform**（分块、清洗、生成摘要）→ **Vectorize**（调用 Embedding 模型生成向量）→ **Store**（持久化到向量数据库）。每个阶段都可以插入自定义处理器。

## 前置准备

### 环境要求

- PHP 8.2+
- Composer
- MongoDB 7.0+（向量搜索）或 Meilisearch 1.6+（全文+向量混合搜索）

### 安装依赖

```bash
# 核心 Platform + Anthropic（内容分析）+ OpenAI（Embedding）
composer require symfony/ai-platform symfony/ai-anthropic-platform symfony/ai-open-ai-platform

# Agent（流程编排，可选）
composer require symfony/ai-agent

# Store 核心 + MongoDB 后端
composer require symfony/ai-store symfony/ai-store-mongodb

# 替代 Store 后端（按需选择）
composer require symfony/ai-store-meilisearch     # Meilisearch
composer require symfony/ai-store-postgres        # PostgreSQL pgvector
composer require symfony/ai-store-elasticsearch   # Elasticsearch
```

### 设置 API 密钥

```bash
# .env
ANTHROPIC_API_KEY=sk-ant-your-anthropic-key
OPENAI_API_KEY=sk-your-openai-key
MONGODB_URI=mongodb://localhost:27017
```

🔒 **安全建议：** 在生产环境中使用 Symfony Secrets Vault 管理 API 密钥和数据库凭据，不要提交到版本控制。

---

## Step 1：加载 RSS 订阅源

使用 Store 模块内置的 `RssFeedLoader` 从多个 RSS/Atom 源加载文章。`RssFeedLoader` 需要 `HttpClientInterface`，并会为每篇文章基于链接生成 UUIDv5 ID，实现天然去重。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Store\Document\Loader\RssFeedLoader;
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create();
$loader = new RssFeedLoader($httpClient);

// RSS 订阅源列表
$feeds = [
    'https://symfony.com/blog/feed',
    'https://stitcher.io/rss',
    'https://blog.jetbrains.com/phpstorm/feed/',
    'https://laravel-news.com/feed',
];

// 逐源加载
foreach ($feeds as $feedUrl) {
    echo "📡 加载：{$feedUrl}\n";

    $count = 0;
    foreach ($loader->load($feedUrl) as $document) {
        ++$count;
        $metadata = $document->getMetadata();
        echo \sprintf(
            "  📄 [%s] %s\n",
            $metadata[Symfony\AI\Store\Document\Metadata::KEY_TITLE] ?? '无标题',
            $document->getId(),
        );
    }

    echo "  ✅ 获取 {$count} 篇文章\n\n";
}
```

### RssFeedLoader 生成的文档结构

| 属性 | 说明 |
|------|------|
| `getId()` | UUIDv5，基于 `<link>` 或 `<guid>` 生成，相同文章始终同一 ID |
| `getContent()` | 文章正文（`<description>` 或 `<content:encoded>`） |
| `getMetadata()[KEY_TITLE]` | 文章标题 |
| `getMetadata()[KEY_SOURCE]` | 文章链接 URL |
| `getMetadata()['pubDate']` | 发布日期 |
| `getMetadata()['author']` | 作者 |

💡 **提示：** UUIDv5 的去重特性意味着：即使你多次抓取同一 RSS 源，相同文章的 ID 不会变。在后续存储时，向量数据库会自动覆盖同 ID 的文档，无需手动去重。

---

## Step 2：文档处理管线

将 RSS 文章通过 `DocumentProcessor` 管线处理：先用 `TextSplitTransformer` 分块（长文章拆成多个 chunk），再由 `Vectorizer` 生成嵌入向量，最后存入 MongoDB。

```php
<?php

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Store\Bridge\MongoDb\Store;
use Symfony\AI\Store\Document\Loader\RssFeedLoader;
use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentProcessor;
use Symfony\AI\Store\Indexer\SourceIndexer;
use Symfony\Component\HttpClient\HttpClient;

// ---- 向量化平台（OpenAI Embedding） ----
$embeddingPlatform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);
$vectorizer = new Vectorizer($embeddingPlatform, 'text-embedding-3-small');

// ---- MongoDB 向量存储 ----
$mongoClient = new \MongoDB\Client($_ENV['MONGODB_URI']);
$store = new Store(
    collection: $mongoClient->selectCollection('rss_aggregator', 'articles'),
    indexName: 'article_vector_index',
    vectorFieldName: 'embedding',
    bulkWrite: true,
);
$store->setup();

// ---- 文档处理管线 ----
$processor = new DocumentProcessor(
    vectorizer: $vectorizer,
    store: $store,
    transformers: [
        new TextSplitTransformer(chunkSize: 800, overlap: 150),
    ],
);

// ---- 加载器 + 索引器 ----
$loader = new RssFeedLoader(HttpClient::create());
$indexer = new SourceIndexer($loader, $processor);

// ---- 索引多个 RSS 源 ----
$feeds = [
    'https://symfony.com/blog/feed',
    'https://stitcher.io/rss',
    'https://blog.jetbrains.com/phpstorm/feed/',
];

foreach ($feeds as $feedUrl) {
    echo "📡 索引 {$feedUrl}...\n";
    $indexer->index($feedUrl);
    echo "  ✅ 完成\n";
}

echo "\n🎉 所有 RSS 源已索引到 MongoDB\n";
```

### 管线组件说明

| 组件 | 类名 | 作用 |
|------|------|------|
| **Loader** | `RssFeedLoader` | 从 URL 加载 RSS/Atom 为 `TextDocument` |
| **Transformer** | `TextSplitTransformer` | 将长文切割为多个 chunk，支持重叠 |
| **Vectorizer** | `Vectorizer` | 调用 Embedding 模型（如 `text-embedding-3-small`）生成向量 |
| **Store** | `MongoDb\Store` | 持久化向量文档到 MongoDB |
| **Indexer** | `SourceIndexer` | 编排 Loader → Processor 的完整管线 |

⚠️ **注意：** `TextSplitTransformer` 的 `overlap` 参数控制相邻 chunk 之间的重叠字符数。重叠太小可能丢失上下文，重叠太大会增加存储和计算成本。建议长文设为 chunk 大小的 15-20%。

---

## Step 3：AI 摘要生成（SummaryGeneratorTransformer）

Store 模块内置了 `SummaryGeneratorTransformer`，可以在索引管线中自动调用 LLM 为每个文档生成摘要，并写入 Metadata。

```php
<?php

use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory as AnthropicFactory;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory as OpenAiFactory;
use Symfony\AI\Store\Bridge\MongoDb\Store;
use Symfony\AI\Store\Document\Loader\RssFeedLoader;
use Symfony\AI\Store\Document\Transformer\SummaryGeneratorTransformer;
use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentProcessor;
use Symfony\AI\Store\Indexer\SourceIndexer;
use Symfony\Component\HttpClient\HttpClient;

// ---- 两个平台：Anthropic 做摘要，OpenAI 做 Embedding ----
$anthropicPlatform = AnthropicFactory::create($_ENV['ANTHROPIC_API_KEY']);
$openaiPlatform = OpenAiFactory::create($_ENV['OPENAI_API_KEY']);

$vectorizer = new Vectorizer($openaiPlatform, 'text-embedding-3-small');

$store = new Store(
    collection: (new \MongoDB\Client($_ENV['MONGODB_URI']))
        ->selectCollection('rss_aggregator', 'articles'),
    indexName: 'article_vector_index',
    vectorFieldName: 'embedding',
    bulkWrite: true,
);

// ---- 带摘要的处理管线 ----
$processor = new DocumentProcessor(
    vectorizer: $vectorizer,
    store: $store,
    transformers: [
        // 1. 先分块
        new TextSplitTransformer(chunkSize: 800, overlap: 150),
        // 2. 再为每个 chunk 生成摘要（写入 metadata['_summary']）
        new SummaryGeneratorTransformer(
            platform: $anthropicPlatform,
            model: 'claude-sonnet-4-5-20250929',
            yieldSummaryDocuments: false,
            systemPrompt: '用中文总结以下技术文章片段，2-3 句话，突出关键技术点。',
        ),
    ],
);

$loader = new RssFeedLoader(HttpClient::create());
$indexer = new SourceIndexer($loader, $processor);

$indexer->index('https://symfony.com/blog/feed');
echo "✅ 文章已索引，每个 chunk 自动生成了摘要\n";
```

📝 **知识扩展：** `SummaryGeneratorTransformer` 的 `yieldSummaryDocuments` 参数控制是否为摘要文本创建独立的文档。设为 `true` 时，除了原文 chunk 外还会额外生成一个摘要文档（带 `_parent_id` 指向原文），适合摘要也需要被检索的场景。设为 `false` 时，摘要仅写入原文档的 `Metadata::KEY_SUMMARY` 字段。

### Embedding 模型选择

| 模型 | 维度 | 提供商 | 适用场景 |
|------|------|--------|---------|
| `text-embedding-3-small` | 1536 | OpenAI | 通用场景，成本低 |
| `text-embedding-3-large` | 3072 | OpenAI | 高精度需求 |
| `mistral-embed` | 1024 | Mistral | 欧洲数据合规 |
| `text-embedding-004` | 768 | Google | Gemini 生态集成 |

---

## Step 4：结构化文章分析

使用结构化输出让 AI 将每篇文章提取为类型安全的 DTO，包含分类、要点、相关度评估。

```php
<?php

namespace App\Dto;

final class ArticleSummary
{
    /**
     * @param string   $title       文章标题
     * @param string   $summary     摘要（3-5 句话）
     * @param string   $category    分类：php|symfony|laravel|devops|ai|frontend|other
     * @param string[] $keyPoints   关键要点列表（3-5 条）
     * @param string   $relevance   与 PHP 生态的相关度：high|medium|low|none
     * @param string   $sentiment   文章情感：positive|neutral|negative
     * @param string   $actionable  可操作建议（可选）
     */
    public function __construct(
        public readonly string $title,
        public readonly string $summary,
        public readonly string $category,
        public readonly array $keyPoints,
        public readonly string $relevance,
        public readonly string $sentiment,
        public readonly string $actionable,
    ) {
    }
}
```

```php
<?php

use App\Dto\ArticleSummary;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Store\Document\Loader\RssFeedLoader;
use Symfony\AI\Store\Document\Metadata;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY']);
$loader = new RssFeedLoader(HttpClient::create());

$summaries = [];
foreach ($loader->load('https://symfony.com/blog/feed') as $document) {
    $content = $document->getContent();

    // 跳过内容过短的文章
    if (null === $content || mb_strlen((string) $content) < 100) {
        continue;
    }

    $title = $document->getMetadata()[Metadata::KEY_TITLE] ?? '未知标题';

    $messages = new MessageBag(
        Message::forSystem(
            '你是技术内容分析师。分析技术文章，提取关键信息。'
            .'重点关注 PHP、Symfony、Laravel 相关内容。'
            .'用中文回答。'
        ),
        Message::ofUser("分析以下文章：\n\n标题：{$title}\n\n".mb_substr((string) $content, 0, 3000)),
    );

    $result = $platform->invoke('claude-sonnet-4-5-20250929', $messages, [
        'response_format' => ArticleSummary::class,
    ]);

    $summary = $result->asObject();
    $summaries[] = $summary;

    echo \sprintf("📄 [%s] %s\n   %s\n\n", $summary->relevance, $summary->title, $summary->summary);
}
```

💡 **提示：** 结构化输出（`response_format`）利用 JSON Schema 约束 AI 的输出格式，确保返回类型安全的 PHP 对象。这比手动解析 JSON 更可靠，也避免了格式错误导致的异常。

---

## Step 5：语义检索（Retriever + HybridQuery）

`Retriever` 是 Store 模块的检索入口。它会根据 Store 的能力自动选择最佳查询方式：如果 Store 同时支持向量和文本搜索，就使用 `HybridQuery`（语义 + 关键词混合检索）。

```php
<?php

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Store\Bridge\MongoDb\Store;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Retriever;

// ---- 检索器初始化 ----
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);
$vectorizer = new Vectorizer($platform, 'text-embedding-3-small');

$mongoClient = new \MongoDB\Client($_ENV['MONGODB_URI']);
$store = new Store(
    collection: $mongoClient->selectCollection('rss_aggregator', 'articles'),
    indexName: 'article_vector_index',
    vectorFieldName: 'embedding',
);

$retriever = new Retriever(
    store: $store,
    vectorizer: $vectorizer,
);

// ---- 语义检索 ----
echo "=== PHP 性能优化相关文章 ===\n\n";
$results = $retriever->retrieve('PHP 性能优化 JIT 编译器', [
    'maxResults' => 5,
]);

foreach ($results as $document) {
    $meta = $document->metadata;
    echo \sprintf(
        "📄 [相似度: %.2f] %s\n   来源：%s\n   %s\n\n",
        $document->score ?? 0.0,
        $meta[Metadata::KEY_TITLE] ?? '未知',
        $meta[Metadata::KEY_SOURCE] ?? '未知',
        mb_substr($meta[Metadata::KEY_SUMMARY] ?? (string) $meta->getText(), 0, 200),
    );
}
```

### 使用 CombinedStore 进行混合检索

当你需要同时利用向量语义搜索和关键词全文搜索时，可以用 `CombinedStore` 将两个 Store 的结果通过 RRF（Reciprocal Rank Fusion）合并。

```php
<?php

use Symfony\AI\Store\Bridge\Meilisearch\Store as MeilisearchStore;
use Symfony\AI\Store\Bridge\MongoDb\Store as MongoDbStore;
use Symfony\AI\Store\CombinedStore;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Query\HybridQuery;
use Symfony\AI\Store\Retriever;

// ---- 两个 Store：MongoDB（向量） + Meilisearch（全文） ----
$vectorStore = new MongoDbStore(/* ... */);
$textStore = new MeilisearchStore(/* ... */);

$combinedStore = new CombinedStore(
    vectorStore: $vectorStore,
    textStore: $textStore,
    rrfK: 60,  // RRF 参数 K，值越大结果越平滑
);

$retriever = new Retriever(
    store: $combinedStore,
    vectorizer: $vectorizer,
);

// HybridQuery：70% 语义 + 30% 关键词
$results = $retriever->retrieve('Symfony 8 新特性 路由改进', [
    'maxResults' => 10,
    'semanticRatio' => 0.7,
]);

foreach ($results as $document) {
    echo \sprintf(
        "📄 [Score: %.4f] %s\n",
        $document->score ?? 0.0,
        $document->metadata[Symfony\AI\Store\Document\Metadata::KEY_TITLE] ?? '未知',
    );
}
```

📝 **知识扩展：** RRF（Reciprocal Rank Fusion）是一种排名融合算法。对于排名位置 `r` 的文档，其 RRF 分数为 `1 / (K + r + 1)`。`CombinedStore` 分别对 `vectorStore` 和 `textStore` 的结果计算 RRF 分数，再按总分排序。参数 `K` 控制平滑程度：K 越大，排名差异的影响越小。默认值 60 在大多数场景下表现良好。

### 三种查询类型对比

| 查询类型 | 类名 | 适用场景 | 需要 Vectorizer |
|----------|------|---------|----------------|
| `VectorQuery` | 纯向量相似度搜索 | 语义相似的内容 | ✅ |
| `TextQuery` | 纯关键词全文搜索 | 精确词匹配 | ❌ |
| `HybridQuery` | 向量 + 关键词混合 | 最佳综合效果 | ✅ |

---

## Step 6：生成每日简报

将当天的文章摘要汇总，用 AI 生成结构化的每日简报。

```php
<?php

namespace App\Dto;

final class DailyBriefing
{
    /**
     * @param string   $date              日期
     * @param int      $totalArticles     总文章数
     * @param int      $relevantArticles  相关文章数
     * @param string   $topStory          今日头条
     * @param string[] $highlights        要闻摘要（5-10 条）
     * @param string[] $trendingTopics    热门话题
     * @param string   $weeklyTrend       本周趋势总结
     * @param string[] $actionItems       需关注的行动项
     */
    public function __construct(
        public readonly string $date,
        public readonly int $totalArticles,
        public readonly int $relevantArticles,
        public readonly string $topStory,
        public readonly array $highlights,
        public readonly array $trendingTopics,
        public readonly string $weeklyTrend,
        public readonly array $actionItems,
    ) {
    }
}
```

```php
<?php

use App\Dto\ArticleSummary;
use App\Dto\DailyBriefing;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY']);

// 汇总所有摘要
$allSummaryText = '';
foreach ($summaries as $s) {
    /** @var ArticleSummary $s */
    $allSummaryText .= \sprintf("- [%s][%s] %s: %s\n", $s->category, $s->relevance, $s->title, $s->summary);
}

$messages = new MessageBag(
    Message::forSystem(
        '你是技术编辑。根据今天收集的文章摘要，生成一份简洁的每日技术简报。'
        .'突出最重要的新闻，发现趋势，提出需要关注的事项。用中文回答。'
    ),
    Message::ofUser(
        '今日文章（共 '.count($summaries)." 篇）：\n\n{$allSummaryText}"
    ),
);

$result = $platform->invoke('claude-sonnet-4-5-20250929', $messages, [
    'response_format' => DailyBriefing::class,
]);

$briefing = $result->asObject();

// ---- 格式化输出 ----
echo "═══════════════════════════════════════\n";
echo "  📰 技术日报 | {$briefing->date}\n";
echo "  📊 共 {$briefing->totalArticles} 篇 | 相关 {$briefing->relevantArticles} 篇\n";
echo "═══════════════════════════════════════\n\n";

echo "🏆 今日头条：\n  {$briefing->topStory}\n\n";

echo "📋 要闻速递：\n";
foreach ($briefing->highlights as $i => $highlight) {
    echo '  '.($i + 1).". {$highlight}\n";
}

echo "\n🔥 热门话题：".implode(' | ', $briefing->trendingTopics)."\n";
echo "\n📈 趋势：{$briefing->weeklyTrend}\n";

if ([] !== $briefing->actionItems) {
    echo "\n🎯 关注事项：\n";
    foreach ($briefing->actionItems as $item) {
        echo "  ⚡ {$item}\n";
    }
}
```

---

## Step 7：定时执行与 CLI 命令

Store 模块提供 `ai:store:index` 命令，可以直接在 cron 或 Symfony Scheduler 中调度 RSS 索引任务。

### 方式一：使用 Symfony 命令 + Cron

```bash
# 每 6 小时索引一次所有 RSS 源
0 */6 * * * cd /path/to/project && php bin/console ai:store:index rss_indexer

# 索引指定 URL
php bin/console ai:store:index rss_indexer https://symfony.com/blog/feed
```

### 方式二：自定义 Symfony Command

```php
<?php

namespace App\Command;

use Symfony\AI\Store\Indexer\SourceIndexer;
use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;
use Symfony\Component\Console\Style\SymfonyStyle;

#[AsCommand(name: 'app:rss:aggregate', description: '抓取 RSS 源并生成每日简报')]
final class RssAggregateCommand extends Command
{
    /** @param string[] $feedUrls */
    public function __construct(
        private readonly SourceIndexer $indexer,
        private readonly array $feedUrls,
    ) {
        parent::__construct();
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $io = new SymfonyStyle($input, $output);

        foreach ($this->feedUrls as $url) {
            $io->info("📡 索引 {$url}");
            $this->indexer->index($url);
        }

        $io->success('所有 RSS 源已索引完成');

        return Command::SUCCESS;
    }
}
```

🏭 **生产建议：**
- 使用 Symfony Messenger 将 RSS 抓取和 AI 分析分为异步任务，避免长时间阻塞。
- 为 AI 调用添加速率限制（如使用 `ChunkDelayTransformer`），避免触发 API 限流。
- 记录每次索引的文章数量和处理时间，便于监控和告警。
- 定期清理过期文章（通过 `Store::remove()` 按日期过滤）。

---

## 替代实现方案

### 使用 PostgreSQL pgvector 存储

```bash
composer require symfony/ai-store-postgres
```

```php
<?php

use Symfony\AI\Store\Bridge\Postgres\Store;

$store = new Store(
    connection: $pdo,
    tableName: 'rss_articles',
    dimension: 1536,  // text-embedding-3-small 的维度
);
$store->setup();
```

### 使用 Meilisearch（全文 + 向量混合）

```bash
composer require symfony/ai-store-meilisearch
```

```php
<?php

use Symfony\AI\Store\Bridge\Meilisearch\Store;
use Meilisearch\Client;

$store = new Store(
    client: new Client('http://localhost:7700', 'master-key'),
    indexName: 'rss_articles',
);
$store->setup();
```

💡 **提示：** Meilisearch 同时支持向量搜索和全文搜索，是 `CombinedStore` 的理想单后端替代方案。它内置 BM25 排名和向量相似度，无需部署两套存储。

### 使用 Gemini 做内容分析

```bash
composer require symfony/ai-gemini-platform
```

```php
<?php

use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;

$platform = PlatformFactory::create($_ENV['GEMINI_API_KEY']);

// Gemini 擅长长文本理解，适合分析技术文章
$result = $platform->invoke('gemini-2.0-flash', $messages, [
    'response_format' => ArticleSummary::class,
]);
```

### 使用 Firecrawl 抓取全文

RSS 只包含摘要时，可以用 Firecrawl 抓取原文链接的完整内容：

```bash
composer require symfony/ai-firecrawl-tool
```

```php
<?php

use Symfony\AI\Agent\Bridge\Firecrawl\Firecrawl;
use Symfony\AI\Store\Document\Metadata;
use Symfony\Component\HttpClient\HttpClient;

$firecrawl = new Firecrawl(HttpClient::create(), $_ENV['FIRECRAWL_API_KEY']);

foreach ($loader->load($feedUrl) as $document) {
    $url = $document->getMetadata()[Metadata::KEY_SOURCE] ?? null;

    if (null !== $url) {
        // 抓取全文（返回 Markdown 格式）
        $fullContent = $firecrawl->scrape($url);
        // 用全文替代 RSS 摘要进行索引
    }
}
```

---

## 完整流程

```
RSS 订阅源（多个 URL）
    │
    ▼
[RssFeedLoader] ── HttpClient ──► 解析 RSS/Atom
    │                              UUIDv5 去重
    ▼
[DocumentProcessor 管线]
    ├── Filter ────► 移除过短 / 无关内容
    ├── TextSplitTransformer ────► 分块（800 chars, 150 overlap）
    ├── SummaryGeneratorTransformer ──► AI 生成摘要
    ├── Vectorizer ────► Embedding（OpenAI text-embedding-3-small）
    └── Store ────► MongoDB / PostgreSQL / Meilisearch
    │
    ▼
[Retriever]
    ├── VectorQuery ────► 语义检索
    ├── TextQuery ────► 关键词检索
    └── HybridQuery ────► 混合检索（CombinedStore + RRF）
    │
    ▼
[AI 结构化分析]
    ├── ArticleSummary ────► 分类/要点/相关度
    └── DailyBriefing ────► 头条/趋势/行动项
    │
    ▼
[定时调度]
    ├── Cron + ai:store:index 命令
    └── Symfony Messenger 异步处理
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `RssFeedLoader` | Store 内置 RSS/Atom 加载器，需要 `HttpClientInterface`，输出 `TextDocument` |
| UUIDv5 去重 | 基于文章 `<link>` 生成稳定 ID，重复抓取自动覆盖 |
| `DocumentProcessor` | 文档处理管线：Filter → Transform → Vectorize → Store |
| `SourceIndexer` | 编排 Loader + Processor 的完整索引管线 |
| `TextSplitTransformer` | 按指定大小和重叠分块，适合长文章 |
| `SummaryGeneratorTransformer` | 调用 LLM 为每个 chunk 自动生成摘要 |
| `Vectorizer` | 调用 Embedding 模型生成向量，支持批量处理 |
| `CombinedStore` | 组合向量 + 文本 Store，通过 RRF 算法融合排名 |
| `HybridQuery` | 混合查询，`semanticRatio` 控制语义 vs 关键词的权重 |
| `Retriever` | 统一检索入口，自动选择最佳查询类型 |
| `Metadata::KEY_*` | 预定义元数据键：`KEY_TITLE`、`KEY_SOURCE`、`KEY_SUMMARY`、`KEY_PARENT_ID` |
| `ai:store:index` | CLI 命令，可用 cron 定时调度 |

## 下一步

如果你需要 AI 自动审查合同和法律文件，请看 [31-contract-review-system.md](./31-contract-review-system.md)。
