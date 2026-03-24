# 智能客户反馈分析系统

## 项目概述

本教程构建一个**生产级客户反馈智能分析系统**——从结构化情感分析、批量处理、向量化存储、相似反馈聚类，到基于 Agent 的趋势报告生成。你将掌握 Symfony AI 中 StructuredOutput、Store、Reranker 和 Agent 的实战组合用法。

与其他教程不同，本教程**不使用 OpenAI 作为主平台**。我们使用 **Anthropic Claude** 作为主分析引擎、**Mistral** 生成 Embedding 向量、**PostgreSQL pgvector** 存储反馈向量，并展示 **CachePlatform** 和 **FailoverPlatform** 如何在生产环境中降低成本与提升可用性。

## 业务场景

你在做一个 SaaS 产品。每天用户通过工单、评论、NPS 问卷、App Store 评价等渠道提交大量反馈。产品团队需要：自动分析每条反馈的情感倾向，提取关键意见和功能需求，将相似反馈聚类，发现趋势性问题。传统做法是人工读每条反馈打标签，效率极低且主观偏差大。

**典型应用场景：**

- 用户反馈情感分析与自动分类
- NPS/CSAT 调研数据结构化提取
- App Store / Google Play 评价监控
- 客诉热点发现与预警
- 产品需求优先级排序
- 竞品评价对比分析
- 周度/月度反馈趋势报告自动生成

---

## 涉及模块

| 模块 | Composer 包 | 用途 |
|------|------------|------|
| **Platform** | `symfony/ai-platform` | AI 平台统一接口、StructuredOutput、消息构建 |
| **Anthropic Bridge** | `symfony/ai-anthropic-platform` | 连接 Anthropic Claude（主分析引擎） |
| **Mistral Bridge** | `symfony/ai-mistral-platform` | 生成 Embedding 向量（用于反馈相似度检索） |
| **DeepSeek Bridge** | `symfony/ai-deep-seek-platform` | 替代分析引擎（高性价比方案） |
| **Cache Bridge** | `symfony/ai-cache-platform` | 缓存分析结果，避免重复 API 调用 |
| **Failover Bridge** | `symfony/ai-failover-platform` | 多平台容灾切换，确保分析服务高可用 |
| **Store** | `symfony/ai-store` | 向量存储核心（文档、向量化、检索） |
| **PostgreSQL Store** | `symfony/ai-postgres-store` | PostgreSQL pgvector 持久化存储 |
| **Agent** | `symfony/ai-agent` | 智能分析助手编排（InputProcessor / OutputProcessor） |

---

## 核心概念

在开始编码前，简要了解几个关键概念：

- **StructuredOutput**：让大模型返回与 PHP 类结构匹配的 JSON，自动反序列化为强类型对象。反馈分析结果天然适合结构化输出——情感、分类、关键词等字段都能精确映射到 DTO。
- **TextDocument + Metadata**：`TextDocument` 是向量存储中的基本文档单元，携带文本内容和 `Metadata` 元数据。将反馈原文和分析标签一起存储，方便后续过滤检索。
- **Vectorizer + DocumentIndexer**：`Vectorizer` 封装了向量化平台调用，`DocumentIndexer` 编排文档处理（分块 → 向量化 → 存储）的完整流水线。
- **Retriever + Reranker**：`Retriever` 封装查询向量化 + 相似度搜索，`Reranker` 对初步检索结果进行语义重排序，显著提升检索精度。
- **CachePlatform**：装饰器模式——包装任意平台实例，对相同输入返回缓存结果。批量反馈分析中大量相似表述可显著降低 API 成本。
- **FailoverPlatform**：接收多个平台实例，按顺序尝试调用。某个平台故障时自动切换，保障分析服务不中断。

---

## 项目实现思路

整个项目分为三个层次：

1. **基础分析层**：DTO 定义 → 单条反馈结构化分析 → 批量处理（Step 1-4）
2. **存储检索层**：向量化存储 → 相似反馈检索 → Reranker 精排（Step 5-7）
3. **智能报告层**：趋势分析 → Agent 编排 → 生产级集成（Step 8-10）

每一层都在前一层的基础上递进，从"分析反馈"到"理解反馈"再到"洞察反馈"。

---

## 项目流程图

```
+--------------------+
|  多渠道用户反馈    |
|  (工单/评论/NPS/   |
|   App Store 评价)  |
+--------+-----------+
         |
         v
+--------+-----------+     +---------------------+
| CachePlatform      |     | FailoverPlatform    |
| (Redis 缓存层)     |     | (Anthropic → DeepSeek)|
+--------+-----------+     +----------+----------+
         |                            |
         v                            v
+--------+----------------------------+----------+
|          Claude 结构化分析                      |
|          (StructuredOutput → DTO)               |
+--------+---------------------------------------+
         |
    +----+----+
    |         |
    v         v
+---+------+  +--------+-----------+
| 分析结果 |  | Vectorizer         |
| DTO 对象 |  | (Mistral Embed)    |
+---+------+  +--------+-----------+
    |                  |
    v                  v
+---+------+  +--------+-----------+
| 控制台   |  | PostgreSQL pgvector|
| 格式化   |  | 持久化向量存储     |
| 输出     |  +--------+-----------+
+-----------+          |
                  +----+----+
                  |         |
                  v         v
         +-------+---+  +--+-------------+
         | Retriever  |  | Reranker       |
         | 相似反馈   |  | 精确重排序     |
         | 检索       |  | (Mistral)      |
         +-------+----+  +--+-------------+
                  |          |
                  v          v
         +-------+----------+-------------+
         |     Agent 智能趋势分析          |
         |     + SystemPromptInputProcessor|
         |     + MemoryInputProcessor      |
         +--------+------------------------+
                  |
                  v
         +--------+-----------+
         | WeeklyFeedbackReport|
         | 结构化趋势报告      |
         +--------------------+
```

---

## 前置准备

### 环境要求

- PHP 8.2+
- Composer
- PostgreSQL 15+（安装 pgvector 扩展）
- Redis（用于缓存分析结果，可选）

### 安装依赖

