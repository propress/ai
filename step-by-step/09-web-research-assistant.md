# 联网研究助手

## 业务场景

你在做一个市场调研工具。分析师输入一个研究主题，AI 自动上网搜索相关信息、抓取网页内容，然后整理成结构化的研究报告，并附上所有参考来源。

**典型应用：** 竞品分析、市场调研、舆情监控、新闻聚合、信息收集、学术调研

## 涉及模块

| 模块 | Composer 包 | 用途 |
|------|------------|------|
| **Platform** | `symfony/ai-platform` + `symfony/ai-gemini-platform` | 连接 AI 平台，发送消息与接收回复 |
| **Agent** | `symfony/ai-agent` | 智能体框架，自动编排工具调用 |
| **Agent Bridge Brave** | `symfony/ai-brave-tool` | Brave 搜索引擎工具 |
| **Agent Bridge Tavily** | `symfony/ai-tavily-tool` | Tavily AI 搜索工具 |
| **Agent Bridge SerpApi** | `symfony/ai-serp-api-tool` | Google 搜索 API 工具 |
| **Agent Bridge Scraper** | `symfony/ai-scraper-tool` | 基础网页内容抓取 |
| **Agent Bridge Firecrawl** | `symfony/ai-firecrawl-tool` | 高级网页抓取（支持 JS 渲染） |
| **Agent Bridge Clock** | `symfony/ai-clock-tool` | 当前时间工具（确保搜索时效性） |
| **Agent Bridge Wikipedia** | `symfony/ai-wikipedia-tool` | 维基百科搜索工具 |

> **💡 提示：** 本教程使用 **Google Gemini** 作为主要平台。前几篇分别使用了 OpenAI 和 Anthropic，本篇使用 Gemini 以展示平台多样性。所有工具与平台无关，换成其他 Bridge 只需改一行代码。

## 项目流程图

```
┌──────────────┐
│  研究主题输入  │
│ (用户提问)    │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    Agent + AgentProcessor                            │
│                (includeSources: true 开启来源追踪)                    │
└──────────────────────────────┬───────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        Gemini API (AI 推理)                          │
│              判断需要哪些工具，规划调用顺序                              │
└──────┬───────────┬──────────┬───────────┬──────────┬────────────────┘
       │           │          │           │          │
       ▼           ▼          ▼           ▼          ▼
┌──────────┐ ┌──────────┐ ┌────────┐ ┌────────┐ ┌────────────┐
│  Clock   │ │  Brave / │ │Scraper │ │Wikipedia│ │ Firecrawl  │
│ 获取时间  │ │ Tavily / │ │ 抓取   │ │ 百科    │ │ JS 渲染抓取 │
│          │ │ SerpApi  │ │ 网页   │ │ 知识    │ │            │
└──────┬───┘ └────┬─────┘ └───┬────┘ └───┬────┘ └─────┬──────┘
       │          │           │          │             │
       └──────────┴───────────┴──────────┴─────────────┘
                              │
                              ▼
                   ┌─────────────────────┐
                   │ SourceCollection    │
                   │ 汇总所有工具的来源    │
                   └──────────┬──────────┘
                              │
                              ▼
                   ┌─────────────────────┐
                   │  LLM 综合所有信息    │
                   │  生成结构化研究报告   │
                   └──────────┬──────────┘
                              │
                              ▼
                   ┌─────────────────────┐
                   │ 返回报告 + 参考来源   │
                   └─────────────────────┘
```

## 搜索工具对比

选择合适的搜索工具是构建研究助手的关键。以下是各工具的特点对比：

| 特性 | Brave | Tavily | SerpApi | Wikipedia |
|------|-------|--------|---------|-----------|
| **数据来源** | Brave 搜索引擎 | AI 优化搜索 | Google 搜索 | 维基百科 |
| **实时性** | ✅ 实时 | ✅ 实时 | ✅ 实时 | ⚠️ 延迟更新 |
| **结果质量** | 通用搜索 | AI 优化排序 | Google 排名 | 权威百科 |
| **注册工具名** | `brave_search` | `tavily_search` / `tavily_extract` | `serpapi` | `wikipedia_search` / `wikipedia_article` |
| **免费额度** | 有（每月 2000 次） | 有（每月 1000 次） | 有（每月 100 次） | 无限制 |
| **适用场景** | 通用网页搜索 | 需要 AI 整理的搜索 | 需要 Google 数据 | 百科知识查询 |
| **API 密钥** | 需要 | 需要 | 需要 | 不需要 |

