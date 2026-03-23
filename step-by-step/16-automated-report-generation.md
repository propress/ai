# 自动化报告生成

## 业务场景

你是一个内容运营团队的负责人。每周需要产出一份"行业动态周报"，涵盖行业新闻、竞品动态、技术趋势。以前这些工作全靠人工搜索和整理。现在你用 AI 自动从网上收集信息，抓取关键页面，整理成结构化的报告。

**典型应用：** 行业周报自动生成、竞品监控报告、舆情分析报告、投资研究报告

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台 |
| **Agent** | 智能体编排搜索和抓取流程 |
| **Agent Bridge Brave** | 网页搜索 |
| **Agent Bridge Tavily** | 深度搜索 + 网页内容提取 |
| **Agent Bridge Clock** | 当前时间（确保搜索时效） |
| **StructuredOutput** | 生成结构化报告对象 |

## 前置准备

```bash
composer require symfony/ai-platform symfony/ai-platform-openai
composer require symfony/ai-agent
composer require symfony/ai-agent-tavily
composer require symfony/ai-agent-clock
```

需要 Tavily API 密钥（免费注册可得）：
```bash
export TAVILY_API_KEY="your-tavily-api-key"
```

---

## Step 1：定义报告结构

```php
<?php

namespace App\Dto;

final class NewsItem
{
    /**
     * @param string  $title       新闻标题
     * @param string  $summary     内容摘要（2-3 句话）
     * @param string  $source      来源网站
     * @param string  $url         原文链接
     * @param string  $relevance   与主题的相关度（high/medium/low）
     */
    public function __construct(
        public readonly string $title,
        public readonly string $summary,
        public readonly string $source,
        public readonly string $url,
        public readonly string $relevance,
    ) {
    }
}

final class ReportSection
{
    /**
     * @param string     $title    章节标题
     * @param string     $analysis 分析内容
     * @param NewsItem[] $sources  该章节的信息来源
     */
    public function __construct(
        public readonly string $title,
        public readonly string $analysis,
        public readonly array $sources,
    ) {
    }
}

final class WeeklyReport
{
    /**
     * @param string          $title         报告标题
     * @param string          $period        报告周期（如"2025年1月13日-1月19日"）
     * @param string          $executiveSummary 执行摘要
     * @param ReportSection[] $sections      报告章节
     * @param string[]        $keyTakeaways  关键要点列表
     * @param string          $outlook       前瞻展望
     */
    public function __construct(
        public readonly string $title,
        public readonly string $period,
        public readonly string $executiveSummary,
        public readonly array $sections,
        public readonly array $keyTakeaways,
        public readonly string $outlook,
    ) {
    }
}
```

---

## Step 2：创建信息收集 Agent

使用 Tavily 工具进行深度搜索和网页内容提取。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\Clock\Clock;
use Symfony\AI\Agent\Bridge\Tavily\Tavily;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\Clock\Clock as SymfonyClock;
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create();
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], $httpClient);

// Tavily：深度搜索 + 网页内容提取
$tavily = new Tavily($httpClient, $_ENV['TAVILY_API_KEY']);
$clock = new Clock(new SymfonyClock());

$toolbox = new Toolbox([$tavily, $clock]);
$processor = new AgentProcessor($toolbox, includeSources: true);

$systemPrompt = new SystemPromptInputProcessor(
    '你是一个专业的行业研究分析师。'
    . '你的任务是使用搜索工具收集信息，然后整理成专业的研究报告。'
    . "\n\n工作流程："
    . "\n1. 先用 clock 获取当前日期"
    . "\n2. 用 tavily_search 针对不同主题搜索最新信息"
    . "\n3. 如有需要，用 tavily_extract 提取特定网页的详细内容"
    . "\n4. 综合分析，形成有深度的报告"
    . "\n\n要求：信息要有来源，分析要有深度，结论要有依据。"
);

$agent = new Agent(
    $platform, 'gpt-4o-mini',
    [$systemPrompt, $processor],
    [$processor],
);
```

---

## Step 3：生成行业研究报告

```php
<?php