```bash
# 核心平台 + Anthropic Claude（本教程主力 LLM）
composer require symfony/ai-platform symfony/ai-anthropic-platform

# Mistral 嵌入向量 + Reranker
composer require symfony/ai-mistral-platform

# DeepSeek 替代方案（高性价比）
composer require symfony/ai-deep-seek-platform

# 缓存与容灾
composer require symfony/ai-cache-platform symfony/ai-failover-platform

# 向量存储 + PostgreSQL pgvector
composer require symfony/ai-store symfony/ai-postgres-store

# Agent 智能助手
composer require symfony/ai-agent

# Symfony 基础组件
composer require symfony/event-dispatcher symfony/http-client
```

### 启动基础设施

```bash
# 启动 PostgreSQL + pgvector（向量存储）
docker run -d --name pgvector -p 5432:5432 \
    -e POSTGRES_USER=feedback \
    -e POSTGRES_PASSWORD=secret \
    -e POSTGRES_DB=feedback_analysis \
    pgvector/pgvector:pg16

# 启用 pgvector 扩展
docker exec -i pgvector psql -U feedback -d feedback_analysis \
    -c "CREATE EXTENSION IF NOT EXISTS vector;"

# 启动 Redis（可选，缓存分析结果）
docker run -d --name redis -p 6379:6379 redis:7-alpine
```

### 设置 API 密钥

```bash
export ANTHROPIC_API_KEY="sk-ant-your-key-here"
export MISTRAL_API_KEY="your-mistral-key-here"
# 可选：DeepSeek 替代方案
export DEEPSEEK_API_KEY="your-deepseek-key-here"
```

> [!WARNING]
> 🔒 **安全提醒：** 永远不要将 API 密钥硬编码到源代码中或提交到版本控制系统。在生产环境中，请使用 Symfony Secrets（`bin/console secrets:set`）、环境变量注入或 Vault 等密钥管理服务。

> [!NOTE]
> 本教程使用 Anthropic Claude 作为主分析引擎、Mistral 生成 Embedding 向量和 Reranking。你只需要这两个 API 密钥即可完成核心功能。DeepSeek 仅用于替代方案演示。

---

## Step 1：定义反馈分析 DTO 结构

StructuredOutput 让大模型返回与 PHP 类结构匹配的 JSON，自动反序列化为强类型对象。首先定义反馈分析的 DTO。

```php
<?php

namespace App\Dto;

/**
 * 单条用户反馈的 AI 分析结果
 */
final class FeedbackAnalysis
{
    /**
     * @param string   $sentiment       情感倾向（positive/neutral/negative）
     * @param float    $sentimentScore  情感评分（-1.0 到 1.0）
     * @param string   $category        反馈类别（bug/feature_request/complaint/praise/question）
     * @param string   $summary         反馈摘要（一句话概括核心诉求）
     * @param string[] $keywords        关键词列表（3-5 个）
     * @param string   $productArea     涉及产品模块（ui/performance/billing/api/onboarding/other）
     * @param string   $urgency         紧急程度（critical/high/medium/low）
     * @param string   $actionItem      建议的行动项（具体可执行的改进措施）
     * @param float    $confidence      分析置信度（0.0 ~ 1.0）
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
        public readonly float $confidence,
    ) {
    }
}
```

> [!TIP]
> 📝 **DTO 设计建议：** `confidence` 字段非常重要——它让你在后续可以根据置信度动态调整处理策略（如低置信度的分析结果送人工复核），而不是完全依赖 AI 的判断。StructuredOutput 的底层原理是：`PlatformSubscriber` 监听平台事件，将 PHP 类转换为 JSON Schema 传给大模型，大模型返回的 JSON 再自动反序列化为对象。

> [!IMPORTANT]
> 使用 StructuredOutput 时，你**必须**将 `PlatformSubscriber` 注册到 `EventDispatcher`。它负责在请求发送前将 `response_format` 选项中的 PHP 类转换为大模型能理解的 JSON Schema，并在响应返回后将 JSON 反序列化为 PHP 对象。

---

## Step 2：定义趋势报告嵌套 DTO

除了单条反馈分析，还需要定义趋势报告的嵌套结构，用于后续的周度汇总分析。

```php
<?php

namespace App\Dto;

/**
 * 产品模块维度的反馈趋势
 */
final class FeedbackTrend
{
    /**
     * @param string   $area           产品模块名称
     * @param int      $feedbackCount  该模块收到的反馈数量
     * @param string   $mainSentiment  主要情感倾向
     * @param string   $topIssue       最突出的问题描述
     * @param string[] $relatedKeywords 相关高频关键词
     */
    public function __construct(
        public readonly string $area,
        public readonly int $feedbackCount,
        public readonly string $mainSentiment,
        public readonly string $topIssue,
        public readonly array $relatedKeywords,
    ) {
    }
}

/**
 * 周度反馈分析报告
 */
final class WeeklyFeedbackReport
{
    /**
     * @param int             $totalCount       总反馈数
     * @param float           $avgSentiment     平均情感评分（-1.0 到 1.0）
     * @param float           $positiveRate     正面反馈占比（0.0 到 1.0）
     * @param FeedbackTrend[] $trends           各模块趋势
     * @param string[]        $topRequests      热门功能请求（按提及次数排序）
     * @param string[]        $criticalIssues   紧急待处理问题
     * @param string[]        $improvements     本周明显改善的方面
     * @param string          $executiveSummary 给管理层的摘要（100 字以内）
     */
    public function __construct(
        public readonly int $totalCount,
        public readonly float $avgSentiment,
        public readonly float $positiveRate,
        public readonly array $trends,
        public readonly array $topRequests,
        public readonly array $criticalIssues,
        public readonly array $improvements,
        public readonly string $executiveSummary,
    ) {
    }
}
```

> [!TIP]
> 💡 **嵌套 DTO 技巧：** `WeeklyFeedbackReport` 中的 `$trends` 字段类型为 `FeedbackTrend[]`。StructuredOutput 会递归解析嵌套的 DTO 类，自动生成对应的 JSON Schema。PHPDoc 中的 `@param` 注解用于标注数组元素类型，这是 StructuredOutput 识别嵌套结构的关键。

---

## Step 3：使用 Anthropic Claude 分析单条反馈

Anthropic Claude 在语义理解和结构化分析方面表现优秀，非常适合作为反馈分析的主引擎。