> **💡 提示：** 可以在一个 Agent 中同时注册多个搜索工具。AI 会根据问题性质自动选择最合适的工具 —— 例如，知识问题用 Wikipedia，时事新闻用 Brave 或 Tavily。

---

## 前置准备

### 环境要求

- PHP 8.2+
- Composer
- Google Gemini API 密钥（[获取地址](https://aistudio.google.com/apikey)）
- 至少一个搜索服务的 API 密钥

### 安装依赖

```bash
# 核心依赖
composer require symfony/ai-platform symfony/ai-gemini-platform symfony/ai-agent

# 搜索工具（选择至少一个）
composer require symfony/ai-brave-tool       # Brave 搜索
composer require symfony/ai-tavily-tool      # Tavily AI 搜索
composer require symfony/ai-serp-api-tool    # SerpApi (Google)

# 网页抓取
composer require symfony/ai-scraper-tool     # 基础抓取
composer require symfony/ai-firecrawl-tool   # JS 渲染抓取

# 辅助工具
composer require symfony/ai-clock-tool       # 时间
composer require symfony/ai-wikipedia-tool   # 维基百科
```

### 设置 API 密钥

```bash
export GEMINI_API_KEY="your-gemini-api-key"
export BRAVE_API_KEY="your-brave-api-key"         # 可选
export TAVILY_API_KEY="your-tavily-api-key"       # 可选
export SERPAPI_API_KEY="your-serpapi-api-key"      # 可选
export FIRECRAWL_API_KEY="your-firecrawl-api-key" # 可选
```

> **🔒 安全建议：** 不要把 API 密钥硬编码在源码中。使用环境变量或 Symfony Secrets 管理敏感信息。

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
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create();
$platform = PlatformFactory::create($_ENV['GEMINI_API_KEY'], $httpClient);

// 创建 Brave 搜索工具
$brave = new Brave($httpClient, $_ENV['BRAVE_API_KEY']);

$toolbox = new Toolbox([$brave]);
$processor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'gemini-2.0-flash', [$processor], [$processor]);

// 搜索最新信息
$messages = new MessageBag(
    Message::forSystem('你是一个研究助手。使用搜索工具查找信息，基于搜索结果回答。不要编造信息。'),
    Message::ofUser('2025 年 PHP 最流行的框架是哪些？各有什么特点？'),
);

$result = $agent->call($messages);
echo $result->getContent() . "\n";
```

> **💡 提示：** 如果你更喜欢使用 OpenAI，只需替换平台创建部分：
> ```php
> use Symfony\AI\Platform\Bridge\OpenAI\PlatformFactory;
> $platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], $httpClient);
> $agent = new Agent($platform, 'gpt-4o-mini', [$processor], [$processor]);
> ```

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
$agent = new Agent($platform, 'gemini-2.0-flash', [$processor], [$processor]);

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

> **⚠️ 注意：** `Scraper` 是基础 HTML 抓取工具，无法处理需要 JavaScript 渲染的页面（如 SPA 应用）。如果目标页面依赖 JS 渲染内容，请使用 Firecrawl（见 Step 5）。

---

## Step 3：追踪信息来源

在研究场景中，知道信息来自哪里很重要。使用 `includeSources: true` 可以追踪每个工具返回的数据来源。所有搜索和抓取 Bridge 都实现了 `HasSourcesInterface`，会自动收集来源信息。

```php
<?php

use Symfony\AI\Agent\Toolbox\Source\Source;
use Symfony\AI\Agent\Toolbox\Source\SourceCollection;

// 开启来源追踪
$processor = new AgentProcessor($toolbox, includeSources: true);
$agent = new Agent($platform, 'gemini-2.0-flash', [$processor], [$processor]);

$messages = new MessageBag(
    Message::ofUser('搜索一下最近的 AI 行业大事件。'),
);

$result = $agent->call($messages);

// 获取回答
echo "=== 研究结果 ===\n";
echo $result->getContent() . "\n\n";

// 获取信息来源（SourceCollection 实现了 IteratorAggregate）
$sources = $result->getMetadata()->get('sources');
if ($sources instanceof SourceCollection && 0 < count($sources)) {
    echo "=== 参考来源 ===\n";
    foreach ($sources as $source) {
        /** @var Source $source */
        echo "- [{$source->getName()}] {$source->getReference()}\n";
        echo "  {$source->getContent()}\n\n";
    }
}
```

**Source 对象包含三个属性：**

| 属性 | 方法 | 说明 |
|------|------|------|
| `name` | `getName()` | 来源名称（如网站标题） |
| `reference` | `getReference()` | 来源引用（如 URL 链接） |
| `content` | `getContent()` | 来源内容摘要 |

> **💡 提示：** `SourceCollection` 同时实现了 `IteratorAggregate` 和 `Countable` 接口，你可以直接用 `foreach` 遍历，也可以调用 `count($sources)` 获取来源数量，或用 `$sources->all()` 获取完整数组。

---

## Step 4：使用 Wikipedia 做知识研究

对于百科知识类问题，Wikipedia 工具更高效 —— 免费、无需 API 密钥、内容权威。

```php
<?php

