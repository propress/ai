# 智能客户反馈分析系统

## 业务场景

你在做一个 SaaS 产品。每天用户通过工单、评论、NPS 问卷、App Store 评价等渠道提交大量反馈。产品团队需要：自动分析每条反馈的情感倾向，提取关键意见和功能需求，将相似反馈聚类，发现趋势性问题。传统做法是人工读每条反馈打标签，效率极低。

**典型应用：** 用户反馈情感分析、NPS/CSAT 调研分析、App Store 评价监控、客诉热点发现、产品需求优先级排序

## 涉及模块

| 模块 | 包名 | 用途 |
|------|------|------|
| **Platform** | `symfony/ai-platform` | 核心抽象层——`PlatformInterface`、消息系统、`DeferredResult`、`Metadata` |
| **Bridge（Anthropic）** | `symfony/ai-anthropic-platform` | 主分析平台，提供 Claude 的 `PlatformFactory`、`Contract`（高质量文本理解） |
| **Bridge（DeepSeek）** | `symfony/ai-deep-seek-platform` | 批量分析的低成本方案，API 兼容 OpenAI 格式 |
| **StructuredOutput** | `symfony/ai-platform`（内置） | 将 AI 输出映射为强类型 PHP 对象（`PlatformSubscriber` + DTO） |
| **Store** | `symfony/ai-store` | 向量存储抽象——文档索引、语义检索、聚类 |
| **Store Bridge（ChromaDb）** | `symfony/ai-chroma-db-store` | 本教程的向量数据库实现，开源免费，适合中小规模 |
| **Agent** | `symfony/ai-agent` | 编排分析流程，自动调用工具完成复杂任务 |

## 架构概述

本教程综合运用了 Symfony AI 的多个核心机制：

- **StructuredOutput**：通过 `PlatformSubscriber` 事件订阅器，在 `InvocationEvent` 中注入 JSON Schema 约束，在 `ResultEvent` 中自动反序列化为 PHP 对象。你只需定义 DTO 类并在 `response_format` 选项中传入类名。
- **Store 索引管线**：`Indexer` 接收文档，调用 `EmbeddingProvider`（如 `text-embedding-3-small`）生成向量，然后存入 `StoreInterface` 实现（如 ChromaDb、Pinecone）。检索时 `Retriever` 将查询文本向量化后执行 `VectorQuery`。
- **事件系统**：`InvocationEvent` 和 `ResultEvent` 贯穿整个调用链，可用于日志记录、Token 用量追踪、动态修改模型选项等。

## 项目流程图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          客户反馈分析完整流程                                  │
└──────────────────────────────────────────────────────────────────────────────┘

  多渠道反馈                                                    结构化分析结果
  (工单/评价/NPS)                                               (DTO 对象)
      │                                                             ▲
      ▼                                                             │
┌───────────┐    ┌──────────────┐    ┌─────────────────┐    ┌──────────────┐
│  反馈文本   │──▶│  MessageBag   │──▶│ Platform::invoke │──▶│ PlatformSub- │
│  原始数据   │    │ System+User  │    │  (Anthropic)     │    │ scriber 反序  │
└───────────┘    └──────────────┘    └─────────────────┘    │ 列化为 DTO    │
                                                             └──────┬───────┘
                                                                    │
      ┌─────────────────────────────────────────────────────────────┘
      │
      ▼
┌───────────┐    ┌──────────────┐    ┌─────────────────┐    ┌──────────────┐
│ Feedback-  │──▶│   Indexer     │──▶│ EmbeddingProvider│──▶│  ChromaDb    │
│ Analysis   │    │  (索引管线)   │    │ (向量化文本)      │    │  Store       │
│ DTO 对象   │    └──────────────┘    └─────────────────┘    └──────┬───────┘
└───────────┘                                                       │
                                                                    │
      ┌─────────────────────────────────────────────────────────────┘
      │
      ├──▶ [Retriever + VectorQuery] → 相似反馈语义检索
      │
      └──▶ [Agent + StructuredOutput] → 趋势报告自动生成