```php
<?php

require 'vendor/autoload.php';

use App\Dto\FeedbackAnalysis;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

// 1. 初始化 EventDispatcher 并注册 PlatformSubscriber（StructuredOutput 必需）
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

// 2. 创建 Anthropic 平台实例
$platform = PlatformFactory::create(
    $_ENV['ANTHROPIC_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);

// 3. 定义分析系统提示词
$analysisPrompt = <<<'PROMPT'
你是专业的产品反馈分析专家。对用户提交的反馈进行全面分析。

分析维度：
1. **情感倾向** — 判断正面/中立/负面，并给出 -1.0 到 1.0 的精确评分
2. **反馈类别** — bug（缺陷报告）/ feature_request（功能需求）/ complaint（投诉抱怨）/ praise（表扬肯定）/ question（咨询提问）
3. **产品模块** — ui / performance / billing / api / onboarding / other
4. **紧急程度** — critical（影响核心功能）/ high（影响用户体验）/ medium（一般问题）/ low（轻微建议）
5. **关键词** — 提取 3-5 个反映核心问题的关键词
6. **行动建议** — 给出具体、可执行的改进措施
7. **置信度** — 对分析结果的确信程度

注意：
- 情感评分要精确到小数点后两位
- 行动建议要具体，不要泛泛而谈
- 如果反馈内容模糊或信息不足，降低置信度
PROMPT;

// 4. 模拟用户反馈
$feedback = '订阅了高级版，但导出PDF功能经常超时，等了3分钟都没反应。'
    . '花了这么多钱结果核心功能都不能用，太让人失望了。希望尽快修复，不然只能退款了。';

// 5. 构建消息并调用分析
$messages = new MessageBag(
    Message::forSystem($analysisPrompt),
    Message::ofUser("请分析以下用户反馈：\n\n{$feedback}"),
);

/** @var FeedbackAnalysis $analysis */
$analysis = $platform->invoke('claude-sonnet-4-20250514', $messages, [
    'response_format' => FeedbackAnalysis::class,
])->asObject();

// 6. 输出分析结果
echo "=== 反馈分析结果 ===\n";
echo "情感倾向：{$analysis->sentiment}（评分：{$analysis->sentimentScore}）\n";
echo "反馈类别：{$analysis->category}\n";
echo "产品模块：{$analysis->productArea}\n";
echo "紧急程度：{$analysis->urgency}\n";
echo "分析摘要：{$analysis->summary}\n";
echo "关键词：" . implode('、', $analysis->keywords) . "\n";
echo "行动建议：{$analysis->actionItem}\n";
echo "置信度：" . number_format($analysis->confidence * 100, 1) . "%\n";
```

> [!TIP]
> 💡 **Prompt 设计要点：** 分析提示词中列出了所有合法的枚举值（如 `positive/neutral/negative`），这不仅引导大模型输出规范值，也方便后续代码做条件判断。在生产环境中，建议对返回值进行校验，确认枚举值在允许范围内。

---

## Step 4：批量反馈分析与 Token 用量监控

处理大量反馈时，需要关注 API 调用效率和成本。以下示例演示批量分析并监控 Token 用量。

```php
<?php

require 'vendor/autoload.php';

use App\Dto\FeedbackAnalysis;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$platform = PlatformFactory::create(
    $_ENV['ANTHROPIC_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);

// 模拟多渠道反馈数据
$feedbacks = [
    ['channel' => 'app_store', 'text' => '新版本的搜索功能太棒了！终于可以按日期筛选了，效率提升很多。'],
    ['channel' => 'support_ticket', 'text' => '登录页面加载很慢，有时候要等10秒以上，严重影响工作效率。'],
    ['channel' => 'nps_survey', 'text' => '能不能增加暗黑模式？现在晚上用太刺眼了，眼睛受不了。'],
    ['channel' => 'support_ticket', 'text' => '付款后没收到确认邮件，很担心钱是不是白花了。客服也联系不上。'],
    ['channel' => 'app_store', 'text' => '你们的API文档写得非常清晰，集成只花了半天，点赞！'],
    ['channel' => 'support_ticket', 'text' => '导出功能又崩了！上次反馈了还没修吗？已经影响我给客户交付了。'],
    ['channel' => 'nps_survey', 'text' => '希望支持企业SSO登录，我们公司有安全合规要求，目前无法采购。'],
    ['channel' => 'app_store', 'text' => '移动端体验很好，响应速度快，界面简洁大方，赞一个。'],
    ['channel' => 'support_ticket', 'text' => '仪表盘的图表数据和实际数据对不上，差了大概15%，不知道哪个准。'],
    ['channel' => 'nps_survey', 'text' => '整体还行，但定价太贵了，中小企业用不起。建议出个精简版。'],
];

$analysisPrompt = <<<'PROMPT'
你是产品反馈分析专家。快速分析用户反馈，提取情感、分类、关键词和行动建议。
情感评分精确到小数点后两位。行动建议要具体可执行。
PROMPT;

// 批量分析并追踪结果
$analyses = [];
$totalInputTokens = 0;
$totalOutputTokens = 0;

foreach ($feedbacks as $i => $feedback) {
    $messages = new MessageBag(
        Message::forSystem($analysisPrompt),
        Message::ofUser("来源渠道：{$feedback['channel']}\n反馈内容：{$feedback['text']}"),
    );

    $result = $platform->invoke('claude-sonnet-4-20250514', $messages, [
        'response_format' => FeedbackAnalysis::class,
    ]);

    /** @var FeedbackAnalysis $analysis */
    $analysis = $result->asObject();
    $analyses[] = ['feedback' => $feedback, 'analysis' => $analysis];

    // 追踪 Token 用量
    $metadata = $result->getMetadata();
    $totalInputTokens += $metadata['input_tokens'] ?? 0;
    $totalOutputTokens += $metadata['output_tokens'] ?? 0;

    // 高紧急度反馈立即标记
    $urgencyIcon = match ($analysis->urgency) {
        'critical' => '🚨',
        'high' => '⚠️',
        'medium' => '📋',
        default => '💡',
    };

    echo "{$urgencyIcon} #{$i} [{$analysis->sentiment}] {$analysis->summary}\n";
}

echo "\n=== 批量分析统计 ===\n";
echo "总反馈数：" . count($feedbacks) . " 条\n";
echo "输入 Token：{$totalInputTokens}\n";
echo "输出 Token：{$totalOutputTokens}\n";

// 按紧急度统计
$urgencyCounts = array_count_values(array_column(
    array_column($analyses, 'analysis'),
    'urgency',
));
echo "紧急度分布：\n";
foreach ($urgencyCounts as $level => $count) {
    echo "  {$level}: {$count} 条\n";
}
```