use Symfony\AI\Agent\Bridge\Wikipedia\Wikipedia;

$wikipedia = new Wikipedia($httpClient, locale: 'zh'); // 中文维基百科

$toolbox = new Toolbox([$wikipedia]);
$processor = new AgentProcessor($toolbox, includeSources: true);
$agent = new Agent($platform, 'gemini-2.0-flash', [$processor], [$processor]);

// Wikipedia 注册两个工具：wikipedia_search（搜索）和 wikipedia_article（获取文章）
$messages = new MessageBag(
    Message::forSystem('使用 Wikipedia 查找信息来回答用户问题。'),
    Message::ofUser('请介绍一下量子计算的基本原理和目前的发展状况。'),
);

$result = $agent->call($messages);
echo $result->getContent() . "\n";
```

---

## Step 5：Tavily AI 搜索 + Firecrawl 深度抓取

Tavily 提供 AI 优化的搜索结果；Firecrawl 能抓取 JavaScript 渲染的页面，适合现代 SPA 网站。

```php
<?php

use Symfony\AI\Agent\Bridge\Tavily\Tavily;
use Symfony\AI\Agent\Bridge\Firecrawl\Firecrawl;
use Symfony\AI\Agent\Bridge\Clock\Clock;
use Symfony\Component\Clock\Clock as SymfonyClock;

// Tavily 注册两个工具：tavily_search（搜索）和 tavily_extract（提取网页内容）
$tavily = new Tavily($httpClient, $_ENV['TAVILY_API_KEY']);

// Firecrawl 注册三个工具：firecrawl_scrape（抓取）、firecrawl_crawl（爬取）、firecrawl_map（站点地图）
$firecrawl = new Firecrawl($httpClient, $_ENV['FIRECRAWL_API_KEY'], 'https://api.firecrawl.dev');

$clock = new Clock(new SymfonyClock());

$toolbox = new Toolbox([$tavily, $firecrawl, $clock]);
$processor = new AgentProcessor($toolbox, includeSources: true);
$agent = new Agent($platform, 'gemini-2.0-flash', [$processor], [$processor]);

$messages = new MessageBag(
    Message::forSystem(
        '你是一个深度研究助手。使用 tavily_search 搜索信息，'
        . '对于需要 JavaScript 渲染的现代网站，使用 firecrawl_scrape 获取完整内容。'
        . '使用 firecrawl_map 可以发现一个网站下的所有页面链接。'
    ),
    Message::ofUser('调研 React Server Components 的最新进展和社区反馈。'),
);

$result = $agent->call($messages);
echo $result->getContent() . "\n";
```

> **💡 提示：** Tavily 的 `tavily_extract` 工具接受 URL 数组参数，可以一次性提取多个页面的内容，适合批量信息收集场景。

---

## Step 6：SerpApi —— Google 搜索数据

SerpApi 提供结构化的 Google 搜索结果，适合需要 Google 排名数据的场景。

```php
<?php

use Symfony\AI\Agent\Bridge\SerpApi\SerpApi;

$serpApi = new SerpApi($httpClient, $_ENV['SERPAPI_API_KEY']);

$toolbox = new Toolbox([$serpApi]);
$processor = new AgentProcessor($toolbox, includeSources: true);
$agent = new Agent($platform, 'gemini-2.0-flash', [$processor], [$processor]);

$messages = new MessageBag(
    Message::forSystem('使用搜索工具查找信息来回答问题。'),
    Message::ofUser('Symfony 框架在 GitHub 上的 star 数量和最新版本是什么？'),
);

