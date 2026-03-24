# 联网研究助手

## 业务场景

你在做一个市场调研工具。分析师输入一个研究主题，AI 自动上网搜索相关信息、抓取网页内容，然后整理成结构化的研究报告。

**典型应用：** 竞品分析、市场调研、舆情监控、新闻聚合、信息收集

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台 |
| **Agent** | 智能体框架，自动调用搜索工具 |
| **Agent Bridge Brave** | Brave 搜索引擎工具 |
| **Agent Bridge Scraper** | 网页内容抓取工具 |
| **Agent Bridge Clock** | 当前时间工具（确保搜索时效性） |
| **Agent Bridge Wikipedia** | 维基百科搜索工具 |

## 前置准备

```bash
composer require symfony/ai-platform symfony/ai-platform-openai
composer require symfony/ai-agent
composer require symfony/ai-agent-brave      # Brave 搜索
composer require symfony/ai-agent-scraper    # 网页抓取
composer require symfony/ai-agent-clock      # 时间
composer require symfony/ai-agent-wikipedia  # 维基百科
```

需要 Brave Search API 密钥（免费申请）：
```bash
export BRAVE_API_KEY="your-brave-api-key"
```

---

## Step 1：基础搜索 —— Brave Search

让 AI 能搜索网页，获取最新信息。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\Brave\Brave;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create();
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], $httpClient);

// 创建 Brave 搜索工具
$brave = new Brave($httpClient, $_ENV['BRAVE_API_KEY']);

$toolbox = new Toolbox([$brave]);
$processor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'gpt-4o-mini', [$processor], [$processor]);

// 搜索最新信息
$messages = new MessageBag(
    Message::forSystem('你是一个研究助手。使用搜索工具查找信息，基于搜索结果回答。不要编造信息。'),
    Message::ofUser('2025 年 PHP 最流行的框架是哪些？各有什么特点？'),
);

$result = $agent->call($messages);
echo $result->getContent() . "\n";
```

---

## Step 2：搜索 + 网页抓取

有时搜索结果的摘要不够详细，需要进一步抓取网页获取完整内容。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\Brave\Brave;
use Symfony\AI\Agent\Bridge\Clock\Clock;
use Symfony\AI\Agent\Bridge\Scraper\Scraper;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\Clock\Clock as SymfonyClock;

// 组合三个工具：搜索 + 抓取 + 时间
$brave = new Brave($httpClient, $_ENV['BRAVE_API_KEY']);
$scraper = new Scraper($httpClient);
$clock = new Clock(new SymfonyClock());

$toolbox = new Toolbox([$brave, $scraper, $clock]);
$processor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'gpt-4o-mini', [$processor], [$processor]);

// AI 会先搜索，找到相关页面后自动抓取内容
$messages = new MessageBag(
    Message::forSystem(
        '你是一个市场研究助手。先用搜索工具查找信息，如果需要更详细的内容，使用抓取工具获取网页全文。'
        . '基于收集到的信息整理成结构化的报告。'
    ),
    Message::ofUser('调研一下 Symfony AI 这个项目的最新动态和主要功能。'),
);

$result = $agent->call($messages);
echo $result->getContent() . "\n";
```

**AI 的工作流程：**
1. 先调用 `clock` 获取当前时间（确保搜索时效性）
2. 调用 `brave_search` 搜索相关网页
3. 从搜索结果中选择最相关的几个 URL
4. 调用 `scraper` 抓取这些网页的内容
5. 综合所有信息生成结构化报告

---

## Step 3：追踪信息来源

在研究场景中，知道信息来自哪里很重要。使用 `includeSources: true` 可以追踪每个工具返回的数据来源。