```

## 前置准备

### 环境要求

- PHP >= 8.2
- Composer
- Docker（可选，用于运行 ChromaDb）

### 安装依赖

```bash
# 核心平台 + Anthropic Bridge（主分析引擎）
composer require symfony/ai-platform symfony/ai-anthropic-platform

# 向量存储（ChromaDb 开源方案）
composer require symfony/ai-store symfony/ai-chroma-db-store

# Agent（编排复杂分析流程）
composer require symfony/ai-agent
```

> **💡 提示：** `symfony/ai-anthropic-platform` 和 `symfony/ai-deep-seek-platform` 是独立的 Bridge 包。核心业务代码只依赖 `symfony/ai-platform` 的抽象接口，切换平台只需更换 Bridge 和 API 密钥——不需要改动任何分析逻辑。

### 启动 ChromaDb（Docker）

```bash
docker run -d --name chromadb -p 8000:8000 chromadb/chroma
```

### 设置 API 密钥

```bash
export ANTHROPIC_API_KEY="sk-ant-your-api-key-here"
```

> **🔒 安全建议：** 反馈分析场景中 API 密钥的泄露风险较高——批量处理脚本容易被误提交到版本库。务必使用 `.env.local` 或环境变量管理密钥，并在 `.gitignore` 中排除所有 `.env*` 文件（除 `.env.dist`）。

---

## Step 1：创建 Platform 并配置 StructuredOutput

StructuredOutput 的核心是 `PlatformSubscriber`——它监听 Platform 的事件系统，在请求发出前注入 JSON Schema，在响应返回后自动反序列化。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

// 创建事件调度器并注册 StructuredOutput 订阅器
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

// 创建 Anthropic Platform 实例
$platform = PlatformFactory::create(
    $_ENV['ANTHROPIC_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);
```

> **📝 知识扩展：** `PlatformSubscriber` 内部做了两件事：
> 1. 监听 `InvocationEvent`：检查 `response_format` 选项，如果是 DTO 类名，通过 `ResponseFormatFactory` 生成 JSON Schema 并注入请求
> 2. 监听 `ResultEvent`：使用 Symfony Serializer 将 AI 返回的 JSON 反序列化为指定的 PHP 对象
>
> 这意味着 StructuredOutput 对所有支持 JSON Mode 的平台通用（Anthropic、OpenAI、Gemini、DeepSeek 等）。

---

## Step 2：定义反馈分析 DTO

DTO（Data Transfer Object）是 StructuredOutput 的核心。Symfony AI 通过 PHP 的类型系统和 `@param` 注解自动生成 JSON Schema——AI 模型会严格按照 Schema 输出结构化数据。

```php
<?php

namespace App\Dto;

final class FeedbackAnalysis
{
    /**
     * @param string   $sentiment       情感倾向（positive/neutral/negative）
     * @param float    $sentimentScore  情感评分（-1.0 到 1.0）
     * @param string   $category        反馈类别（bug/feature_request/complaint/praise/question）
     * @param string   $summary         反馈摘要（一句话）
     * @param string[] $keywords        关键词列表
     * @param string   $productArea     涉及产品模块（ui/performance/billing/api/onboarding/other）
     * @param string   $urgency         紧急程度（critical/high/medium/low）
     * @param string   $actionItem      建议的行动项
     */
    public function __construct(
        public readonly string $sentiment,
        public readonly float $sentimentScore,
        public readonly string $category,
        public readonly string $summary,
        public readonly array $keywords,
        public readonly string $productArea,
        public readonly string $urgency,
        public readonly string $actionItem,
    ) {
    }
}
```

> **💡 提示：** DTO 设计要点：
> - 所有字段使用 `readonly` 确保不可变性
> - `@param` 注解中的说明文字会被编入 JSON Schema 的 `description`，直接影响 AI 输出质量——写得越清晰，AI 越准确
> - 使用枚举值（如 `positive/neutral/negative`）限定输出范围，减少不一致性
> - `float` 类型的 `$sentimentScore` 让后续可以做数值统计和排序

---

## Step 3：单条反馈分析

