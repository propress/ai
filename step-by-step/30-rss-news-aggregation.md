# RSS 新闻聚合与智能摘要

## 业务场景

你是一个行业分析师，需要每天跟踪 10+ 个技术博客、新闻网站和行业媒体的 RSS 订阅源。每天新文章有几十到上百篇，不可能全部读完。你需要 AI 自动抓取 RSS 源、筛选与你关注领域相关的文章、生成每篇文章的结构化摘要、最后汇总为一份每日简报。

**典型应用：** 行业新闻监控、技术趋势追踪、舆情监控、投资情报、内容策展

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台 |
| **Store Document Loader (RSS)** | 加载 RSS/Atom 订阅源 |
| **Store** | 文章向量存储，去重和检索 |
| **StructuredOutput** | 结构化文章摘要 |
| **Agent** | 编排抓取→分析→汇总流程 |

## 前置准备

```bash
composer require symfony/ai-platform symfony/ai-platform-openai
composer require symfony/ai-agent
composer require symfony/ai-store
```

---

## Step 1：加载 RSS 订阅源

使用 Store 模块的 RSS Document Loader 获取文章列表。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Store\Document\Loader\RssLoader;

// RSS 订阅源列表
$feeds = [
    'https://symfony.com/blog/feed',
    'https://stitcher.io/rss',
    'https://blog.jetbrains.com/phpstorm/feed/',
    'https://laravel-news.com/feed',
];

$loader = new RssLoader();

$allArticles = [];
foreach ($feeds as $feedUrl) {
    echo "📡 加载：{$feedUrl}\n";
    $documents = $loader->load($feedUrl);

    foreach ($documents as $doc) {
        $allArticles[] = $doc;
    }
    echo "  ✅ 获取 " . count($documents) . " 篇文章\n";
}

echo "\n📰 总计 " . count($allArticles) . " 篇新文章\n";
```

---

## Step 2：文章筛选与结构化摘要

```php
<?php

namespace App\Dto;

final class ArticleSummary
{
    /**
     * @param string   $title       文章标题
     * @param string   $summary     摘要（3-5 句话）
     * @param string   $category    分类（php/symfony/laravel/devops/ai/frontend/other）
     * @param string[] $keyPoints   关键要点列表
     * @param string   $relevance   与 PHP 生态的相关度（high/medium/low/none）
     * @param string   $sentiment   文章情感（positive/neutral/negative）
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
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$platform = PlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);

// 分析每篇文章
$summaries = [];
foreach ($allArticles as $article) {
    $content = $article->content;

    // 跳过内容太短的
    if (mb_strlen($content) < 100) {
        continue;
    }

    $messages = new MessageBag(
        Message::forSystem(
            '你是技术内容分析师。分析技术文章，提取关键信息。'
            . '重点关注 PHP、Symfony、Laravel 相关内容。'
        ),
        Message::ofUser("分析以下文章：\n\n" . mb_substr($content, 0, 3000)),
    );

    $result = $platform->invoke('gpt-4o-mini', $messages, [
        'response_format' => ArticleSummary::class,
    ]);

    $summary = $result->asObject();
    $summaries[] = $summary;

    echo "📄 [{$summary->relevance}] {$summary->title}\n";
    echo "   {$summary->summary}\n\n";
}
```

---

## Step 3：去重与向量化存储

```php
<?php

use Symfony\AI\Store\Document\Document;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Indexer;

$indexer = new Indexer($platform, 'text-embedding-3-small', $store);

foreach ($summaries as $summary) {
    // 只存储有相关性的文章
    if ('none' === $summary->relevance) {
        continue;
    }

    $doc = new Document(
        id: 'article-' . md5($summary->title),  // 用标题 hash 去重
        content: $summary->summary . "\n" . implode('; ', $summary->keyPoints),
        metadata: new Metadata([
            'title' => $summary->title,
            'category' => $summary->category,
            'relevance' => $summary->relevance,
            'sentiment' => $summary->sentiment,
            'date' => date('Y-m-d'),
        ]),
    );

    $indexer->index([$doc]);
}

echo "✅ 文章已索引，可用于后续检索\n";
```

---

## Step 4：生成每日简报

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

use App\Dto\DailyBriefing;

// 汇总所有摘要
$allSummaryText = '';
foreach ($summaries as $s) {
    $allSummaryText .= "- [{$s->category}][{$s->relevance}] {$s->title}: {$s->summary}\n";
}

$messages = new MessageBag(
    Message::forSystem(
        '你是技术编辑。根据今天收集的文章摘要，生成一份简洁的每日技术简报。'
        . '突出最重要的新闻，发现趋势，提出需要关注的事项。'
    ),
    Message::ofUser(
        "今日文章（共 " . count($summaries) . " 篇）：\n\n{$allSummaryText}"
    ),
);

$result = $platform->invoke('gpt-4o', $messages, [
    'response_format' => DailyBriefing::class,
]);

$briefing = $result->asObject();

echo "═══════════════════════════════════════\n";
echo "  📰 技术日报 | {$briefing->date}\n";
echo "  📊 共 {$briefing->totalArticles} 篇 | 相关 {$briefing->relevantArticles} 篇\n";
echo "═══════════════════════════════════════\n\n";

echo "🏆 今日头条：\n  {$briefing->topStory}\n\n";

echo "📋 要闻速递：\n";
foreach ($briefing->highlights as $i => $highlight) {
    echo "  " . ($i + 1) . ". {$highlight}\n";
}

echo "\n🔥 热门话题：" . implode(' | ', $briefing->trendingTopics) . "\n";

echo "\n📈 趋势：{$briefing->weeklyTrend}\n";

if ([] !== $briefing->actionItems) {
    echo "\n🎯 关注事项：\n";
    foreach ($briefing->actionItems as $item) {
        echo "  ⚡ {$item}\n";
    }
}
```

---

## Step 5：历史文章检索

从积累的文章库中按主题检索。

```php
<?php

use Symfony\AI\Store\Query\VectorQuery;

// 检索特定主题的历史文章
$query = new VectorQuery('PHP 性能优化 JIT', maxResults: 5);
$results = $retriever->retrieve($query);

echo "=== PHP 性能优化相关文章 ===\n";
foreach ($results as $result) {
    $meta = $result->metadata;
    echo "📄 [{$meta['date']}] {$meta['title']}\n";
    echo "   {$result->content}\n\n";
}
```

---

## 完整流程

```
RSS 订阅源（多个）
    │
    ▼
[RssLoader] → 获取文章列表
    │
    ▼
[AI 分析] → ArticleSummary（分类/摘要/相关度）
    │
    ├──► 过滤无关文章
    │
    ├──► [向量化存储] → Store（去重 + 索引）
    │
    ├──► [生成简报] → DailyBriefing
    │         ├─ 头条新闻
    │         ├─ 要闻速递
    │         ├─ 热门话题
    │         └─ 趋势分析
    │
    └──► [历史检索] → VectorQuery 主题搜索
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `RssLoader` | 从 RSS/Atom 源加载文章为 Document |
| 内容筛选 | AI 评估每篇文章的相关度 |
| 去重 | 用标题 hash 作为 Document ID |
| 每日简报 | `DailyBriefing` 结构化汇总 |
| 历史检索 | 向量存储积累后，按主题语义搜索 |

## 下一步

如果你需要 AI 自动审查合同和法律文件，请看 [31-contract-review-system.md](./31-contract-review-system.md)。