> [!TIP]
> 🏭 **生产建议：** 批量分析时，可以使用 Symfony Messenger 将每条反馈作为消息发送到队列，由多个 Worker 并行消费。这样单条反馈的处理失败不会影响其他反馈，且可以通过增加 Worker 数量线性扩展吞吐量。

> [!WARNING]
> ⚠️ **速率限制注意：** Anthropic API 有速率限制（RPM/TPM）。批量处理大量反馈时，建议在每次调用之间加入适当延迟（如 `usleep(200000)` = 200ms），或使用 `FailoverPlatform` 分散请求到多个平台。

---

## Step 5：向量化存储反馈数据

将分析过的反馈存入 PostgreSQL pgvector 向量数据库，为后续的相似反馈检索和聚类分析做准备。

```php
<?php

require 'vendor/autoload.php';

use App\Dto\FeedbackAnalysis;
use Symfony\AI\Platform\Bridge\Mistral\PlatformFactory as MistralFactory;
use Symfony\AI\Store\Bridge\Postgres\Store;
use Symfony\AI\Store\Document\DocumentProcessor;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentIndexer;
use Symfony\Component\HttpClient\HttpClient;

// 1. 创建 Mistral 平台实例（用于 Embedding 向量化）
$mistralPlatform = MistralFactory::create(
    $_ENV['MISTRAL_API_KEY'],
    HttpClient::create(),
);

// 2. 创建向量化器
$vectorizer = new Vectorizer($mistralPlatform, 'mistral-embed');

// 3. 创建 PostgreSQL 向量存储
$pdo = new \PDO(
    'pgsql:host=localhost;port=5432;dbname=feedback_analysis',
    'feedback',
    'secret',
);
$store = new Store($pdo, 'feedback_vectors', 'feedback_vector_idx');

// 4. 创建文档处理器和索引器
$processor = new DocumentProcessor($vectorizer);
$indexer = new DocumentIndexer($processor);

// 5. 将分析结果转换为文档并索引
// 假设 $analyses 来自 Step 4 的批量分析结果
$documents = [];
foreach ($analyses as $i => $item) {
    /** @var FeedbackAnalysis $analysis */
    $analysis = $item['analysis'];
    $feedback = $item['feedback'];

    $documents[] = new TextDocument(
        id: 'feedback-' . ($i + 1),
        content: $feedback['text'],
        metadata: new Metadata([
            'channel' => $feedback['channel'],
            'sentiment' => $analysis->sentiment,
            'sentiment_score' => $analysis->sentimentScore,
            'category' => $analysis->category,
            'product_area' => $analysis->productArea,
            'urgency' => $analysis->urgency,
            'summary' => $analysis->summary,
            'keywords' => implode(',', $analysis->keywords),
            'action_item' => $analysis->actionItem,
            'confidence' => $analysis->confidence,
            'analyzed_at' => date('Y-m-d H:i:s'),
        ]),
    );
}

// 批量索引
$indexer->index($store, $documents);

echo "✅ 已索引 " . count($documents) . " 条反馈到 PostgreSQL pgvector\n";
```

> [!TIP]
> 📝 **Metadata 设计：** 将分析标签（情感、分类、紧急度等）存入 `Metadata` 是关键设计——这让你在检索时不仅能做语义相似度搜索，还能在应用层按 metadata 字段过滤结果（如只看 `urgency=critical` 的反馈）。

> [!IMPORTANT]
> PostgreSQL 需要安装 pgvector 扩展才能存储和检索向量数据。使用 `CREATE EXTENSION IF NOT EXISTS vector;` 启用。Docker 用户可直接使用 `pgvector/pgvector:pg16` 镜像，已预装该扩展。

---

## Step 6：相似反馈检索

当产品经理想了解某个问题的所有相关反馈时，用向量检索快速定位语义相似的反馈。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\Mistral\PlatformFactory as MistralFactory;
use Symfony\AI\Store\Bridge\Postgres\Store;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Retriever;
use Symfony\Component\HttpClient\HttpClient;

// 复用 Mistral 平台和向量化器
$mistralPlatform = MistralFactory::create(
    $_ENV['MISTRAL_API_KEY'],
    HttpClient::create(),
);
$vectorizer = new Vectorizer($mistralPlatform, 'mistral-embed');

// 连接 PostgreSQL 向量存储
$pdo = new \PDO(
    'pgsql:host=localhost;port=5432;dbname=feedback_analysis',
    'feedback',
    'secret',
);
$store = new Store($pdo, 'feedback_vectors', 'feedback_vector_idx');

// 创建检索器
$retriever = new Retriever($store, $vectorizer);

// 场景 1：产品经理想查性能相关反馈
echo "=== 性能相关反馈 ===\n";
$results = $retriever->retrieve('页面加载慢，性能问题，响应超时');

foreach ($results as $result) {
    $meta = $result->metadata;
    $urgencyIcon = match ($meta['urgency'] ?? 'low') {
        'critical' => '🚨',
        'high' => '⚠️',
        'medium' => '📋',
        default => '💡',
    };
    echo "{$urgencyIcon} [{$meta['sentiment']}] {$meta['summary']}\n";
    echo "   原文：{$result->content}\n";
    echo "   模块：{$meta['product_area']} | 来源：{$meta['channel']}\n\n";
}

// 场景 2：检索定价相关反馈
echo "=== 定价相关反馈 ===\n";
$results = $retriever->retrieve('价格太贵，定价策略，付费方案');