```php
<?php

use App\Dto\FeedbackAnalysis;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$feedback = '订阅了高级版，但导出PDF功能经常超时，等了3分钟都没反应。花了这么多钱结果核心功能都不能用，太让人失望了。希望尽快修复，不然只能退款了。';

$messages = new MessageBag(
    Message::forSystem(
        '你是产品反馈分析专家。分析用户反馈，提取情感、分类、关键词和行动建议。'
        . '保持客观中立，不要夸大或淡化用户的情绪。'
    ),
    Message::ofUser("请分析以下用户反馈：\n\n{$feedback}"),
);

$result = $platform->invoke('claude-sonnet', $messages, [
    'response_format' => FeedbackAnalysis::class,
]);

/** @var FeedbackAnalysis $analysis */
$analysis = $result->asObject();

echo "=== 反馈分析 ===\n";
echo "情感：{$analysis->sentiment}（{$analysis->sentimentScore}）\n";
echo "类别：{$analysis->category}\n";
echo "模块：{$analysis->productArea}\n";
echo "紧急：{$analysis->urgency}\n";
echo "摘要：{$analysis->summary}\n";
echo "关键词：" . implode('、', $analysis->keywords) . "\n";
echo "建议：{$analysis->actionItem}\n";
```

调用 `$result->asObject()` 时，`PlatformSubscriber` 已经将 AI 的 JSON 输出反序列化为 `FeedbackAnalysis` 实例。你可以直接使用强类型属性，不需要手动解析 JSON。

### 查看 Token 用量与元数据

每次调用都返回 `Metadata`，包含模型信息和 Token 消耗：

```php
$metadata = $result->getMetadata();

echo "\n--- 调用详情 ---\n";
echo "模型：" . $metadata->get('model') . "\n";

$usage = $metadata->get('token_usage');
echo "输入 Token：{$usage->inputTokens}\n";
echo "输出 Token：{$usage->outputTokens}\n";
echo "总计 Token：{$usage->totalTokens}\n";
```

> **🏭 生产建议：** 批量分析场景下，Token 用量直接决定成本。建议在事件监听器中累加 Token 统计，用于成本控制和预算告警。Anthropic Claude 的提示缓存（`cacheRetention: 'short'`）可以显著降低重复系统提示的 Token 消耗——对批量分析特别有效。

---

## Step 4：批量反馈分析与向量化存储

将分析过的反馈存入向量数据库，后续可以做聚类和相似检索。这里使用 ChromaDb 作为向量存储后端。

```php
<?php

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory as OpenAiPlatformFactory;
use Symfony\AI\Store\Bridge\ChromaDb\Store;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Indexer;

// 嵌入模型使用 OpenAI（Anthropic 目前不提供嵌入 API）
$embeddingPlatform = OpenAiPlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    HttpClient::create(),
);

// ChromaDb 向量存储
$chromaClient = new \Codewithkyrian\ChromaDB\Client('http://localhost:8000');
$store = new Store($chromaClient, 'feedback-collection');
$store->setup();

// 创建 Indexer（嵌入 + 存储一步完成）
$indexer = new Indexer($embeddingPlatform, 'text-embedding-3-small', $store);

// 模拟批量反馈
$feedbacks = [
    '新版本的搜索功能太棒了！终于可以按日期筛选了。',
    '登录页面加载很慢，有时候要等10秒以上。',
    '能不能增加暗黑模式？现在晚上用太刺眼了。',
    '付款后没收到确认邮件，很担心钱是不是白花了。',
    '你们的API文档写得非常清晰，集成只花了半天。',
    '导出功能又崩了！上次反馈还没修吗？',
    '希望支持企业SSO登录，我们公司有安全合规要求。',
    '移动端体验很好，响应速度快，赞一个。',
];

$analyses = [];

foreach ($feedbacks as $i => $feedback) {
    // AI 结构化分析
    $messages = new MessageBag(
        Message::forSystem('你是产品反馈分析专家。精确分析用户反馈的情感和分类。'),
        Message::ofUser("分析：{$feedback}"),
    );

    $result = $platform->invoke('claude-sonnet', $messages, [
        'response_format' => FeedbackAnalysis::class,
    ]);

    /** @var FeedbackAnalysis $analysis */
    $analysis = $result->asObject();
    $analyses[] = $analysis;

    // 构建文档存入向量数据库
    $document = new TextDocument(
        id: 'feedback-' . ($i + 1),
        content: $feedback,
        metadata: new Metadata([
            'sentiment' => $analysis->sentiment,
            'sentiment_score' => $analysis->sentimentScore,
            'category' => $analysis->category,
            'product_area' => $analysis->productArea,
            'urgency' => $analysis->urgency,
            'summary' => $analysis->summary,
            'keywords' => implode(',', $analysis->keywords),
        ]),
    );

    $indexer->index([$document]);
    echo "✅ 已分析反馈 #" . ($i + 1) . "：[{$analysis->sentiment}] {$analysis->summary}\n";
}
```

