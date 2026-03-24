# 智能客户反馈分析系统

## 业务场景

你在做一个 SaaS 产品。每天用户通过工单、评论、NPS 问卷、App Store 评价等渠道提交大量反馈。产品团队需要：自动分析每条反馈的情感倾向，提取关键意见和功能需求，将相似反馈聚类，发现趋势性问题。传统做法是人工逐条阅读、手动打标签，效率极低且容易遗漏关键信号。

本指南使用 **StructuredOutput**（结构化输出）让 AI 将分析结果映射为强类型 PHP 对象，同时使用 **Store**（向量存储）对反馈进行语义索引和检索——两者结合实现从"分析"到"洞察"的完整闭环。

**典型应用：** 用户反馈情感分析、NPS/CSAT 调研分析、App Store 评价监控、客诉热点发现、产品需求优先级排序、客户流失预警

## 涉及模块

| 模块 | 包名 | 用途 |
|------|------|------|
| **Platform** | `symfony/ai-platform` | 核心抽象层——`PlatformInterface`、消息系统、事件系统 |
| **Bridge（Anthropic）** | `symfony/ai-anthropic-platform` | 主分析平台，使用 Claude 进行情感分析和报告生成 |
| **Bridge（OpenAI）** | `symfony/ai-open-ai-platform` | 嵌入模型平台，使用 text-embedding-3-small 生成向量 |
| **StructuredOutput** | `symfony/ai-platform`（内置） | 将 LLM 输出映射为 PHP DTO 对象 |
| **Store** | `symfony/ai-store` | 向量存储抽象层——`StoreInterface`、`Vectorizer`、`Retriever` |
| **Store Bridge（Postgres）** | `symfony/ai-postgres-store` | PostgreSQL pgvector 向量数据库实现 |

## 架构概述

本项目的核心架构由两个子系统协同工作：

### StructuredOutput 子系统

StructuredOutput 基于 **EventSubscriber 模式**，通过 `PlatformSubscriber` 拦截 Platform 的调用过程：

1. **请求阶段（InvocationEvent）**：检测 `response_format` 选项中的 PHP 类名，使用 `ResponseFormatFactory` 将该类的属性和 PHPDoc 类型信息转换为 JSON Schema，注入到 LLM 请求中
2. **响应阶段（ResultEvent）**：拦截 LLM 返回的 JSON 字符串，使用 Serializer 反序列化为目标 PHP 对象

### Store 子系统

Store 提供统一的向量存储和检索抽象：

1. **向量化**：`Vectorizer` 调用嵌入模型（如 OpenAI text-embedding-3-small）将文本转换为向量
2. **存储**：`StoreInterface` 实现（如 PostgreSQL pgvector）持久化 `VectorDocument`
3. **检索**：`Retriever` 自动根据 Store 能力选择最优查询类型（`VectorQuery`、`TextQuery` 或 `HybridQuery`）

两个子系统通过 **不同的 Platform 实例**协作——分析用 Anthropic Claude（擅长理解和推理），嵌入用 OpenAI（text-embedding-3-small 性价比最优）。

> **🏭 生产建议：** 在 Symfony 项目中推荐通过 AI Bundle（`symfony/ai-bundle`）自动配置所有 Platform 实例和 Store 服务，通过依赖注入获取，而不是手动调用工厂方法。

## 项目流程图

```
┌──────────┐     ┌───────────────────┐     ┌──────────────────────┐
│ 用户反馈   │ ──▶│ Anthropic Claude   │ ──▶│ StructuredOutput      │
│(工单/评价) │     │ 情感分析 + 分类    │     │ → FeedbackAnalysis DTO│
└──────────┘     └───────────────────┘     └──────────┬───────────┘
                                                       │
                                                       ▼
┌──────────────────────┐     ┌──────────────┐     ┌──────────────────┐
│ Vectorizer            │ ──▶│ OpenAI        │ ──▶│ TextDocument       │
│ 文本 → 向量           │     │ Embeddings   │     │ → VectorDocument   │
└──────────────────────┘     └──────────────┘     └────────┬─────────┘
                                                            │
                                                            ▼
┌──────────────────────────────────────────────────────────────────┐
│                    PostgreSQL pgvector Store                      │
│  ┌─────────────────┐    ┌─────────────────┐    ┌──────────────┐  │
│  │ VectorDocument   │    │ VectorDocument   │    │ VectorDocument│  │
│  │ + Metadata       │    │ + Metadata       │    │ + Metadata   │  │
│  └─────────────────┘    └─────────────────┘    └──────────────┘  │
└──────────────────────────────┬───────────────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                 ▼
     ┌──────────────┐  ┌────────────┐  ┌──────────────────┐
     │ Retriever     │  │ 直接查询    │  │ Anthropic Claude  │
     │ 语义检索      │  │ VectorQuery │  │ 趋势报告生成      │
     │ (自动选择     │  │ TextQuery   │  │ → WeeklyReport DTO│
     │  查询类型)    │  │ HybridQuery │  └──────────────────┘
     └──────────────┘  └────────────┘
```