foreach ($results as $result) {
    $meta = $result->metadata;
    echo "[{$meta['category']}] {$meta['summary']}\n";
    echo "   建议：{$meta['action_item']}\n\n";
}
```

> [!TIP]
> 💡 **检索查询技巧：** 向量检索基于语义相似度，而非关键词匹配。查询文本可以用自然语言描述你想找的内容（如"用户对价格的不满"），而不需要精确匹配反馈中的词汇。多个相关词组合（如"页面加载慢，性能问题，响应超时"）可以扩大语义覆盖范围。

---

## Step 7：Reranker 精确排序

初步向量检索的结果可能包含一些相关度不高的条目。使用 Reranker 对检索结果进行二次精排，显著提升结果质量。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\Mistral\PlatformFactory as MistralFactory;
use Symfony\AI\Store\Bridge\Postgres\Store;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Reranker\Reranker;
use Symfony\AI\Store\Retriever;
use Symfony\Component\HttpClient\HttpClient;

$mistralPlatform = MistralFactory::create(
    $_ENV['MISTRAL_API_KEY'],
    HttpClient::create(),
);
$vectorizer = new Vectorizer($mistralPlatform, 'mistral-embed');

$pdo = new \PDO(
    'pgsql:host=localhost;port=5432;dbname=feedback_analysis',
    'feedback',
    'secret',
);
$store = new Store($pdo, 'feedback_vectors', 'feedback_vector_idx');

// 1. 先做向量检索（召回候选集）
$retriever = new Retriever($store, $vectorizer);
$query = '导出功能有问题，导出失败，PDF 超时';
$candidates = $retriever->retrieve($query);

echo "=== 向量检索结果（重排序前） ===\n";
foreach ($candidates as $i => $doc) {
    echo ($i + 1) . ". [{$doc->metadata['urgency']}] {$doc->content}\n";
}

// 2. 使用 Reranker 精排
$reranker = new Reranker($mistralPlatform, 'mistral-rerank');
$reranked = $reranker->rerank($query, $candidates);

echo "\n=== Reranker 精排结果 ===\n";
foreach ($reranked as $i => $doc) {
    echo ($i + 1) . ". [{$doc->metadata['urgency']}] {$doc->content}\n";
}
```

> [!TIP]
> 📝 **Reranker 原理：** 向量检索是"双塔"模型——查询和文档分别编码后计算余弦相似度，速度快但精度有限。Reranker 是"交叉编码器"——将查询和每个候选文档拼接后联合编码，能捕捉更细粒度的语义关系。典型工作流是：向量检索召回 Top-20 → Reranker 精排取 Top-5。

> [!WARNING]
> ⚠️ **Reranker 成本：** Reranker 需要对每个候选文档独立打分，因此候选集越大、API 调用成本越高。建议将向量检索的召回数量控制在 10-20 条，而不是直接对整个数据库做 Reranking。

---

## Step 8：Agent 编排趋势分析报告

使用 Agent 自动汇总一段时间内的反馈数据，生成结构化的趋势报告。Agent 可以通过 `InputProcessor` 自动注入上下文信息。

```php
<?php

require 'vendor/autoload.php';

use App\Dto\WeeklyFeedbackReport;
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;
use Symfony\AI\Agent\Memory\StaticMemoryProvider;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

// 初始化平台
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$platform = PlatformFactory::create(
    $_ENV['ANTHROPIC_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);

// 定义报告生成的系统提示词
$reportPrompt = <<<'PROMPT'
你是资深产品运营分析专家。根据用户反馈数据生成周度分析报告。

分析维度：
1. **整体情感趋势** — 正面/负面/中立的占比变化
2. **各模块趋势** — 哪些产品模块收到最多反馈，主要问题是什么
3. **热门功能需求** — 被提及最多的功能请求，按优先级排序
4. **紧急问题** — 需要立即处理的 critical/high 级别问题
5. **积极改善** — 本周有哪些方面收到了正面反馈
6. **管理层摘要** — 100 字以内的核心发现总结

要求：
- 用数据说话，引用具体反馈条数和占比
- 管理层摘要要突出最需要关注的 1-2 个问题
- 热门需求要给出优先级建议
PROMPT;

// 构建 Agent（含记忆注入）
$agent = new Agent(
    platform: $platform,
    model: 'claude-sonnet-4-20250514',
    inputProcessors: [
        new SystemPromptInputProcessor($reportPrompt),
        new MemoryInputProcessor([
            new StaticMemoryProvider(
                '本产品是 B2B SaaS 项目管理工具',
                '当前版本 v3.2，上周发布了搜索功能升级',
                '已知问题：导出 PDF 超时（已排期修复，预计下周上线）',
                '竞品最近推出了暗黑模式和 SSO 集成',
            ),
        ]),
    ],
    outputProcessors: [],
);

// 准备本周反馈汇总数据
// 假设 $analyses 来自 Step 4 的批量分析结果
$weeklyData = '';
foreach ($analyses as $i => $item) {
    $a = $item['analysis'];
    $f = $item['feedback'];
    $weeklyData .= "- [{$a->sentiment}][{$a->productArea}][{$a->urgency}][{$f['channel']}] {$f['text']}\n";
}

$messages = new MessageBag(
    Message::ofUser(
        "以下是本周 " . count($analyses) . " 条用户反馈的结构化数据：\n\n"
        . $weeklyData
        . "\n请生成周度反馈分析报告。"
    ),
);

/** @var WeeklyFeedbackReport $report */
$report = $agent->call($messages, [
    'response_format' => WeeklyFeedbackReport::class,
])->asObject();

// 格式化输出报告
echo "╔══════════════════════════════════════╗\n";
echo "║       周度用户反馈分析报告           ║\n";
echo "╚══════════════════════════════════════╝\n\n";

echo "📊 总反馈：{$report->totalCount} 条\n";
echo "📈 平均情感分：" . number_format($report->avgSentiment, 2) . "\n";
echo "😊 正面反馈率：" . number_format($report->positiveRate * 100, 1) . "%\n\n";

echo "📋 各模块趋势：\n";
echo str_repeat('-', 60) . "\n";
foreach ($report->trends as $trend) {
    echo "  [{$trend->area}] {$trend->feedbackCount} 条 | {$trend->mainSentiment}\n";
    echo "    主要问题：{$trend->topIssue}\n";
    echo "    关键词：" . implode('、', $trend->relatedKeywords) . "\n\n";
}

echo "🔥 热门功能需求：\n";
foreach ($report->topRequests as $j => $request) {
    echo "  " . ($j + 1) . ". {$request}\n";
}

echo "\n🚨 紧急问题：\n";
foreach ($report->criticalIssues as $issue) {
    echo "  ❌ {$issue}\n";
}

echo "\n✅ 本周改善：\n";
foreach ($report->improvements as $improvement) {
    echo "  ✓ {$improvement}\n";
}

echo "\n📝 管理层摘要：\n{$report->executiveSummary}\n";
```