> **⚠️ 注意：** `Indexer` 会为每个文档调用嵌入 API 生成向量。批量处理时，建议控制并发数量避免触发 API 速率限制。对于数千条以上的批量场景，可以先收集所有文档再一次调用 `$indexer->index($allDocuments)` 提高效率。

> **📝 知识扩展：** 为什么嵌入用 OpenAI 而分析用 Anthropic？这正是 Bridge 架构的优势——不同任务可以选择最合适的平台。Anthropic Claude 在文本理解和分类上表现优异，而 OpenAI 的 `text-embedding-3-small` 是目前性价比最高的嵌入模型之一。两者通过统一的 `PlatformInterface` 无缝协作。

---

## Step 5：相似反馈语义检索

当产品经理想了解某个问题的所有相关反馈时，用 `Retriever` 进行语义检索。

```php
<?php

use Symfony\AI\Store\Retriever;

// 创建 Retriever（查询时自动向量化）
$retriever = new Retriever($store, $embeddingPlatform, 'text-embedding-3-small');

// 产品经理想查：有哪些关于性能问题的反馈？
$results = $retriever->retrieve('页面加载慢，性能问题，响应超时');

echo "=== 性能相关反馈 ===\n";
foreach ($results as $doc) {
    $meta = $doc->getMetadata();
    $score = $doc->getScore();
    echo "[相似度：" . round($score, 3) . "] [{$meta['urgency']}] {$meta['summary']}\n";
    echo "  原文：{$meta->getText()}\n\n";
}
```

> **💡 提示：** `Retriever` 内部会根据 Store 的能力自动选择查询方式：如果 Store 支持 `VectorQuery`，则使用向量搜索；如果支持 `HybridQuery`，则同时使用向量和关键词搜索并可配置权重比。ChromaDb 同时支持 `VectorQuery` 和 `TextQuery`。

### 按条件过滤检索结果

如果只想检索 "negative" 情感的反馈，可以在检索后过滤：

```php
$negativeResults = array_filter(
    iterator_to_array($results),
    fn ($doc) => 'negative' === $doc->getMetadata()['sentiment'],
);
```

---

## Step 6：事件系统——成本监控与日志

批量分析场景下，跟踪每次 AI 调用的成本至关重要。通过事件系统可以无侵入地实现：

```php
<?php

use Symfony\AI\Platform\Event\InvocationEvent;
use Symfony\AI\Platform\Event\ResultEvent;

$totalInputTokens = 0;
$totalOutputTokens = 0;
$callCount = 0;

// 记录每次调用的 Token 用量
$dispatcher->addListener(ResultEvent::class, function (ResultEvent $event) use (&$totalInputTokens, &$totalOutputTokens, &$callCount) {
    $usage = $event->result->getMetadata()->get('token_usage');
    if (null !== $usage) {
        $totalInputTokens += $usage->inputTokens;
        $totalOutputTokens += $usage->outputTokens;
    }
    ++$callCount;
});

// 在 InvocationEvent 中动态调整参数（例如限制输出长度控制成本）
$dispatcher->addListener(InvocationEvent::class, function (InvocationEvent $event) {
    // 对批量分析场景，限制最大输出 Token
    $options = $event->getOptions();
    if (!isset($options['max_tokens'])) {
        $event->setOptions(array_merge($options, ['max_tokens' => 500]));
    }
});
```

处理完成后输出统计：

