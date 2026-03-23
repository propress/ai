# 竞品情报监控系统

## 业务场景

你是产品部门的竞争情报分析师。需要持续监控 5-10 个竞品的动态：官网更新、产品功能变化、定价调整、新闻报道。以前靠每天人工浏览竞品网站，费时费力还容易遗漏。现在用 AI 自动抓取竞品网页、提取关键信息、对比分析、生成竞品动态报告。

**典型应用：** 竞品功能追踪、定价监控、市场趋势发现、招标信息收集、SEO 竞品分析

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台 |
| **Agent Bridge Firecrawl** | 深度网页抓取（JS 渲染、整站爬取） |
| **Agent Bridge Tavily** | 搜索竞品最新新闻和动态 |
| **Agent** | 编排抓取和分析流程 |
| **StructuredOutput** | 结构化竞品信息 |
| **Store** | 存储历史数据，追踪变化 |

## 前置准备

```bash
composer require symfony/ai-platform symfony/ai-platform-openai
composer require symfony/ai-agent
composer require symfony/ai-agent-firecrawl  # 网页抓取
composer require symfony/ai-agent-tavily     # 新闻搜索
composer require symfony/ai-store
```

需要 Firecrawl API 密钥（有免费额度）：
```bash
export FIRECRAWL_API_KEY="your-firecrawl-key"
```

---

## Step 1：定义竞品信息结构

```php
<?php

namespace App\Dto;

final class PricingPlan
{
    /**
     * @param string $name     套餐名称
     * @param string $price    价格（如 "$29/月"）
     * @param string[] $features 功能列表
     */
    public function __construct(
        public readonly string $name,
        public readonly string $price,
        public readonly array $features,
    ) {
    }
}

final class CompetitorSnapshot
{
    /**
     * @param string        $name          竞品名称
     * @param string        $tagline       一句话定位
     * @param string[]      $keyFeatures   核心功能列表
     * @param PricingPlan[] $pricing       定价方案
     * @param string        $targetAudience 目标用户群
     * @param string        $latestUpdate  最新动态/更新
     * @param string        $techStack     技术栈（如能识别）
     * @param string        $strength      竞争优势
     * @param string        $weakness      薄弱环节
     */
    public function __construct(
        public readonly string $name,
        public readonly string $tagline,
        public readonly array $keyFeatures,
        public readonly array $pricing,
        public readonly string $targetAudience,
        public readonly string $latestUpdate,
        public readonly string $techStack,
        public readonly string $strength,
        public readonly string $weakness,
    ) {
    }
}
```

---

## Step 2：抓取竞品网站

使用 Firecrawl 抓取竞品页面内容（支持 JavaScript 渲染的现代网站）。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\Firecrawl\Firecrawl;
use Symfony\AI\Agent\Bridge\Tavily\Tavily;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create();
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], $httpClient);

// 网页抓取工具
$firecrawl = new Firecrawl($httpClient, $_ENV['FIRECRAWL_API_KEY']);
$tavily = new Tavily($httpClient, $_ENV['TAVILY_API_KEY']);

$toolbox = new Toolbox([$firecrawl, $tavily]);
$processor = new AgentProcessor($toolbox);

$agent = new Agent(
    $platform, 'gpt-4o',
    [
        new SystemPromptInputProcessor(
            '你是竞争情报分析师。使用工具收集竞品信息：'
            . "\n1. 用 firecrawl_scrape 抓取指定 URL 的页面内容"
            . "\n2. 用 tavily_search 搜索竞品最新新闻"
            . "\n分析时注意：功能对比、定价变化、市场定位。"
        ),
        $processor,
    ],
    [$processor],
);

// 抓取竞品官网定价页
$result = $agent->call(new MessageBag(
    Message::ofUser(
        '请分析竞品 Notion 的最新情况：'
        . "\n1. 抓取 https://www.notion.so/pricing 获取定价信息"
        . "\n2. 搜索 'Notion 2025 新功能 发布' 获取最新动态"
        . "\n3. 给出综合分析"
    ),
));

echo $result->getContent() . "\n";
```

---

## Step 3：结构化竞品分析

```php
<?php

use App\Dto\CompetitorSnapshot;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$structuredPlatform = PlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    $httpClient,
    eventDispatcher: $dispatcher,
);

// 将 Agent 收集的原始信息，转为结构化数据
$rawIntel = $result->getContent();

$messages = new MessageBag(
    Message::forSystem(
        '根据收集的竞品情报，提取结构化信息。'
        . '如果某些信息无法确认，标注为"未知"。'
    ),
    Message::ofUser("竞品情报原始数据：\n\n{$rawIntel}"),
);

$structuredResult = $structuredPlatform->invoke('gpt-4o-mini', $messages, [
    'response_format' => CompetitorSnapshot::class,
]);

$snapshot = $structuredResult->asObject();

echo "=== 竞品快照：{$snapshot->name} ===\n";
echo "定位：{$snapshot->tagline}\n";
echo "目标用户：{$snapshot->targetAudience}\n";
echo "技术栈：{$snapshot->techStack}\n\n";

echo "核心功能：\n";
foreach ($snapshot->keyFeatures as $feature) {
    echo "  ✦ {$feature}\n";
}

echo "\n定价方案：\n";
foreach ($snapshot->pricing as $plan) {
    echo "  💰 {$plan->name}：{$plan->price}\n";
    foreach ($plan->features as $f) {
        echo "     - {$f}\n";
    }
}