$result = $agent->call($messages);
echo $result->getContent() . "\n";
```

> **💡 提示：** 在生产环境中，建议组合使用多个搜索工具以提高覆盖率。不同搜索引擎对相同关键词的排名和结果可能有显著差异。

---

## 完整示例：竞品分析助手

模拟市场分析师使用 AI 进行竞品调研的完整流程 —— 多工具协作 + 来源追踪。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\Brave\Brave;
use Symfony\AI\Agent\Bridge\Clock\Clock;
use Symfony\AI\Agent\Bridge\Firecrawl\Firecrawl;
use Symfony\AI\Agent\Bridge\Scraper\Scraper;
use Symfony\AI\Agent\Bridge\Wikipedia\Wikipedia;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Source\Source;
use Symfony\AI\Agent\Toolbox\Source\SourceCollection;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\Clock\Clock as SymfonyClock;
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create();
$platform = PlatformFactory::create($_ENV['GEMINI_API_KEY'], $httpClient);

// 配备完整的研究工具集
$toolbox = new Toolbox([
    new Brave($httpClient, $_ENV['BRAVE_API_KEY']),      // 通用网页搜索
    new Scraper($httpClient),                             // 基础网页抓取
    new Firecrawl(                                        // JS 渲染抓取
        $httpClient,
        $_ENV['FIRECRAWL_API_KEY'],
        'https://api.firecrawl.dev',
    ),
    new Clock(new SymfonyClock()),                        // 当前时间
    new Wikipedia($httpClient),                           // 百科知识
]);

// 开启来源追踪，并限制工具调用次数防止无限循环
$processor = new AgentProcessor($toolbox, includeSources: true, maxToolCalls: 20);
$agent = new Agent($platform, 'gemini-2.5-flash', [$processor], [$processor]);

$systemPrompt = '你是一个专业的市场研究分析师。'
    . '使用可用的工具搜索和收集信息，整理成专业的分析报告。'
    . '对于现代 SPA 网站，优先使用 firecrawl_scrape 而非 scraper 进行内容抓取。'
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
    if ($sources instanceof SourceCollection && 0 < count($sources)) {
        echo "=== 参考来源 ===\n";
        foreach ($sources as $source) {
            /** @var Source $source */
            echo "- {$source->getName()}: {$source->getReference()}\n";
        }
    }
    echo "\n";
}
```

---

## 生产环境建议

### 限速与重试

> **🏭 生产建议：** 搜索 API 通常有速率限制。使用 Symfony HttpClient 的 `retry_failed` 配置来自动重试失败的请求：
> ```php
> $httpClient = HttpClient::create([
>     'timeout' => 30,
> ]);
> ```
> 同时通过 `AgentProcessor` 的 `maxToolCalls` 参数限制单次对话的工具调用次数，避免 AI 进入无限调用循环。

### 内容验证

> **⚠️ 注意：** 网页抓取的内容可能包含不完整或不相关的数据。建议在 system prompt 中指导 AI：
> - 交叉验证多个来源的信息
> - 对于关键数据，标注信息的可信度
> - 忽略明显的广告或无关内容
> - 当信息矛盾时，优先信任权威来源

### 缓存搜索结果

> **🏭 生产建议：** 对于相同或相似的查询，搜索 API 会返回类似结果但消耗额度。使用 Symfony HttpClient 的缓存 HTTP 客户端或在应用层实现缓存策略：
> ```php
> use Symfony\Component\HttpClient\CachingHttpClient;
> use Symfony\Component\HttpKernel\HttpCache\Store;
>
> $store = new Store('/path/to/cache');
> $httpClient = new CachingHttpClient(HttpClient::create(), $store);
> ```

### 内容安全

> **🔒 安全建议：** 从网络抓取的内容可能包含恶意数据。在将抓取内容展示给用户前，务必进行适当的 HTML 清理。使用 `strip_tags()` 或 Symfony 的 HtmlSanitizer 组件对输出进行过滤。

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Brave` | Brave 搜索引擎工具（`brave_search`） |
| `Tavily` | AI 优化搜索（`tavily_search` / `tavily_extract`） |
| `SerpApi` | Google 搜索 API（`serpapi`） |
| `Wikipedia` | 维基百科工具（`wikipedia_search` / `wikipedia_article`） |
| `Scraper` | 基础 HTML 抓取（`scraper`） |
| `Firecrawl` | JS 渲染抓取（`firecrawl_scrape` / `firecrawl_crawl` / `firecrawl_map`） |
| `Clock` | 时间工具（`clock`），确保搜索时效性 |
| `HasSourcesInterface` | 工具实现此接口后，自动收集来源到 `SourceCollection` |
| `includeSources: true` | 启用 `AgentProcessor` 的来源追踪 |
| `$result->getMetadata()->get('sources')` | 获取 `SourceCollection`（含所有工具的来源） |
| `maxToolCalls` | 限制工具调用次数，防止无限循环 |

## 下一步

最后，我们把所有模块组合起来，构建一个完整的 RAG 聊天系统，它有知识库、持久化对话、工具调用和记忆能力。请看 [10-rag-chat-with-persistence.md](./10-rag-chat-with-persistence.md)。