```php
echo "\n=== 成本统计 ===\n";
echo "总调用次数：{$callCount}\n";
echo "总输入 Token：{$totalInputTokens}\n";
echo "总输出 Token：{$totalOutputTokens}\n";
echo "总 Token：" . ($totalInputTokens + $totalOutputTokens) . "\n";
```

> **🏭 生产建议：** 在正式环境中，应将 Token 统计发送到监控系统（如 Prometheus、Datadog），并设置预算告警。Anthropic Claude 的 `cacheRetention` 参数可以缓存系统提示，对重复的分析模板能节省 30-60% 的输入 Token 费用。

---

## Step 7：趋势报告生成（嵌套 DTO）

用嵌套 DTO 定义复杂的报告结构，AI 会按照层次化 Schema 生成完整的结构化报告。

```php
<?php

namespace App\Dto;

final class FeedbackTrend
{
    /**
     * @param string $area           产品模块
     * @param int    $feedbackCount  反馈数量
     * @param string $mainSentiment  主要情感（positive/neutral/negative）
     * @param string $topIssue       最突出的问题
     */
    public function __construct(
        public readonly string $area,
        public readonly int $feedbackCount,
        public readonly string $mainSentiment,
        public readonly string $topIssue,
    ) {
    }
}

final class WeeklyFeedbackReport
{
    /**
     * @param int             $totalCount      总反馈数
     * @param float           $avgSentiment    平均情感分（-1.0 到 1.0）
     * @param FeedbackTrend[] $trends          各模块趋势
     * @param string[]        $topRequests     热门功能请求（按频率排序）
     * @param string[]        $criticalIssues  紧急问题（需立即处理）
     * @param string          $executiveSummary 给管理层的摘要（2-3 句话）
     */
    public function __construct(
        public readonly int $totalCount,
        public readonly float $avgSentiment,
        public readonly array $trends,
        public readonly array $topRequests,
        public readonly array $criticalIssues,
        public readonly string $executiveSummary,
    ) {
    }
}
```

> **📝 知识扩展：** 嵌套 DTO 是 StructuredOutput 的高级用法。`WeeklyFeedbackReport` 中的 `$trends` 类型为 `FeedbackTrend[]`——`PlatformSubscriber` 会递归生成嵌套的 JSON Schema，包括子对象的所有字段约束。这确保 AI 输出的每一层数据都是类型安全的。

```php
<?php

use App\Dto\WeeklyFeedbackReport;

// 收集本周所有分析结果
$weeklyData = implode("\n", array_map(
    fn (string $feedback, FeedbackAnalysis $analysis) =>
        "- [{$analysis->sentiment}][{$analysis->productArea}][{$analysis->urgency}] {$feedback}",
    $feedbacks,
    $analyses,
));

$messages = new MessageBag(
    Message::forSystem(
        '你是产品运营分析专家。根据本周用户反馈数据，生成结构化周报。'
        . '聚焦：情感趋势、热门问题、功能需求排序、紧急需处理的问题。'
        . '管理层摘要应简洁有力，突出关键数字和行动建议。'
    ),
    Message::ofUser("以下是本周 " . count($feedbacks) . " 条用户反馈：\n\n{$weeklyData}"),
);

// 报告生成使用更强的模型以确保分析质量
$result = $platform->invoke('claude-sonnet', $messages, [
    'response_format' => WeeklyFeedbackReport::class,
]);

/** @var WeeklyFeedbackReport $report */
$report = $result->asObject();

echo "=== 周度反馈报告 ===\n";
echo "总反馈：{$report->totalCount} 条\n";
echo "情感均分：{$report->avgSentiment}\n\n";

echo "📊 各模块趋势：\n";
foreach ($report->trends as $trend) {
    echo "  [{$trend->area}] {$trend->feedbackCount} 条 | {$trend->mainSentiment} | {$trend->topIssue}\n";
}

echo "\n🔥 热门需求：\n";
foreach ($report->topRequests as $request) {
    echo "  💡 {$request}\n";
}

echo "\n🚨 紧急问题：\n";
foreach ($report->criticalIssues as $issue) {
    echo "  ⚠️ {$issue}\n";
}

echo "\n📝 管理层摘要：\n{$report->executiveSummary}\n";
```