> [!TIP]
> 📝 **StaticMemoryProvider 妙用：** 通过 `StaticMemoryProvider` 注入产品背景信息（版本号、已知问题、竞品动态），让 AI 在生成报告时能结合上下文给出更有针对性的分析。例如当反馈提到"导出超时"时，AI 知道这是已排期修复的已知问题，会在报告中标注进展状态。

---

## Step 9：CachePlatform 与 FailoverPlatform 生产配置

在生产环境中，使用 CachePlatform 降低 API 成本，使用 FailoverPlatform 保障服务高可用。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory as AnthropicFactory;
use Symfony\AI\Platform\Bridge\Cache\CachePlatform;
use Symfony\AI\Platform\Bridge\DeepSeek\PlatformFactory as DeepSeekFactory;
use Symfony\AI\Platform\Bridge\Failover\FailoverPlatform;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\Cache\Adapter\RedisAdapter;
use Symfony\Component\Cache\Adapter\TagAwareAdapter;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create();
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

// 1. 创建主平台（Anthropic Claude）
$anthropicPlatform = AnthropicFactory::create(
    $_ENV['ANTHROPIC_API_KEY'],
    $httpClient,
    eventDispatcher: $dispatcher,
);

// 2. 创建备用平台（DeepSeek —— 高性价比替代方案）
$deepSeekPlatform = DeepSeekFactory::create(
    $_ENV['DEEPSEEK_API_KEY'],
    $httpClient,
    eventDispatcher: $dispatcher,
);

// 3. 配置 FailoverPlatform（Anthropic 故障时自动切换到 DeepSeek）
$failoverPlatform = new FailoverPlatform(
    $anthropicPlatform,
    $deepSeekPlatform,
);

// 4. 配置 CachePlatform（Redis 缓存分析结果）
$redisConnection = RedisAdapter::createConnection('redis://localhost:6379');
$platform = new CachePlatform(
    $failoverPlatform,
    cache: new TagAwareAdapter(new RedisAdapter($redisConnection)),
);

// 现在 $platform 具备：
// - 缓存层：相同反馈内容直接返回缓存结果（降低 API 成本）
// - 容灾层：Anthropic 不可用时自动切换到 DeepSeek（保障可用性）
// - StructuredOutput：分析结果自动映射为 PHP 对象

echo "✅ 生产级平台已就绪\n";
echo "   主平台：Anthropic Claude\n";
echo "   备用平台：DeepSeek\n";
echo "   缓存：Redis\n";
```

> [!TIP]
> 🏭 **缓存策略建议：** 反馈分析的缓存非常有效——用户经常用相似或相同的话反馈同一个问题。设置合理的 TTL（如 24 小时）即可。注意：趋势报告因为包含聚合数据，不建议长时间缓存。

> [!TIP]
> 💡 **FailoverPlatform 使用场景：** 除了平台故障切换，FailoverPlatform 还可以用于成本优化——将便宜的模型（如 DeepSeek）作为主平台处理简单反馈，只在分析失败时切换到更强的 Claude。

---

## Step 10：DeepSeek 替代方案

如果你更关注成本控制，DeepSeek 提供了极高性价比的分析能力。以下是使用 DeepSeek 作为独立分析引擎的示例。

```php
<?php

require 'vendor/autoload.php';

use App\Dto\FeedbackAnalysis;
use Symfony\AI\Platform\Bridge\DeepSeek\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