echo "\n优势：{$snapshot->strength}\n";
echo "弱点：{$snapshot->weakness}\n";
echo "最新动态：{$snapshot->latestUpdate}\n";
```

---

## Step 4：竞品对比报告

```php
<?php

namespace App\Dto;

final class FeatureComparison
{
    /**
     * @param string $feature    功能名称
     * @param string $ourStatus  我方状态（available/partial/missing/planned）
     * @param string $competitor1 竞品1状态
     * @param string $competitor2 竞品2状态
     * @param string $priority   优先级建议（high/medium/low）
     */
    public function __construct(
        public readonly string $feature,
        public readonly string $ourStatus,
        public readonly string $competitor1,
        public readonly string $competitor2,
        public readonly string $priority,
    ) {
    }
}

final class CompetitiveReport
{
    /**
     * @param FeatureComparison[] $featureMatrix   功能对比矩阵
     * @param string[]            $ourAdvantages   我方优势
     * @param string[]            $threatFeatures  威胁功能（竞品有我们没有）
     * @param string[]            $opportunities   市场机会
     * @param string              $strategySummary 战略建议摘要
     */
    public function __construct(
        public readonly array $featureMatrix,
        public readonly array $ourAdvantages,
        public readonly array $threatFeatures,
        public readonly array $opportunities,
        public readonly string $strategySummary,
    ) {
    }
}
```

```php
<?php

use App\Dto\CompetitiveReport;

$messages = new MessageBag(
    Message::forSystem(
        '你是产品战略分析师。根据我方产品和竞品的信息，生成功能对比矩阵和战略建议。'
    ),
    Message::ofUser(
        "我方产品：CloudNote（在线协作笔记）\n"
        . "核心功能：Markdown 编辑、团队协作、API 集成、模板库\n"
        . "定价：$9/月\n\n"
        . "竞品1（Notion）：{$snapshot->tagline}\n"
        . "功能：" . implode(', ', $snapshot->keyFeatures) . "\n\n"
        . "竞品2（Obsidian）：本地优先笔记工具，Markdown 原生，插件生态\n\n"
        . '请生成竞品分析报告。'
    ),
);

$result = $structuredPlatform->invoke('gpt-4o', $messages, [
    'response_format' => CompetitiveReport::class,
]);

$report = $result->asObject();

echo "=== 竞品对比报告 ===\n\n";

echo "📊 功能矩阵：\n";
echo str_pad('功能', 20) . str_pad('我方', 12) . str_pad('Notion', 12) . str_pad('Obsidian', 12) . "优先级\n";
echo str_repeat('-', 68) . "\n";
foreach ($report->featureMatrix as $row) {
    echo str_pad($row->feature, 20)
        . str_pad($row->ourStatus, 12)
        . str_pad($row->competitor1, 12)
        . str_pad($row->competitor2, 12)
        . $row->priority . "\n";
}

echo "\n💪 我方优势：\n";
foreach ($report->ourAdvantages as $adv) {
    echo "  ✅ {$adv}\n";
}

echo "\n⚠️ 威胁功能：\n";
foreach ($report->threatFeatures as $threat) {
    echo "  🔴 {$threat}\n";
}

echo "\n💡 市场机会：\n";
foreach ($report->opportunities as $opp) {
    echo "  🟢 {$opp}\n";
}

echo "\n📋 战略建议：\n{$report->strategySummary}\n";
```

---

## Step 5：变化追踪

将快照存入向量库，下次抓取时对比变化。

```php
<?php

use Symfony\AI\Store\Document\Document;
use Symfony\AI\Store\Document\Metadata;

// 保存当前快照
$document = new Document(
    id: 'competitor-notion-' . date('Y-m-d'),
    content: json_encode([
        'name' => $snapshot->name,
        'features' => $snapshot->keyFeatures,
        'pricing' => array_map(fn ($p) => "{$p->name}: {$p->price}", $snapshot->pricing),
        'latest' => $snapshot->latestUpdate,
    ], JSON_UNESCAPED_UNICODE),
    metadata: new Metadata([
        'competitor' => $snapshot->name,
        'date' => date('Y-m-d'),
        'pricing_hash' => md5(json_encode($snapshot->pricing)),
    ]),
);

$indexer->index([$document]);
echo "✅ 快照已保存，可与历史数据对比\n";
```

---

## 完整流程

```
竞品列表
    │
    ├──► [Firecrawl 抓取官网] → 功能/定价页面
    │
    ├──► [Tavily 搜索新闻] → 最新动态
    │
    ▼
[Agent 综合分析]
    │
    ▼
[结构化提取] → CompetitorSnapshot
    │
    ├──► [功能对比] → CompetitiveReport（矩阵 + 战略）
    │
    └──► [存储快照] → Store（追踪变化）
              │
              ▼
         下次对比 → 发现变化 → 告警
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Firecrawl` Bridge | 深度网页抓取，支持 JS 渲染 |
| `Tavily` Bridge | AI 搜索，获取最新新闻 |
| `CompetitorSnapshot` | 结构化竞品快照（功能/定价/定位） |
| `CompetitiveReport` | 功能对比矩阵 + 战略建议 |
| 变化追踪 | 定期快照存入 Store，对比历史数据 |
| Agent 编排 | 自动抓取 + 搜索 + 分析 |

## 下一步

如果你需要构建高可用的 AI 服务架构（多平台故障转移 + 缓存），请看 [28-high-availability-ai-architecture.md](./28-high-availability-ai-architecture.md)。