// 方式一：自由文本报告
$result = $agent->call(new MessageBag(
    Message::ofUser(
        '请为我生成一份 AI/LLM 行业本周动态报告，包含以下部分：'
        . "\n1. 重大产品发布和更新"
        . "\n2. 值得关注的开源项目"
        . "\n3. 行业趋势分析"
        . "\n4. 关键要点和展望"
    ),
));

echo "=== AI 行业周报 ===\n\n";
echo $result->getContent() . "\n\n";

// 打印信息来源
$sources = $result->getMetadata()->get('sources');
if (null !== $sources && [] !== $sources) {
    echo "=== 参考来源 ===\n";
    foreach ($sources as $source) {
        echo "- {$source->name}: {$source->link}\n";
    }
}
```

---

## Step 4：生成结构化报告

结合 StructuredOutput 生成可程序化处理的报告对象。

```php
<?php

use App\Dto\WeeklyReport;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

// 使用支持结构化输出的 Platform
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$structuredPlatform = PlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    $httpClient,
    eventDispatcher: $dispatcher,
);

// 先用 Agent 收集信息
$researchResult = $agent->call(new MessageBag(
    Message::ofUser(
        '搜索 PHP 生态系统最近一周的动态，包括：'
        . '框架更新、新工具发布、社区热点讨论。'
        . '尽量多搜索几个不同的话题。'
    ),
));

$collectedInfo = $researchResult->getContent();

// 再用结构化输出整理成报告
$messages = new MessageBag(
    Message::forSystem(
        '你是报告编辑。将研究员收集的信息整理成结构化的周报。'
        . '报告要有执行摘要、分章节分析、关键要点和前瞻展望。'
    ),
    Message::ofUser("请将以下研究信息整理成结构化周报：\n\n{$collectedInfo}"),
);

$result = $structuredPlatform->invoke('gpt-4o-mini', $messages, [
    'response_format' => WeeklyReport::class,
]);

$report = $result->asObject();

// 格式化输出报告
echo "╔════════════════════════════════════╗\n";
echo "║  {$report->title}\n";
echo "║  {$report->period}\n";
echo "╚════════════════════════════════════╝\n\n";

echo "【执行摘要】\n{$report->executiveSummary}\n\n";

foreach ($report->sections as $i => $section) {
    echo "━━━ " . ($i + 1) . ". {$section->title} ━━━\n";
    echo "{$section->analysis}\n\n";

    if ([] !== $section->sources) {
        echo "  参考来源：\n";
        foreach ($section->sources as $item) {
            echo "  📰 [{$item->relevance}] {$item->title}\n";
            echo "     {$item->summary}\n";
            echo "     来源：{$item->source} | {$item->url}\n\n";
        }
    }
}

echo "━━━ 关键要点 ━━━\n";
foreach ($report->keyTakeaways as $takeaway) {
    echo "  ✦ {$takeaway}\n";
}

echo "\n━━━ 前瞻展望 ━━━\n{$report->outlook}\n";
```

---

## Step 5：定制化报告主题

轻松切换不同的报告主题。

```php
<?php

// 竞品分析报告
$result = $agent->call(new MessageBag(
    Message::ofUser(
        '请搜索以下三个竞品的最新动态，生成竞品分析报告：'
        . "\n- Notion"
        . "\n- Monday.com"  
        . "\n- Asana"
        . "\n\n重点关注：新功能发布、定价变化、用户反馈。"
    ),
));
echo $result->getContent() . "\n";

// 技术趋势报告
$result = $agent->call(new MessageBag(
    Message::ofUser(
        '搜索 2025 年 Web 开发领域的技术趋势，生成趋势报告。'
        . '涵盖：前端框架、后端技术、DevOps 工具、AI 集成。'
    ),
));
echo $result->getContent() . "\n";
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Tavily` search | 深度网页搜索，返回结构化结果 |
| `Tavily` extract | 提取指定 URL 的页面内容 |
| `includeSources: true` | 追踪信息来源 |
| 两阶段生成 | Agent 搜集信息 → StructuredOutput 整理报告 |
| 结构化报告对象 | `WeeklyReport` 可程序化处理和格式化 |
| 嵌套数据结构 | 报告包含章节、新闻条目等嵌套对象 |

## 下一步

如果你需要 AI 帮助管理和整理文件系统中的文档，请看 [17-ai-file-management-assistant.md](./17-ai-file-management-assistant.md)。