// 使用 DeepSeek 平台
$platform = PlatformFactory::create(
    $_ENV['DEEPSEEK_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);

$feedback = '你们的移动端 App 在 iOS 17 上经常闪退，特别是打开项目列表的时候。'
    . '已经反馈了两次了，能不能重视一下？';

$messages = new MessageBag(
    Message::forSystem('你是产品反馈分析专家。快速分析用户反馈。'),
    Message::ofUser("分析以下反馈：\n\n{$feedback}"),
);

/** @var FeedbackAnalysis $analysis */
$analysis = $platform->invoke('deepseek-chat', $messages, [
    'response_format' => FeedbackAnalysis::class,
])->asObject();

echo "=== DeepSeek 分析结果 ===\n";
echo "情感：{$analysis->sentiment}（{$analysis->sentimentScore}）\n";
echo "类别：{$analysis->category}\n";
echo "紧急：{$analysis->urgency}\n";
echo "摘要：{$analysis->summary}\n";
echo "建议：{$analysis->actionItem}\n";
```

> [!TIP]
> 💡 **多平台对比建议：** 在正式上线前，建议用同一批反馈样本分别测试 Claude、DeepSeek 等模型的分析质量，对比情感判断准确率、关键词提取精度和行动建议的实用性。通常 Claude 在细粒度情感理解上更优，DeepSeek 在中文反馈分析上有不错的性价比。

---

## 完整示例：生产级反馈分析 CLI 工具

将前面所有步骤整合为一个完整的命令行工具，支持批量分析、存储、检索和报告生成。

```php
<?php

require 'vendor/autoload.php';

use App\Dto\FeedbackAnalysis;
use App\Dto\WeeklyFeedbackReport;
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;
use Symfony\AI\Agent\Memory\StaticMemoryProvider;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory as AnthropicFactory;
use Symfony\AI\Platform\Bridge\Cache\CachePlatform;
use Symfony\AI\Platform\Bridge\DeepSeek\PlatformFactory as DeepSeekFactory;
use Symfony\AI\Platform\Bridge\Failover\FailoverPlatform;
use Symfony\AI\Platform\Bridge\Mistral\PlatformFactory as MistralFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\AI\Store\Bridge\Postgres\Store;
use Symfony\AI\Store\Document\DocumentProcessor;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentIndexer;
use Symfony\AI\Store\Reranker\Reranker;
use Symfony\AI\Store\Retriever;
use Symfony\Component\Cache\Adapter\RedisAdapter;
use Symfony\Component\Cache\Adapter\TagAwareAdapter;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

// =============================================
// 1. 基础设施初始化
// =============================================

$httpClient = HttpClient::create();
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

// Anthropic Claude — 主分析引擎
$anthropicPlatform = AnthropicFactory::create(
    $_ENV['ANTHROPIC_API_KEY'],
    $httpClient,
    eventDispatcher: $dispatcher,
);

// DeepSeek — 备用平台
$deepSeekPlatform = DeepSeekFactory::create(
    $_ENV['DEEPSEEK_API_KEY'],
    $httpClient,
    eventDispatcher: $dispatcher,
);

// FailoverPlatform + CachePlatform
$failoverPlatform = new FailoverPlatform($anthropicPlatform, $deepSeekPlatform);
$redisConnection = RedisAdapter::createConnection('redis://localhost:6379');
$platform = new CachePlatform(
    $failoverPlatform,
    cache: new TagAwareAdapter(new RedisAdapter($redisConnection)),
);

// Mistral — Embedding 向量化 + Reranker
$mistralPlatform = MistralFactory::create(
    $_ENV['MISTRAL_API_KEY'],
    $httpClient,
);
$vectorizer = new Vectorizer($mistralPlatform, 'mistral-embed');
$reranker = new Reranker($mistralPlatform, 'mistral-rerank');

// PostgreSQL pgvector — 向量存储
$pdo = new \PDO(
    'pgsql:host=localhost;port=5432;dbname=feedback_analysis',
    'feedback',
    'secret',
);
$store = new Store($pdo, 'feedback_vectors', 'feedback_vector_idx');

// 文档索引器
$processor = new DocumentProcessor($vectorizer);
$indexer = new DocumentIndexer($processor);

// 检索器
$retriever = new Retriever($store, $vectorizer);

// =============================================
// 2. 分析系统提示词
// =============================================

$analysisPrompt = <<<'PROMPT'
你是专业的产品反馈分析专家。对用户反馈进行全面结构化分析。

分析维度：情感倾向、类别、产品模块、紧急程度、关键词、行动建议。
情感评分精确到小数点后两位。行动建议必须具体可执行。
如果反馈信息不足以判断某个维度，降低置信度。
PROMPT;

// =============================================
// 3. 批量分析 + 向量化存储
// =============================================

$feedbacks = [
    ['channel' => 'app_store', 'text' => '新版本的搜索功能太棒了！终于可以按日期筛选了。'],
    ['channel' => 'support_ticket', 'text' => '登录页面加载很慢，有时候要等10秒以上。'],
    ['channel' => 'nps_survey', 'text' => '能不能增加暗黑模式？现在晚上用太刺眼了。'],
    ['channel' => 'support_ticket', 'text' => '付款后没收到确认邮件，很担心钱是不是白花了。'],
    ['channel' => 'app_store', 'text' => '你们的API文档写得非常清晰，集成只花了半天。'],
    ['channel' => 'support_ticket', 'text' => '导出功能又崩了！上次反馈还没修吗？'],
    ['channel' => 'nps_survey', 'text' => '希望支持企业SSO登录，我们公司有安全合规要求。'],
    ['channel' => 'app_store', 'text' => '移动端体验很好，响应速度快，赞一个。'],
];

echo "📥 开始批量分析 " . count($feedbacks) . " 条反馈...\n\n";

$analyses = [];
$documents = [];

foreach ($feedbacks as $i => $feedback) {
    $messages = new MessageBag(
        Message::forSystem($analysisPrompt),
        Message::ofUser("来源：{$feedback['channel']}\n反馈：{$feedback['text']}"),
    );

    /** @var FeedbackAnalysis $analysis */
    $analysis = $platform->invoke('claude-sonnet-4-20250514', $messages, [
        'response_format' => FeedbackAnalysis::class,
    ])->asObject();

    $analyses[] = ['feedback' => $feedback, 'analysis' => $analysis];

    // 构建向量文档
    $documents[] = new TextDocument(
        id: 'feedback-' . ($i + 1),
        content: $feedback['text'],
        metadata: new Metadata([
            'channel' => $feedback['channel'],
            'sentiment' => $analysis->sentiment,
            'sentiment_score' => $analysis->sentimentScore,
            'category' => $analysis->category,
            'product_area' => $analysis->productArea,
            'urgency' => $analysis->urgency,
            'summary' => $analysis->summary,
            'keywords' => implode(',', $analysis->keywords),
        ]),
    );

    $urgencyIcon = match ($analysis->urgency) {
        'critical' => '🚨',
        'high' => '⚠️',
        'medium' => '📋',
        default => '💡',
    };
    echo "{$urgencyIcon} #{$i} [{$analysis->sentiment}] {$analysis->summary}\n";
}

// 批量索引到向量数据库
$indexer->index($store, $documents);
echo "\n✅ 已索引 " . count($documents) . " 条反馈到 PostgreSQL pgvector\n";

// =============================================
// 4. 相似反馈检索 + Reranking
// =============================================

echo "\n🔍 检索性能相关反馈...\n";
$query = '页面加载慢，性能问题，超时';
$candidates = $retriever->retrieve($query);
$reranked = $reranker->rerank($query, $candidates);

foreach ($reranked as $j => $doc) {
    echo "  " . ($j + 1) . ". [{$doc->metadata['urgency']}] {$doc->content}\n";
}

// =============================================
// 5. Agent 生成趋势报告
// =============================================

$reportPrompt = <<<'PROMPT'
你是资深产品运营分析专家。根据用户反馈数据生成结构化周度报告。
用数据说话，管理层摘要控制在 100 字以内。
PROMPT;

$agent = new Agent(
    platform: $platform,
    model: 'claude-sonnet-4-20250514',
    inputProcessors: [
        new SystemPromptInputProcessor($reportPrompt),
        new MemoryInputProcessor([
            new StaticMemoryProvider(
                '本产品是 B2B SaaS 项目管理工具，当前版本 v3.2',
                '已知问题：导出 PDF 超时（已排期修复）',
            ),
        ]),
    ],
    outputProcessors: [],
);

$weeklyData = '';
foreach ($analyses as $item) {
    $a = $item['analysis'];
    $f = $item['feedback'];
    $weeklyData .= "- [{$a->sentiment}][{$a->productArea}][{$a->urgency}] {$f['text']}\n";
}

$reportMessages = new MessageBag(
    Message::ofUser("本周 " . count($analyses) . " 条反馈：\n\n{$weeklyData}\n请生成分析报告。"),
);

/** @var WeeklyFeedbackReport $report */
$report = $agent->call($reportMessages, [
    'response_format' => WeeklyFeedbackReport::class,
])->asObject();

echo "\n╔══════════════════════════════════════╗\n";
echo "║       周度用户反馈分析报告           ║\n";
echo "╚══════════════════════════════════════╝\n\n";
echo "📊 总反馈：{$report->totalCount} 条 | ";
echo "情感均分：" . number_format($report->avgSentiment, 2) . " | ";
echo "正面率：" . number_format($report->positiveRate * 100, 1) . "%\n\n";

foreach ($report->trends as $trend) {
    echo "  📌 [{$trend->area}] {$trend->feedbackCount} 条 — {$trend->topIssue}\n";
}

echo "\n🔥 热门需求：\n";
foreach ($report->topRequests as $k => $request) {
    echo "  " . ($k + 1) . ". {$request}\n";
}

echo "\n🚨 紧急问题：\n";
foreach ($report->criticalIssues as $issue) {
    echo "  ❌ {$issue}\n";
}

echo "\n📝 管理层摘要：{$report->executiveSummary}\n";
```

---

## 生产环境建议

> [!TIP]
> 🏭 **异步处理：** 在高流量场景下，不要在请求中同步分析反馈。使用 Symfony Messenger 将反馈分析请求发送到消息队列，由后台 Worker 异步处理。这样前端页面立即返回"分析中"的状态，分析完成后通过 WebSocket 或轮询更新结果。

> [!WARNING]
> ⚠️ **数据隐私合规：** 用户反馈可能包含个人隐私信息（PII）——姓名、手机号、邮箱、合同细节等。建议：1）在发送给 AI 分析前做 PII 脱敏；2）向量存储中不保存原始 PII；3）遵守 GDPR 等数据保护法规；4）设置反馈数据的保留期限（如 12 个月自动清理）。

> [!TIP]
> 🏭 **监控与告警：** 生产分析系统必须配置监控：1）分析延迟 P99 超过阈值时告警；2）负面反馈突增时告警（可能是线上事故）；3）API 调用失败率监控（配合 FailoverPlatform 日志）；4）缓存命中率监控（过低说明缓存策略需要优化）；5）每日 Token 用量监控（控制成本预算）。

---

## 存储方案对比

本教程使用 PostgreSQL pgvector 作为向量存储，以下是各方案的适用场景对比：

| 存储方案 | 适用场景 | 优势 | 限制 |
|----------|----------|------|------|
| **PostgreSQL pgvector** | 生产环境（本教程） | 成熟稳定，SQL 生态兼容，支持混合查询 | 需安装 pgvector 扩展 |
| **MongoDB Store** | 文档型数据偏好 | 灵活 Schema，水平扩展 | 向量搜索性能不如专用引擎 |
| **InMemory Store** | 开发、测试、演示 | 零配置，即开即用 | 数据不持久，重启丢失 |
| **Qdrant / Milvus** | 大规模向量检索 | 专为向量设计，性能最优 | 额外运维成本 |

> [!TIP]
> 💡 **混合查询优势：** PostgreSQL pgvector 的一大优势是支持 SQL + 向量的混合查询——你可以在向量相似度搜索的同时，用 SQL WHERE 条件过滤 metadata（如 `WHERE urgency = 'critical' AND channel = 'support_ticket'`），这在纯向量数据库中较难实现。

---

## 关键知识点总结

| 概念 | 类 / 组件 | 本教程中的用途 |
|------|-----------|--------------|
| **StructuredOutput** | `PlatformSubscriber` + DTO 类 | 将反馈分析结果映射为强类型 `FeedbackAnalysis` |
| **Anthropic Bridge** | `Bridge\Anthropic\PlatformFactory` | 创建 Claude 平台实例作为主分析引擎 |
| **Mistral Bridge** | `Bridge\Mistral\PlatformFactory` | 生成 Embedding 向量 + Reranker 精排 |
| **DeepSeek Bridge** | `Bridge\DeepSeek\PlatformFactory` | 高性价比替代分析方案 |
| **CachePlatform** | `Bridge\Cache\CachePlatform` | Redis 缓存分析结果，降低 API 成本 |
| **FailoverPlatform** | `Bridge\Failover\FailoverPlatform` | 多平台容灾切换（Anthropic → DeepSeek） |
| **TextDocument** | `Store\Document\TextDocument` | 反馈文本 + Metadata 元数据封装 |
| **Vectorizer** | `Store\Document\Vectorizer` | 将文本转换为向量嵌入（使用 Mistral Embed） |
| **DocumentIndexer** | `Store\Indexer\DocumentIndexer` | 编排文档的向量化和存储流水线 |
| **PostgreSQL Store** | `Store\Bridge\Postgres\Store` | pgvector 持久化向量存储 |
| **Retriever** | `Store\Retriever` | 封装查询向量化 + 相似度搜索的检索器 |
| **Reranker** | `Store\Reranker\Reranker` | 对检索结果进行语义精排 |
| **Agent** | `Agent\Agent` | 智能分析助手，编排趋势报告生成 |
| **SystemPromptInputProcessor** | `Agent\InputProcessor\SystemPromptInputProcessor` | 注入分析系统提示词 |
| **MemoryInputProcessor** | `Agent\Memory\MemoryInputProcessor` | 注入产品背景知识到 Agent 上下文 |
| **StaticMemoryProvider** | `Agent\Memory\StaticMemoryProvider` | 固定的产品信息记忆 |
| **嵌套 DTO** | `WeeklyFeedbackReport` → `FeedbackTrend` | StructuredOutput 递归解析嵌套结构 |

---

## 下一步

如果你需要用 AI 自动筛选和匹配简历，请看 [25-ai-recruitment-screening.md](./25-ai-recruitment-screening.md)。
