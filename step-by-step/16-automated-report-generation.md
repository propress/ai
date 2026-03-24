# 自动化报告生成

## 业务场景

你是一个内容运营团队的负责人。每周需要产出一份"行业动态周报"，涵盖行业新闻、竞品动态、技术趋势。以前这些工作全靠人工搜索和整理，效率低且容易遗漏。现在你用 AI 自动从网上收集信息，抓取关键页面，整理成结构化的报告——从信息采集到报告输出，全程自动化。

**典型应用：** 行业周报自动生成、竞品监控报告、舆情分析报告、投资研究报告

## 项目流程图

```
┌──────────────┐     ┌───────────────────┐     ┌──────────────────────┐
│   用户输入     │ ──▶ │  Agent + 系统提示   │ ──▶ │  DeepSeek Platform   │
│ (报告主题)     │     │ (研究分析师角色)    │     │  (deepseek-chat)     │
└──────────────┘     └───────────────────┘     └──────────┬───────────┘
                                                           │
                                              LLM 决定调用哪些工具
                                                           │
                         ┌─────────────────────────────────┼──────────────────┐
                         ▼                                 ▼                  ▼
                ┌──────────────────┐            ┌───────────────┐    ┌──────────────┐
                │  tavily_search   │            │ tavily_extract │    │    clock      │
                │  深度网页搜索     │            │ 提取页面详情    │    │  获取当前日期  │
                └────────┬─────────┘            └───────┬───────┘    └──────┬───────┘
                         │                              │                   │
                         ▼                              ▼                   ▼
                ┌──────────────────────────────────────────────────────────────────┐
                │              AgentProcessor（includeSources: true）               │
                │            汇总工具结果 + 收集信息来源 → 发回 LLM                   │
                └─────────────────────────────┬────────────────────────────────────┘
                                              │
                                              ▼
                                   ┌───────────────────┐
                                   │  LLM 综合分析      │
                                   │  生成自由文本报告   │
                                   └─────────┬─────────┘
                                              │
                                              ▼
                              ┌────────────────────────────┐
                              │    StructuredOutput 阶段    │
                              │ 将信息整理为 WeeklyReport   │
                              │ (嵌套 DTO: ReportSection,  │
                              │  NewsItem)                  │
                              └──────────────┬─────────────┘
                                              │
                                              ▼
                              ┌────────────────────────────┐
                              │  结构化报告输出              │
                              │  执行摘要 / 分章节分析 /     │
                              │  关键要点 / 前瞻展望         │
                              └────────────────────────────┘
```

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform（DeepSeek）** | 连接 DeepSeek AI 平台（主要） |
| **Agent** | 智能体编排搜索和抓取流程 |
| **Agent Bridge Tavily** | 深度搜索（`tavily_search`）+ 网页内容提取（`tavily_extract`） |
| **Agent Bridge Clock** | 获取当前时间，确保搜索时效性 |
| **StructuredOutput** | 将自由文本转为嵌套 DTO 结构化报告 |

## 前置准备

```bash
composer require symfony/ai-platform symfony/ai-deepseek-platform
composer require symfony/ai-agent
composer require symfony/ai-agent-tavily
composer require symfony/ai-agent-clock
```

> **💡 提示：** 如果你更熟悉 OpenAI，可以替换安装包：
> ```bash
> composer require symfony/ai-platform-openai
> ```
> 后续代码只需更换 `PlatformFactory` 和模型名称即可。

需要 DeepSeek 和 Tavily 的 API 密钥：

```bash
export DEEPSEEK_API_KEY="your-deepseek-api-key"
export TAVILY_API_KEY="your-tavily-api-key"
```

---

## Step 1：定义报告结构（嵌套 DTO）

报告采用三层嵌套结构：`WeeklyReport` → `ReportSection` → `NewsItem`。

```php
<?php

namespace App\Dto;

final class NewsItem
{
    /**
     * @param string $title     新闻标题
     * @param string $summary   内容摘要（2-3 句话）
     * @param string $source    来源网站
     * @param string $url       原文链接
     * @param string $relevance 与主题的相关度（high/medium/low）
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
     * @param string     $analysis 分析内容（含深度洞察）
     * @param NewsItem[] $sources  该章节引用的信息来源
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
     * @param string          $title            报告标题
     * @param string          $period           报告周期（如"2025年1月13日-1月19日"）
     * @param string          $executiveSummary  执行摘要
     * @param ReportSection[] $sections          报告章节（嵌套 ReportSection）
     * @param string[]        $keyTakeaways      关键要点列表
     * @param string          $outlook           前瞻展望
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

> **💡 提示：** 嵌套 DTO 的好处是每个层级都有类型约束。`ReportSection` 中的 `$sources` 字段明确指定为
> `NewsItem[]`，确保 AI 为每条引用信息都提供标题、来源和链接，不会遗漏关键元数据。

---

## Step 2：创建信息收集 Agent

使用 Tavily 的 `search`（深度搜索）和 `extract`（网页内容提取）两个工具，配合 Clock 确保时间上下文。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\Clock\Clock;
use Symfony\AI\Agent\Bridge\Tavily\Tavily;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\DeepSeek\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\Clock\Clock as SymfonyClock;
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create();
$platform = PlatformFactory::create($_ENV['DEEPSEEK_API_KEY'], $httpClient);

// Tavily：注册后即有免费额度，提供 search 和 extract 两个工具
$tavily = new Tavily($httpClient, $_ENV['TAVILY_API_KEY']);
$clock = new Clock(new SymfonyClock());

$toolbox = new Toolbox([$tavily, $clock]);

// includeSources: true 启用来源追踪，工具返回的每条信息都会记录出处
$processor = new AgentProcessor($toolbox, includeSources: true);

$systemPrompt = new SystemPromptInputProcessor(
    '你是一个专业的行业研究分析师。'
    . '你的任务是使用搜索工具收集信息，然后整理成专业的研究报告。'
    . "\n\n工作流程："
    . "\n1. 先用 clock 获取当前日期，确保搜索最近时效"
    . "\n2. 用 tavily_search 针对不同主题搜索最新信息"
    . "\n3. 用 tavily_extract 提取关键网页的详细内容"
    . "\n4. 综合分析，形成有深度的报告"
    . "\n\n要求：信息要有来源，分析要有深度，结论要有依据。"
    . "\n当不同来源信息有冲突时，优先采信权威来源并标注分歧。"
);

$agent = new Agent(
    $platform, 'deepseek-chat',
    [$systemPrompt, $processor],
    [$processor],
);
```