> **📝 知识扩展：** `PlatformSubscriber` 是一个实现了 `EventSubscriberInterface` 的类，它订阅了两个事件：`InvocationEvent::class => 'processInput'` 和 `ResultEvent::class => 'processResult'`。在 `processInput` 中，它检查 `$options['response_format']` 是否为类名（而非 `null` 或已解析的 schema），如果是，就调用 `ResponseFormatFactory::create()` 生成 JSON Schema 并替换选项值。在 `processResult` 中，它将 LLM 返回的 JSON 字符串通过 Serializer 反序列化为目标对象，赋值到 `DeferredResult` 中供 `asObject()` 返回。这种 EventSubscriber 模式意味着你的业务代码完全无感知——只需传入类名，拿到对象。

## 前置准备

### 环境要求

- PHP >= 8.2
- Composer
- PostgreSQL 带 pgvector 扩展（`CREATE EXTENSION vector;`）

### 安装依赖

```bash
# 核心平台 + Anthropic（分析）+ OpenAI（嵌入）
composer require symfony/ai-platform symfony/ai-anthropic-platform symfony/ai-open-ai-platform

# 向量存储
composer require symfony/ai-store symfony/ai-postgres-store
```

### 设置 API 密钥

```bash
export ANTHROPIC_API_KEY="sk-ant-your-api-key-here"
export OPENAI_API_KEY="sk-your-api-key-here"
```

> **🔒 安全建议：** 永远不要将 API 密钥硬编码在源代码中。在 Symfony 项目中，应使用 `.env.local` 文件或服务器环境变量来管理密钥；在 CI/CD 中使用 Secrets 管理。

### 初始化 PostgreSQL 表

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE feedback_vectors (
    id VARCHAR(255) PRIMARY KEY,
    embedding vector(1536),  -- text-embedding-3-small 输出 1536 维
    metadata JSONB
);

-- lists 值建议根据数据量调整：rows / 1000（数据量 < 1M 时用 100 即可）
CREATE INDEX ON feedback_vectors USING ivfflat (embedding vector_l2_ops) WITH (lists = 100);
```

---

## Step 1：定义反馈分析 DTO

StructuredOutput 的关键在于定义一个 PHP DTO 类，其属性类型和 `@param` PHPDoc 会被 `ResponseFormatFactory` 转换为 JSON Schema，引导 LLM 按照精确的格式输出。

```php
<?php

namespace App\Dto;