---

## 完整示例

以下是将所有步骤串联的完整可运行脚本：

```php
<?php

require 'vendor/autoload.php';

use App\Dto\FeedbackAnalysis;
use App\Dto\WeeklyFeedbackReport;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory as OpenAiPlatformFactory;
use Symfony\AI\Platform\Event\ResultEvent;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\AI\Store\Bridge\ChromaDb\Store;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Indexer;
use Symfony\AI\Store\Retriever;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

// === 1. 初始化 ===

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

// Token 统计
$totalTokens = 0;
$dispatcher->addListener(ResultEvent::class, function (ResultEvent $event) use (&$totalTokens) {
    $usage = $event->result->getMetadata()->get('token_usage');
    if (null !== $usage) {
        $totalTokens += $usage->totalTokens;
    }
});

// Anthropic 用于文本分析
$platform = PlatformFactory::create(
    $_ENV['ANTHROPIC_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);

// OpenAI 用于嵌入
$embeddingPlatform = OpenAiPlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    HttpClient::create(),
);

// ChromaDb 向量存储
$chromaClient = new \Codewithkyrian\ChromaDB\Client('http://localhost:8000');
$store = new Store($chromaClient, 'feedback-analysis');
$store->setup();

$indexer = new Indexer($embeddingPlatform, 'text-embedding-3-small', $store);
$retriever = new Retriever($store, $embeddingPlatform, 'text-embedding-3-small');

// === 2. 批量分析 ===

$feedbacks = [
    '新版本的搜索功能太棒了！终于可以按日期筛选了。',
    '登录页面加载很慢，有时候要等10秒以上。',
    '能不能增加暗黑模式？现在晚上用太刺眼了。',
    '付款后没收到确认邮件，很担心钱是不是白花了。',
    '你们的API文档写得非常清晰，集成只花了半天。',
    '导出功能又崩了！上次反馈还没修吗？',
    '希望支持企业SSO登录，我们公司有安全合规要求。',
    '移动端体验很好，响应速度快，赞一个。',
];

$analyses = [];

foreach ($feedbacks as $i => $feedback) {
    $messages = new MessageBag(
        Message::forSystem('你是产品反馈分析专家。精确分析用户反馈的情感和分类。'),
        Message::ofUser("分析：{$feedback}"),
    );

    $analysis = $platform->invoke('claude-sonnet', $messages, [
        'response_format' => FeedbackAnalysis::class,
    ])->asObject();

    $analyses[] = $analysis;

    $indexer->index([new TextDocument(
        id: 'feedback-' . ($i + 1),
        content: $feedback,
        metadata: new Metadata([
            'sentiment' => $analysis->sentiment,
            'sentiment_score' => $analysis->sentimentScore,
            'category' => $analysis->category,
            'product_area' => $analysis->productArea,
            'urgency' => $analysis->urgency,
            'summary' => $analysis->summary,
        ]),
    )]);

    echo "✅ #{$i}: [{$analysis->sentiment}] {$analysis->summary}\n";
}

// === 3. 语义检索 ===

echo "\n=== 性能相关反馈 ===\n";
foreach ($retriever->retrieve('页面加载慢，性能问题') as $doc) {
    echo "[{$doc->getMetadata()['urgency']}] {$doc->getMetadata()['summary']}\n";
}

// === 4. 趋势报告 ===

$weeklyData = implode("\n", array_map(
    fn (string $fb, FeedbackAnalysis $a) => "- [{$a->sentiment}][{$a->productArea}] {$fb}",
    $feedbacks,
    $analyses,
));

$report = $platform->invoke('claude-sonnet', new MessageBag(
    Message::forSystem('你是产品运营分析专家。根据反馈数据生成周报。'),
    Message::ofUser("本周 " . count($feedbacks) . " 条反馈：\n\n{$weeklyData}"),
), [
    'response_format' => WeeklyFeedbackReport::class,
])->asObject();

echo "\n=== 周报 ===\n";
echo "总反馈：{$report->totalCount} | 情感均分：{$report->avgSentiment}\n";
echo "管理层摘要：{$report->executiveSummary}\n";

echo "\n总 Token 消耗：{$totalTokens}\n";
```