> **⚠️ 注意：** Tavily 免费版有请求频率限制。在循环中调用 `tavily_search` 时，建议在系统提示中
> 控制搜索次数（如"最多搜索 5 个主题"），避免短时间内触发 API 速率限制。

> **💡 替换为 OpenAI：** 只需更改两行代码：
> ```php
> use Symfony\AI\Platform\Bridge\OpenAI\PlatformFactory;
> $platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], $httpClient);
> // Agent 模型换为 'gpt-4o-mini'
> ```

---

## Step 3：生成自由文本报告

Agent 自动调用 `clock` → `tavily_search` → `tavily_extract` 完成信息收集，并追踪所有来源。

```php
<?php

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

// includeSources: true 使所有工具来源自动收集到元数据中
$sources = $result->getMetadata()->get('sources');
if (null !== $sources && [] !== $sources) {
    echo "=== 参考来源 ===\n";
    foreach ($sources as $source) {
        echo "- {$source->name}: {$source->link}\n";
    }
}
```

> **💡 提示：** `includeSources: true` 会自动收集 Tavily 和 Clock 等实现了 `HasSourcesInterface`
> 的工具返回的来源信息。你无需手动解析工具输出——来源追踪在框架层自动完成。

---

## Step 4：生成结构化报告

采用"两阶段"策略：先用 Agent 搜集信息，再用 StructuredOutput 将自由文本整理为嵌套 DTO。

```php
<?php

use App\Dto\WeeklyReport;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

// 创建支持结构化输出的 Platform
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$structuredPlatform = PlatformFactory::create(
    $_ENV['DEEPSEEK_API_KEY'],
    $httpClient,
    eventDispatcher: $dispatcher,
);

// 第一阶段：Agent 搜集原始信息
$researchResult = $agent->call(new MessageBag(
    Message::ofUser(
        '搜索 PHP 生态系统最近一周的动态，包括：'
        . '框架更新、新工具发布、社区热点讨论。'
        . '尽量多搜索几个不同的话题。'
    ),
));

$collectedInfo = $researchResult->getContent();

// 第二阶段：StructuredOutput 转为嵌套 DTO
$messages = new MessageBag(
    Message::forSystem(
        '你是报告编辑。将研究员收集的信息整理成结构化的周报。'
        . '报告要有执行摘要、分章节分析、关键要点和前瞻展望。'
        . '每个章节必须包含引用来源（NewsItem），标注来源网站和链接。'
    ),
    Message::ofUser("请将以下研究信息整理成结构化周报：\n\n{$collectedInfo}"),
);

$result = $structuredPlatform->invoke('deepseek-chat', $messages, [
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

> **💡 提示：** 为什么分两阶段？Agent 阶段负责信息搜集（需要工具调用能力），StructuredOutput 阶段
> 负责格式化整理（需要 JSON Schema 约束）。拆分后每个阶段的提示词更聚焦，输出质量更高。

> **⚠️ 注意：** 当多个来源对同一事件的描述存在矛盾时（例如发布日期不一致），建议在系统提示中明确要求
> AI "标注信息冲突并注明各来源说法"，而不是静默地只采用其中一个版本。这对投资研究类报告尤为重要。

---

## Step 5：定制化报告主题

同一套 Agent 架构可灵活切换不同报告主题——只需改变用户消息内容。

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

## 生产环境建议

> **🏭 报告调度：** 使用 Symfony Scheduler 或 cron 定时任务自动执行周报生成。例如每周一早上 8 点触发，
> 将报告通过邮件或 Slack Webhook 推送给团队。

> **🏭 结果缓存：** Tavily 搜索结果可以缓存，避免短时间内重复搜索相同主题浪费 API 配额。使用
> Symfony Cache 组件按主题 + 日期做键值缓存，设置合理的 TTL（如 4 小时）。

> **🏭 审计日志：** 对于投资研究、合规类报告，建议记录每次报告生成的完整上下文：使用的搜索关键词、
> 返回的原始来源列表、AI 模型版本、生成时间戳。这为后续审计和溯源提供依据。

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `tavily_search` | 深度网页搜索，返回结构化结果和来源链接 |
| `tavily_extract` | 提取指定 URL 的页面详细内容 |
| `includeSources: true` | 启用框架层来源自动追踪 |
| 两阶段生成 | Agent 搜集信息 → StructuredOutput 整理报告 |
| 嵌套 DTO 结构 | `WeeklyReport` → `ReportSection` → `NewsItem` 三层结构 |
| 多来源冲突处理 | 提示词中要求标注信息分歧，不静默忽略 |
| DeepSeek 平台 | 高性价比国产 AI 平台，API 兼容主流接口 |

## 下一步

如果你需要 AI 帮助管理和整理文件系统中的文档，请看 [17-ai-file-management-assistant.md](./17-ai-file-management-assistant.md)。
