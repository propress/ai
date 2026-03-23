# 智能客户反馈分析系统

## 业务场景

你在做一个 SaaS 产品。每天用户通过工单、评论、NPS 问卷、App Store 评价等渠道提交大量反馈。产品团队需要：自动分析每条反馈的情感倾向，提取关键意见和功能需求，将相似反馈聚类，发现趋势性问题。传统做法是人工读每条反馈打标签，效率极低。

**典型应用：** 用户反馈情感分析、NPS/CSAT 调研分析、App Store 评价监控、客诉热点发现、产品需求优先级排序

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台 |
| **StructuredOutput** | 反馈分析结果映射为 PHP 对象 |
| **Store** | 向量存储，用于反馈聚类和相似度检索 |
| **Store Reranking** | 相似反馈精确排序 |
| **Agent** | 编排分析流程 |

## 前置准备

```bash
composer require symfony/ai-platform symfony/ai-platform-openai
composer require symfony/ai-agent
composer require symfony/ai-store symfony/ai-store-pinecone  # 或其他向量存储
```

---

## Step 1：定义反馈分析结构

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

---

## Step 2：单条反馈分析

```php
<?php

require 'vendor/autoload.php';

use App\Dto\FeedbackAnalysis;
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

$feedback = '订阅了高级版，但导出PDF功能经常超时，等了3分钟都没反应。花了这么多钱结果核心功能都不能用，太让人失望了。希望尽快修复，不然只能退款了。';

$messages = new MessageBag(
    Message::forSystem(
        '你是产品反馈分析专家。分析用户反馈，提取情感、分类、关键词和行动建议。'
    ),
    Message::ofUser("请分析以下用户反馈：\n\n{$feedback}"),
);

$result = $platform->invoke('gpt-4o-mini', $messages, [
    'response_format' => FeedbackAnalysis::class,
]);

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

---

## Step 3：批量反馈分析与向量化存储

将分析过的反馈存入向量数据库，后续可以做聚类和相似检索。

```php
<?php

use Symfony\AI\Platform\Bridge\OpenAi\Embeddings;
use Symfony\AI\Store\Bridge\Pinecone\Store;
use Symfony\AI\Store\Document\Document;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Indexer;

// 创建嵌入模型和向量存储
$embeddingPlatform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());
$store = new Store(/* Pinecone 配置 */);
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

// 分析每条反馈并存入向量数据库
foreach ($feedbacks as $i => $feedback) {
    // 分析
    $messages = new MessageBag(
        Message::forSystem('你是产品反馈分析专家。'),
        Message::ofUser("分析：{$feedback}"),
    );
    $result = $platform->invoke('gpt-4o-mini', $messages, [
        'response_format' => FeedbackAnalysis::class,
    ]);
    $analysis = $result->asObject();

    // 构建文档存入向量数据库
    $document = new Document(
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
    echo "✅ 已分析反馈 #{$i}: [{$analysis->sentiment}] {$analysis->summary}\n";
}
```

---

## Step 4：相似反馈检索与 Reranking

当产品经理想了解某个问题的所有相关反馈时，用向量检索 + Reranking 精准定位。

```php
<?php

use Symfony\AI\Store\Query\VectorQuery;
use Symfony\AI\Store\Retriever;

$retriever = new Retriever($embeddingPlatform, 'text-embedding-3-small', $store);

// 产品经理想查：有哪些关于性能问题的反馈？
$query = new VectorQuery('页面加载慢，性能问题', maxResults: 10);
$results = $retriever->retrieve($query);

echo "=== 性能相关反馈 ===\n";
foreach ($results as $result) {
    $meta = $result->metadata;
    echo "[{$meta['urgency']}] {$meta['summary']}\n";
    echo "  原文：{$result->content}\n\n";
}
```

---

## Step 5：反馈趋势报告

用 Agent 自动汇总一周的反馈，生成趋势报告。

```php
<?php

namespace App\Dto;

final class FeedbackTrend
{
    /**
     * @param string $area           产品模块
     * @param int    $feedbackCount  反馈数量
     * @param string $mainSentiment  主要情感
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
     * @param float           $avgSentiment    平均情感分
     * @param FeedbackTrend[] $trends          各模块趋势
     * @param string[]        $topRequests     热门功能请求
     * @param string[]        $criticalIssues  紧急问题
     * @param string          $executiveSummary 给管理层的摘要
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

```php
<?php

use App\Dto\WeeklyFeedbackReport;

// 收集本周所有分析结果（从数据库加载）
$weeklyData = implode("\n", array_map(
    fn ($feedback, $analysis) => "- [{$analysis->sentiment}][{$analysis->productArea}] {$feedback}",
    $feedbacks,
    $analyses,
));

$messages = new MessageBag(
    Message::forSystem(
        '你是产品运营分析专家。根据本周用户反馈数据，生成周报。'
        . '聚焦：情感趋势、热门问题、功能需求排序、紧急需处理的问题。'
    ),
    Message::ofUser("以下是本周 " . count($feedbacks) . " 条用户反馈：\n\n{$weeklyData}"),
);

$result = $platform->invoke('gpt-4o', $messages, [
    'response_format' => WeeklyFeedbackReport::class,
]);

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

## 完整流程

```
用户反馈（工单/评价/问卷）
    │
    ▼
[AI 结构化分析] → FeedbackAnalysis 对象
    │                ├─ 情感 + 评分
    │                ├─ 分类 + 模块
    │                ├─ 关键词
    │                └─ 行动建议
    ▼
[向量化存储] → Pinecone/Qdrant
    │
    ├──► [相似反馈检索] + Reranking → 精准定位
    │
    └──► [趋势汇总] → WeeklyFeedbackReport
              ├─ 各模块趋势
              ├─ 热门功能请求
              ├─ 紧急问题
              └─ 管理层摘要
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `FeedbackAnalysis` DTO | 情感、分类、关键词、紧急度一次性提取 |
| `Document` + `Metadata` | 反馈内容 + 分析标签一起存入向量库 |
| `Indexer` | 自动嵌入并索引文档 |
| `VectorQuery` | 语义搜索相似反馈 |
| `Retriever` | 从向量库中检索相关文档 |
| Reranking | 对检索结果精确排序 |
| 趋势报告 DTO | `WeeklyFeedbackReport` 嵌套 `FeedbackTrend` |

## 下一步

如果你需要用 AI 自动筛选和匹配简历，请看 [25-ai-recruitment-screening.md](./25-ai-recruitment-screening.md)。