---

## 替代实现方案

### 方案 A：DeepSeek 低成本批量分析

如果反馈量很大（每日数千条以上），可以使用 DeepSeek 降低成本——其 API 价格约为 Anthropic 的 1/10：

```php
use Symfony\AI\Platform\Bridge\DeepSeek\PlatformFactory as DeepSeekPlatformFactory;

$platform = DeepSeekPlatformFactory::create(
    $_ENV['DEEPSEEK_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);

// 业务代码完全不变——MessageBag、StructuredOutput、DTO 全部通用
$result = $platform->invoke('deepseek-chat', $messages, [
    'response_format' => FeedbackAnalysis::class,
]);
```

> **💡 提示：** DeepSeek 在中文反馈分析上的效果与 Anthropic 相当，但英文反馈的情感细粒度稍逊。建议根据你的用户群体语言选择最合适的平台。

### 方案 B：CachePlatform 缓存重复分析

如果同一条反馈会被多个系统重复请求分析（如 API 和定时任务），使用 `CachePlatform` 避免重复调用：

```php
use Symfony\AI\Platform\Bridge\Cache\CachePlatform;
use Symfony\Component\Cache\Adapter\RedisAdapter;

$cache = RedisAdapter::createConnection('redis://localhost');
$cachedPlatform = new CachePlatform($platform, cache: $cache);

// 相同输入的第二次调用直接返回缓存结果
$result = $cachedPlatform->invoke('claude-sonnet', $messages, [
    'response_format' => FeedbackAnalysis::class,
    'prompt_cache_key' => 'feedback-analysis',  // 必须设置才能启用缓存
    'prompt_cache_ttl' => 3600,                  // 缓存 1 小时
]);

// 检查是否命中缓存
$isCached = $result->getMetadata()->get('cached');
echo $isCached ? "📦 缓存命中\n" : "🔄 实时分析\n";
```

### 方案 C：FailoverPlatform 高可用

对于关键业务场景，使用 `FailoverPlatform` 确保服务可用性：

```php
use Symfony\AI\Platform\Bridge\Failover\FailoverPlatformFactory;
use Symfony\Component\RateLimiter\RateLimiterFactory;
use Symfony\Component\RateLimiter\Storage\InMemoryStorage;

$failoverPlatform = FailoverPlatformFactory::create(
    platforms: [
        $anthropicPlatform,   // 首选
        $deepSeekPlatform,    // 备选
    ],
    rateLimiterFactory: new RateLimiterFactory(
        ['id' => 'ai-failover', 'policy' => 'sliding_window', 'limit' => 100, 'interval' => '1 minute'],
        new InMemoryStorage(),
    ),
);
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `PlatformFactory` | 每个 Bridge 提供的静态工厂，一行代码创建 Platform 实例 |
| `PlatformSubscriber` | StructuredOutput 的核心，通过事件系统实现 JSON Schema 注入和反序列化 |
| `FeedbackAnalysis` DTO | 通过 `@param` 注解驱动 Schema 生成，AI 严格按结构输出 |
| 嵌套 DTO | `WeeklyFeedbackReport` 包含 `FeedbackTrend[]`，递归生成层次化 Schema |
| `TextDocument` + `Metadata` | 向量库文档 = 内容 + 结构化元数据，元数据在检索后可直接使用 |
| `Indexer` | 文档索引管线：接收文档 → 调用嵌入 API → 存入向量库 |
| `Retriever` | 语义检索：查询文本 → 向量化 → 在 Store 中搜索最相似文档 |
| `InvocationEvent` / `ResultEvent` | 事件系统贯穿调用链，用于日志、Token 统计、动态参数调整 |
| `CachePlatform` | 缓存包装器，对相同输入避免重复调用 |
| `FailoverPlatform` | 故障转移，多平台自动切换确保可用性 |
| Bridge 混合使用 | 分析用 Anthropic、嵌入用 OpenAI——不同任务选最优平台 |

## 下一步

如果你需要用 AI 自动筛选和匹配简历，请看 [25-ai-recruitment-screening.md](./25-ai-recruitment-screening.md)。