final class FeedbackAnalysis
{
    /**
     * @param string   $sentiment      情感倾向（positive/neutral/negative）
     * @param float    $sentimentScore 情感评分（-1.0 到 1.0）
     * @param string   $category       反馈类别（bug/feature_request/complaint/praise/question）
     * @param string   $summary        反馈摘要（一句话）
     * @param string[] $keywords       关键词列表
     * @param string   $productArea    涉及产品模块（ui/performance/billing/api/onboarding/other）
     * @param string   $urgency        紧急程度（critical/high/medium/low）
     * @param string   $actionItem     建议的行动项
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

> **📝 知识扩展：** `ResponseFormatFactory` 内部使用 `Factory`（Schema Factory）来解析 PHP 类的元数据。它会读取构造函数参数的原生类型（`string`、`float`、`array`）以及 PHPDoc 中的 `@param` 泛型标注（如 `string[]`）来生成精确的 JSON Schema。例如，`string[] $keywords` 会被转换为 `{"type": "array", "items": {"type": "string"}}`。生成的 Schema 会带上 `"strict": true`，确保 LLM 严格遵循格式。这也是为什么 `@param` PHPDoc 如此重要——没有它，`array` 类型会被当作 `object` 处理。

---

## Step 2：单条反馈分析（Anthropic Claude）

使用 Anthropic Claude 作为分析模型——它在理解复杂语义、识别情感细微差异方面表现优异。

```php
<?php

require 'vendor/autoload.php';

use App\Dto\FeedbackAnalysis;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

// 1. 注册 PlatformSubscriber——必须在创建 Platform 之前
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

// 2. 创建 Anthropic Platform，传入事件调度器
$platform = PlatformFactory::create(
    $_ENV['ANTHROPIC_API_KEY'],
    eventDispatcher: $dispatcher,
);

// 3. 构建消息
$feedback = '订阅了高级版，但导出PDF功能经常超时，等了3分钟都没反应。'
    . '花了这么多钱结果核心功能都不能用，太让人失望了。希望尽快修复，不然只能退款了。';

$messages = new MessageBag(
    Message::forSystem(
        '你是产品反馈分析专家。请仔细分析用户反馈，提取情感倾向、分类、关键词和行动建议。'
        . '情感评分范围 -1.0（极度不满）到 1.0（极度满意），0 为中性。'
    ),
    Message::ofUser("请分析以下用户反馈：\n\n{$feedback}"),
);

// 4. 调用 AI——response_format 传入 DTO 类名
$result = $platform->invoke('claude-sonnet-4-20250514', $messages, [
    'response_format' => FeedbackAnalysis::class,
]);

// 5. asObject() 返回 FeedbackAnalysis 实例
$analysis = $result->asObject();

echo "=== 反馈分析结果 ===\n";
echo "情感：{$analysis->sentiment}（{$analysis->sentimentScore}）\n";
echo "类别：{$analysis->category}\n";
echo "模块：{$analysis->productArea}\n";
echo "紧急：{$analysis->urgency}\n";
echo "摘要：{$analysis->summary}\n";
echo "关键词：" . implode('、', $analysis->keywords) . "\n";
echo "建议：{$analysis->actionItem}\n";

// 获取 Token 用量
$tokenUsage = $result->getMetadata()->get('token_usage');
if (null !== $tokenUsage) {
    echo sprintf(
        "\n[Token 用量] 输入: %d, 输出: %d, 合计: %d\n",
        $tokenUsage->getPromptTokens(),
        $tokenUsage->getCompletionTokens(),
        $tokenUsage->getTotalTokens(),
    );
}
```

### PlatformSubscriber 构造参数

`PlatformSubscriber` 接受两个可选参数，大多数场景使用默认值即可：

```php
public function __construct(
    private readonly ResponseFormatFactoryInterface $responseFormatFactory = new ResponseFormatFactory(),
    ?SerializerInterface $serializer = null,  // 默认使用内置 Serializer
)
```

如果你的 DTO 需要自定义反序列化逻辑（如日期格式转换），可以传入自定义 Serializer。

> **💡 提示：** `PlatformSubscriber` 必须在创建 Platform 之前注册到 EventDispatcher。因为 `PlatformFactory::create()` 会将 EventDispatcher 注入到 Platform 实例中，之后的每次 `invoke()` 都会触发事件。如果先创建 Platform 再注册 Subscriber，事件将无法被拦截。

---

## Step 3：向量化存储——TextDocument + Vectorizer + Store

分析完成后，将反馈文本和分析结果存入向量数据库，为后续的语义检索做准备。这里涉及三个核心概念：

- **`TextDocument`**：文本文档，包含 ID、内容和元数据
- **`Vectorizer`**：将文本（或 TextDocument）转换为向量
- **`StoreInterface`**：向量数据库操作接口

### 创建基础设施

```php
<?php

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory as OpenAiPlatformFactory;
use Symfony\AI\Store\Bridge\Postgres\Store;
use Symfony\AI\Store\Document\Vectorizer;

// 嵌入模型用 OpenAI——text-embedding-3-small 性价比最优
$embeddingPlatform = OpenAiPlatformFactory::create($_ENV['OPENAI_API_KEY']);

// 创建 Vectorizer：平台 + 模型名
$vectorizer = new Vectorizer(
    platform: $embeddingPlatform,
    model: 'text-embedding-3-small',
);

// 创建 PostgreSQL pgvector Store
$pdo = new \PDO('pgsql:host=localhost;dbname=feedback', 'user', 'password');
$store = new Store(
    connection: $pdo,
    tableName: 'feedback_vectors',
    vectorFieldName: 'embedding',
);
```

### 构建 TextDocument 并向量化

```php
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Metadata;

// 模拟批量反馈（实际场景从数据库或消息队列获取）
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

// 逐条分析并构建文档
$textDocuments = [];
foreach ($feedbacks as $i => $feedback) {
    // 用 Anthropic 分析
    $messages = new MessageBag(
        Message::forSystem('你是产品反馈分析专家。'),
        Message::ofUser("分析：{$feedback}"),
    );
    $result = $platform->invoke('claude-sonnet-4-20250514', $messages, [
        'response_format' => FeedbackAnalysis::class,
    ]);
    $analysis = $result->asObject();

    // 构建 TextDocument（注意：是 TextDocument，不是 Document）
    $textDocuments[] = new TextDocument(
        id: 'feedback-' . ($i + 1),
        content: $feedback,
        metadata: new Metadata([
            Metadata::KEY_SOURCE => 'user_feedback',
            'sentiment' => $analysis->sentiment,
            'sentiment_score' => $analysis->sentimentScore,
            'category' => $analysis->category,
            'product_area' => $analysis->productArea,
            'urgency' => $analysis->urgency,
            'summary' => $analysis->summary,
            'keywords' => implode(',', $analysis->keywords),
        ]),
    );

    echo "✅ 已分析反馈 #" . ($i + 1) . ": [{$analysis->sentiment}] {$analysis->summary}\n";
}
```

### 批量向量化并存入 Store

`Vectorizer` 支持批量操作——传入数组即可一次性向量化多个文档，减少 API 调用次数：

```php
// 批量向量化：TextDocument[] → VectorDocument[]
$vectorDocuments = $vectorizer->vectorize($textDocuments);

// 批量存入 Store
$store->add($vectorDocuments);

echo sprintf("\n🎉 已存入 %d 条反馈向量\n", count($vectorDocuments));
```

> **📝 知识扩展：** `Metadata` 类继承自 `\ArrayObject`，提供了 6 个预定义常量键：`KEY_PARENT_ID`（父文档 ID，用于分层文档）、`KEY_TEXT`（原始文本）、`KEY_SOURCE`（来源标识）、`KEY_SUMMARY`（摘要）、`KEY_TITLE`（标题）、`KEY_DEPTH`（层级深度）。这些键以 `_` 前缀命名（如 `_source`），用于框架内部逻辑。你也可以自由添加业务字段（如 `sentiment`、`category`），它们和预定义键共存于同一个 `ArrayObject` 中。`Metadata` 还提供了 `hasText()`、`getText()`、`setText()` 等便捷方法来操作预定义键。

---

## Step 4：语义检索——Retriever 与查询类型

### 使用 Retriever（推荐方式）

`Retriever` 是最常用的检索入口——它封装了向量化查询和 Store 查询的完整流程。你只需传入文本字符串，`Retriever` 会自动根据 Store 的能力选择最优查询类型：

```php
<?php

use Symfony\AI\Store\Retriever;

// 创建 Retriever：Store + Vectorizer
$retriever = new Retriever(
    store: $store,
    vectorizer: $vectorizer,
);

// 产品经理想查：有哪些关于性能问题的反馈？
// retrieve() 接受纯字符串，内部自动向量化 + 选择查询类型
$results = $retriever->retrieve('页面加载慢，性能问题', [
    'maxResults' => 10,
]);

echo "=== 性能相关反馈 ===\n";
foreach ($results as $doc) {
    $meta = $doc->getMetadata();
    $score = $doc->getScore();
    echo sprintf("[%s][相似度 %.2f] %s\n", $meta['urgency'], $score, $meta['summary']);
    echo "  原文：{$meta[Metadata::KEY_TEXT]}\n\n";
}
```

> **📝 知识扩展：** `Retriever::retrieve()` 内部的 `createQuery()` 方法使用以下决策逻辑来选择查询类型：①如果没有配置 Vectorizer → 使用 `TextQuery`（纯文本匹配）；②如果 Store 不支持 `VectorQuery` → 退回 `TextQuery`；③如果 Store 支持 `HybridQuery` → 使用 `HybridQuery`（向量 + 文本混合，默认 `semanticRatio=0.5`）；④否则 → 使用 `VectorQuery`（纯向量匹配）。PostgreSQL pgvector Store 支持 `VectorQuery` 和 `TextQuery`，但不支持 `HybridQuery`，所以配合 Vectorizer 时会自动选择 `VectorQuery`。Meilisearch、Typesense 等全文搜索引擎支持 `HybridQuery`。

### 直接使用 Store 查询（高级用法）

如果你需要更精细的控制（如自定义向量、元数据过滤），可以直接使用 Store 的 `query()` 方法：

```php
use Symfony\AI\Store\Query\VectorQuery;

// 手动向量化查询文本
$queryVector = $vectorizer->vectorize('导出功能 PDF 问题');

// VectorQuery 构造函数接受 Vector 对象（不是字符串）
$query = new VectorQuery(vector: $queryVector);

// 直接查询 Store
$results = $store->query($query, [
    'maxResults' => 5,
]);

foreach ($results as $doc) {
    echo sprintf(
        "[%s] %s (score: %.4f)\n",
        $doc->getMetadata()['category'],
        $doc->getMetadata()['summary'],
        $doc->getScore(),
    );
}
```

### 带元数据过滤的查询

不同 Store 实现支持通过 `options` 传入元数据过滤条件，具体语法取决于底层数据库：

```php
// PostgreSQL pgvector——通过 options 传入 SQL WHERE 条件
$results = $store->query($query, [
    'maxResults' => 10,
    'filter' => "metadata->>'urgency' IN ('critical', 'high')",
]);

echo "=== 紧急/高优先级反馈 ===\n";
foreach ($results as $doc) {
    $meta = $doc->getMetadata();
    echo "[{$meta['urgency']}] {$meta['summary']}\n";
}
```

> **⚠️ 注意：** 元数据过滤的语法因 Store 实现而异。PostgreSQL 使用 JSONB 查询语法，Qdrant 使用 Filter 对象，Meilisearch 使用字符串表达式。请参考各 Store Bridge 的文档了解具体用法。

---

## Step 5：反馈趋势报告（嵌套 DTO）

StructuredOutput 支持嵌套 DTO——通过 `@param` PHPDoc 中的泛型数组标注，`ResponseFormatFactory` 会递归生成嵌套的 JSON Schema。

```php
<?php

namespace App\Dto;

final class FeedbackTrend
{
    /**
     * @param string $area          产品模块
     * @param int    $feedbackCount 反馈数量
     * @param string $mainSentiment 主要情感倾向
     * @param string $topIssue      最突出的问题
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
     * @param int             $totalCount       总反馈数
     * @param float           $avgSentiment     平均情感分
     * @param FeedbackTrend[] $trends           各模块趋势
     * @param string[]        $topRequests      热门功能请求
     * @param string[]        $criticalIssues   紧急问题
     * @param string          $executiveSummary  给管理层的摘要
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
    fn (string $feedback, FeedbackAnalysis $analysis) =>
        "- [{$analysis->sentiment}][{$analysis->productArea}][{$analysis->urgency}] {$feedback}",
    $feedbacks,
    $analyses,
));

$messages = new MessageBag(
    Message::forSystem(
        '你是产品运营分析专家。根据本周用户反馈数据，生成结构化周报。'
        . '聚焦：情感趋势、各模块问题分布、热门功能请求排序、紧急需处理的问题。'
        . '管理层摘要控制在 3-5 句话。'
    ),
    Message::ofUser("以下是本周 " . count($feedbacks) . " 条用户反馈及分析标签：\n\n{$weeklyData}"),
);

$result = $platform->invoke('claude-sonnet-4-20250514', $messages, [
    'response_format' => WeeklyFeedbackReport::class,
]);

$report = $result->asObject();

echo "=== 周度反馈报告 ===\n";
echo "总反馈：{$report->totalCount} 条\n";
echo sprintf("情感均分：%.2f\n\n", $report->avgSentiment);

echo "📊 各模块趋势：\n";
foreach ($report->trends as $trend) {
    echo sprintf(
        "  [%s] %d 条 | %s | %s\n",
        $trend->area,
        $trend->feedbackCount,
        $trend->mainSentiment,
        $trend->topIssue,
    );
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

> **💡 提示：** 嵌套 DTO（如 `FeedbackTrend[] $trends`）的 `@param` 标注必须使用完全限定类名或在同一命名空间下的短类名。`ResponseFormatFactory` 会递归解析每个嵌套类的属性，生成完整的嵌套 JSON Schema。对于简单列表（如 `string[] $keywords`），PHPDoc 标注就足够了。

---

## Step 6：CachePlatform——缓存重复分析

对于相同反馈文本的重复分析（如定时任务重跑、UI 刷新），使用 `CachePlatform` 包装可以避免重复 API 调用：

```php
<?php

use Symfony\AI\Platform\Bridge\Cache\CachePlatform;
use Symfony\Component\Cache\Adapter\TagAwareAdapter;
use Symfony\Component\Cache\Adapter\FilesystemAdapter;

// 用 TagAwareAdapter 包装 FilesystemAdapter
$cacheAdapter = new TagAwareAdapter(new FilesystemAdapter('ai_cache', 3600));

$cachedPlatform = new CachePlatform(
    platform: $platform,         // 原始 Anthropic Platform
    cache: $cacheAdapter,
    cacheTtl: 86400,             // 缓存 24 小时
);

// 使用 cachedPlatform 替代 platform——相同输入不会重复调用 API
$result = $cachedPlatform->invoke('claude-sonnet-4-20250514', $messages, [
    'response_format' => FeedbackAnalysis::class,
]);

$analysis = $result->asObject();
```

> **🏭 生产建议：** `CachePlatform` 基于请求内容的哈希值做缓存键，因此只有完全相同的模型+消息+选项才会命中缓存。在批量处理场景中，它可以显著降低因程序中断重跑时的 API 费用。配合 `TagAwareAdapter`，你还可以按标签（如日期、批次）批量清除缓存。

---

## Step 7：检索驱动的深度分析

将 Retriever 的检索结果作为上下文，交给 Claude 做更深层的分析——这是 RAG（检索增强生成）模式在反馈分析中的应用：

```php
<?php

// 检索关于「性能」的反馈
$performanceFeedbacks = $retriever->retrieve('性能 加载速度 响应慢 超时', [
    'maxResults' => 20,
]);

// 将检索结果拼接为上下文
$context = '';
foreach ($performanceFeedbacks as $doc) {
    $meta = $doc->getMetadata();
    $context .= sprintf(
        "- [%s][%s] %s\n",
        $meta['urgency'],
        $meta['category'],
        $meta[Metadata::KEY_TEXT] ?? $meta['summary'],
    );
}

// 用 Claude 做根因分析
$messages = new MessageBag(
    Message::forSystem(
        '你是技术运营专家。根据以下用户反馈，分析性能问题的根本原因，'
        . '按严重程度排序，并给出技术建议。'
    ),
    Message::ofUser("以下是关于性能问题的用户反馈：\n\n{$context}"),
);

$result = $platform->invoke('claude-sonnet-4-20250514', $messages);
echo "=== 性能问题根因分析 ===\n";
echo $result->asText() . "\n";
```

---

## 完整示例

以下是一个完整的独立脚本，包含从反馈分析到存储到检索的全流程：

```php
<?php

require 'vendor/autoload.php';

use App\Dto\FeedbackAnalysis;
use App\Dto\WeeklyFeedbackReport;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory as AnthropicFactory;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory as OpenAiFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\AI\Store\Bridge\Postgres\Store;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Retriever;
use Symfony\Component\EventDispatcher\EventDispatcher;

// === 1. 初始化平台 ===

// Anthropic 用于分析（注册 StructuredOutput 事件订阅器）
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$analysisPlatform = AnthropicFactory::create(
    $_ENV['ANTHROPIC_API_KEY'],
    eventDispatcher: $dispatcher,
);

// OpenAI 用于嵌入
$embeddingPlatform = OpenAiFactory::create($_ENV['OPENAI_API_KEY']);
$vectorizer = new Vectorizer($embeddingPlatform, 'text-embedding-3-small');

// PostgreSQL pgvector Store
$pdo = new \PDO('pgsql:host=localhost;dbname=feedback', 'user', 'password');
$store = new Store($pdo, 'feedback_vectors');

// Retriever
$retriever = new Retriever($store, $vectorizer);

// === 2. 批量分析反馈 ===

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

$textDocuments = [];
$analyses = [];
foreach ($feedbacks as $i => $feedback) {
    $messages = new MessageBag(
        Message::forSystem(
            '你是产品反馈分析专家。分析用户反馈，提取情感、分类、关键词和行动建议。'
            . '情感评分范围 -1.0（极度不满）到 1.0（极度满意）。'
        ),
        Message::ofUser("分析以下反馈：\n\n{$feedback}"),
    );

    $result = $analysisPlatform->invoke('claude-sonnet-4-20250514', $messages, [
        'response_format' => FeedbackAnalysis::class,
    ]);
    $analysis = $result->asObject();
    $analyses[] = $analysis;

    $textDocuments[] = new TextDocument(
        id: 'feedback-' . ($i + 1),
        content: $feedback,
        metadata: new Metadata([
            Metadata::KEY_SOURCE => 'user_feedback',
            'sentiment' => $analysis->sentiment,
            'sentiment_score' => $analysis->sentimentScore,
            'category' => $analysis->category,
            'product_area' => $analysis->productArea,
            'urgency' => $analysis->urgency,
            'summary' => $analysis->summary,
        ]),
    );

    echo "✅ #{$i}: [{$analysis->sentiment}] {$analysis->summary}\n";
}

// === 3. 批量向量化并存储 ===

$vectorDocuments = $vectorizer->vectorize($textDocuments);
$store->add($vectorDocuments);
echo sprintf("\n🎉 已存入 %d 条反馈向量\n\n", count($vectorDocuments));

// === 4. 语义检索 ===

echo "=== 检索：性能相关反馈 ===\n";
$results = $retriever->retrieve('页面加载慢 性能问题 超时');
foreach ($results as $doc) {
    $meta = $doc->getMetadata();
    echo sprintf("[%s] %s\n", $meta['urgency'], $meta['summary']);
}

// === 5. 生成周报 ===

$weeklyData = implode("\n", array_map(
    fn (string $fb, FeedbackAnalysis $a) =>
        "- [{$a->sentiment}][{$a->productArea}][{$a->urgency}] {$fb}",
    $feedbacks,
    $analyses,
));

$messages = new MessageBag(
    Message::forSystem('你是产品运营分析专家。根据反馈数据生成结构化周报。'),
    Message::ofUser("本周 " . count($feedbacks) . " 条反馈：\n\n{$weeklyData}"),
);

$result = $analysisPlatform->invoke('claude-sonnet-4-20250514', $messages, [
    'response_format' => WeeklyFeedbackReport::class,
]);

$report = $result->asObject();
echo "\n=== 周度反馈报告 ===\n";
echo "总反馈：{$report->totalCount} 条 | 情感均分：{$report->avgSentiment}\n";
foreach ($report->trends as $trend) {
    echo "  [{$trend->area}] {$trend->feedbackCount} 条 | {$trend->mainSentiment}\n";
}
echo "\n📝 管理层摘要：\n{$report->executiveSummary}\n";
```

---

## 替代实现方案

### Gemini——Google Vertex AI 替代

如果需要使用 Google 生态系统，Gemini 同样支持 StructuredOutput：

```bash
composer require symfony/ai-gemini-platform
```

```php
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$platform = PlatformFactory::create(
    $_ENV['GEMINI_API_KEY'],
    eventDispatcher: $dispatcher,
);

$result = $platform->invoke('gemini-2.5-flash', $messages, [
    'response_format' => FeedbackAnalysis::class,
]);

$analysis = $result->asObject();
```

### Mistral——欧洲合规方案

对于有 GDPR 严格要求的场景，Mistral AI（法国公司）提供欧洲托管的模型：

```bash
composer require symfony/ai-mistral-platform
```

```php
use Symfony\AI\Platform\Bridge\Mistral\PlatformFactory;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$platform = PlatformFactory::create(
    $_ENV['MISTRAL_API_KEY'],
    eventDispatcher: $dispatcher,
);

$result = $platform->invoke('mistral-large-latest', $messages, [
    'response_format' => FeedbackAnalysis::class,
]);
```

### 替代向量数据库

Symfony AI Store 提供了 24 种向量数据库 Bridge，可根据基础设施选择：

| Store Bridge | 包名 | 适用场景 |
|-------------|------|---------|
| **Qdrant** | `symfony/ai-qdrant-store` | 专用向量数据库，元数据过滤强大 |
| **ChromaDb** | `symfony/ai-chroma-db-store` | 轻量开发/测试环境 |
| **Meilisearch** | `symfony/ai-meilisearch-store` | 全文搜索 + 向量混合查询（支持 HybridQuery） |
| **Redis** | `symfony/ai-redis-store` | 已有 Redis 基础设施时 |
| **Elasticsearch** | `symfony/ai-elasticsearch-store` | 已有 ELK 栈时 |
| **Sqlite** | `symfony/ai-sqlite-store` | 单机嵌入式场景 |

```php
// 示例：切换到 Qdrant
use Symfony\AI\Store\Bridge\Qdrant\Store;

$store = new Store(
    client: $qdrantClient,
    collectionName: 'feedbacks',
);

// StoreInterface 统一——Retriever 和查询代码无需修改
$retriever = new Retriever($store, $vectorizer);
```

### 生产环境增强

- **`CachePlatform`**：缓存相同请求的响应，减少重复调用的 API 费用（见 Step 6）
- **`FailoverPlatform`**：在多个平台之间自动故障转移——当 Anthropic 不可用时自动切换到 OpenAI

```php
use Symfony\AI\Platform\Bridge\Failover\FailoverPlatform;

$failoverPlatform = new FailoverPlatform(
    platforms: [$anthropicPlatform, $openAiPlatform, $geminiPlatform],
    rateLimiterFactory: $rateLimiterFactory,
);
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `PlatformSubscriber` | EventSubscriber，订阅 `InvocationEvent` 和 `ResultEvent`，实现 StructuredOutput 的请求注入和响应反序列化 |
| `ResponseFormatFactory` | 将 PHP DTO 类的属性类型和 PHPDoc 转换为 JSON Schema，发送给 LLM 约束输出格式 |
| `response_format` 选项 | `invoke()` 的选项，传入 DTO 类名（如 `FeedbackAnalysis::class`），触发 StructuredOutput 流程 |
| `DeferredResult::asObject()` | 返回 StructuredOutput 反序列化后的 PHP 对象实例 |
| `TextDocument` | 文本文档类（不是 `Document`），包含 `id`、`content`、`metadata`，实现 `EmbeddableDocumentInterface` |
| `VectorDocument` | 向量文档类，包含 `id`、`vector`（`VectorInterface`）、`metadata`、`score` |
| `Metadata` | 继承 `\ArrayObject`，预定义键常量：`KEY_TEXT`、`KEY_SOURCE`、`KEY_SUMMARY`、`KEY_TITLE`、`KEY_PARENT_ID`、`KEY_DEPTH` |
| `Vectorizer` | 将文本或 `TextDocument`（支持数组批量）转换为 `Vector` 或 `VectorDocument`，内部调用嵌入模型 |
| `Retriever` | 高级检索接口，`retrieve(string)` 自动向量化查询并根据 Store 能力选择查询类型 |
| `VectorQuery` | 纯向量查询，构造函数接受 `Vector` 对象（不是字符串） |
| `TextQuery` | 纯文本查询，接受 `string\|array<string>`，多个文本以 OR 逻辑组合 |
| `HybridQuery` | 混合查询（向量+文本），`semanticRatio` 控制语义搜索占比（0.0-1.0） |
| `StoreInterface` | 向量存储统一接口：`add()`、`remove()`、`query()`、`supports()`，24 种数据库实现 |
| `CachePlatform` | 包装 Platform，缓存相同请求的响应，减少重复 API 调用 |
| DTO 嵌套模式 | `FeedbackTrend[] $trends` 通过 PHPDoc 泛型标注实现嵌套 DTO，`ResponseFormatFactory` 递归生成 JSON Schema |

## 下一步

如果你需要用 AI 自动筛选和匹配简历，请看 [25-ai-recruitment-screening.md](./25-ai-recruitment-screening.md)。