```php
<?php

use Symfony\AI\Agent\Toolbox\Source\Source;

// 开启来源追踪
$processor = new AgentProcessor($toolbox, includeSources: true);
$agent = new Agent($platform, 'gpt-4o-mini', [$processor], [$processor]);

$messages = new MessageBag(
    Message::ofUser('搜索一下最近的 AI 行业大事件。'),
);

$result = $agent->call($messages);

// 获取回答
echo "=== 研究结果 ===\n";
echo $result->getContent() . "\n\n";

// 获取信息来源
echo "=== 参考来源 ===\n";
$sources = $result->getMetadata()->get('sources');
if (null !== $sources) {
    foreach ($sources as $source) {
        /** @var Source $source */
        echo "- [{$source->name}] {$source->link}\n";
        echo "  {$source->description}\n\n";
    }
}
```

---

## Step 4：使用 Wikipedia 做知识研究

对于百科知识类问题，Wikipedia 工具更高效。

```php
<?php

use Symfony\AI\Agent\Bridge\Wikipedia\Wikipedia;

$wikipedia = new Wikipedia($httpClient, locale: 'zh'); // 中文维基百科

$toolbox = new Toolbox([$wikipedia]);
$processor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'gpt-4o-mini', [$processor], [$processor]);

// Wikipedia 提供两个工具：搜索和获取文章
$messages = new MessageBag(
    Message::forSystem('使用 Wikipedia 查找信息来回答用户问题。'),
    Message::ofUser('请介绍一下量子计算的基本原理和目前的发展状况。'),
);

$result = $agent->call($messages);
echo $result->getContent() . "\n";
```

---

## 完整示例：竞品分析助手

模拟市场分析师使用 AI 进行竞品调研的完整流程。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\Brave\Brave;
use Symfony\AI\Agent\Bridge\Clock\Clock;
use Symfony\AI\Agent\Bridge\Scraper\Scraper;
use Symfony\AI\Agent\Bridge\Wikipedia\Wikipedia;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\Clock\Clock as SymfonyClock;
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create();
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], $httpClient);

// 配备完整的研究工具集
$toolbox = new Toolbox([
    new Brave($httpClient, $_ENV['BRAVE_API_KEY']),
    new Scraper($httpClient),
    new Clock(new SymfonyClock()),
    new Wikipedia($httpClient),
]);

$processor = new AgentProcessor($toolbox, includeSources: true);
$agent = new Agent($platform, 'gpt-4o-mini', [$processor], [$processor]);

$systemPrompt = '你是一个专业的市场研究分析师。'
    . '使用可用的工具搜索和收集信息，整理成专业的分析报告。'
    . '报告要求：1) 有数据支撑 2) 标注信息来源 3) 给出分析和建议';

// 模拟分析师的研究任务
$researchTopics = [
    '帮我分析 PHP 在 2025 年的市场地位，与 Python 和 Node.js 对比，包括使用率、就业市场、主要应用领域。',
];

foreach ($researchTopics as $topic) {
    echo "=== 研究任务 ===\n";
    echo $topic . "\n\n";

    $result = $agent->call(new MessageBag(
        Message::forSystem($systemPrompt),
        Message::ofUser($topic),
    ));

    echo "=== 研究报告 ===\n";
    echo $result->getContent() . "\n\n";

    // 打印来源
    $sources = $result->getMetadata()->get('sources');
    if (null !== $sources && [] !== $sources) {
        echo "=== 参考来源 ===\n";
        foreach ($sources as $source) {
            echo "- {$source->name}: {$source->link}\n";
        }
    }
    echo "\n";
}
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Brave` | Brave 搜索引擎工具，返回网页搜索结果 |
| `Scraper` | 网页内容抓取工具，获取页面全文 |
| `Wikipedia` | 维基百科工具，搜索和获取文章 |
| `Clock` | 时间工具，让 AI 知道当前时间 |
| `includeSources: true` | 启用来源追踪 |
| `$result->getMetadata()->get('sources')` | 获取信息来源列表 |
| 工具组合 | 多工具协作，AI 自动编排调用顺序 |

## 下一步

最后，我们把所有模块组合起来，构建一个完整的 RAG 聊天系统，它有知识库、持久化对话、工具调用和记忆能力。请看 [10-rag-chat-with-persistence.md](./10-rag-chat-with-persistence.md)。
