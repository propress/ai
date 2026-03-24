# 智能天气出行助手

## 业务场景

你在做一个旅行服务 App。用户输入出发地和目的地，AI 自动查询目的地坐标、获取实时天气和未来几天的天气预报，综合给出穿衣建议、出行建议和行程规划。这是一个将多种工具（地理编码、天气、时钟）组合使用的典型场景，展示了 Agent 框架的**多工具自动编排**能力。

**典型应用：** 旅行助手、出行规划、户外活动推荐、物流天气预警

## 涉及模块

| 模块 | Composer 包 | 用途 |
|------|------------|------|
| **Platform** | `symfony/ai-platform` | AI 平台统一接口 |
| **Mistral Bridge** | `symfony/ai-mistral-platform` | 连接 Mistral AI 平台 |
| **Agent** | `symfony/ai-agent` | 智能体框架，自动编排多工具调用 |
| **Agent Bridge Mapbox** | `symfony/ai-agent-mapbox` | 地理编码：地址 ↔ 经纬度坐标 |
| **Agent Bridge OpenMeteo** | `symfony/ai-agent-open-meteo` | 天气查询：实时天气 + 未来预报 |
| **Agent Bridge Clock** | `symfony/ai-agent-clock` | 当前时间，确保建议时效性 |

> **💡 提示：** 本教程使用 **Mistral AI** 作为主要平台。Symfony AI 的工具系统与平台无关——切换平台只需更换 `PlatformFactory` 和模型名，所有工具（Mapbox、OpenMeteo、Clock）无需任何修改。文末提供 OpenAI 等其他平台的切换方式。

## 项目流程图

```
┌──────────────┐     ┌───────────────┐     ┌──────────────────┐     ┌─────────────┐
│   用户输入     │ ──▶ │ InputProcessor │ ──▶ │  Agent 调用 LLM   │ ──▶ │ Mistral API │
│ "去杭州旅游"   │     │ (系统提示+工具)  │     │ (携带工具定义)     │     │  (AI 推理)   │
└──────────────┘     └───────────────┘     └──────────────────┘     └──────┬──────┘
                                                                            │
                                                                            ▼
                                                                    ┌──────────────┐
                                                              ┌──── │  LLM 返回结果  │
                                                              │     └──────────────┘
                                                              │
                                  ┌───────────────────────────┼────────────────────────────┐
                                  │                           │                            │
                                  ▼                           ▼                            ▼
                         ┌──────────────┐          ┌────────────────┐          ┌──────────────────┐
                         │  直接文本回复  │          │  请求调用工具    │          │  请求调用多个工具  │
                         │  (无需工具)   │          │  (单个工具)     │          │  (自动编排)       │
                         └──────┬───────┘          └────────┬───────┘          └────────┬─────────┘
                                │                           │                            │
                                │                           ▼                            ▼
                                │                  ┌────────────────┐          ┌──────────────────┐
                                │                  │ AgentProcessor  │          │ AgentProcessor   │
                                │                  │ 执行单个工具     │          │ 依次执行多个工具   │
                                │                  └────────┬───────┘          └────────┬─────────┘
                                │                           │                            │
                                │                           ▼                            ▼
                                │                  ┌────────────────┐          ┌──────────────────┐
                                │                  │  clock()        │          │ ① clock()        │
                                │                  │  geocode()      │          │ ② geocode("杭州") │
                                │                  │  weather_*()    │          │ ③ weather_forecast │
                                │                  └────────┬───────┘          └────────┬─────────┘
                                │                           │                            │
                                │                           ▼                            ▼
                                │                  ┌────────────────┐          ┌──────────────────┐
                                │                  │  工具结果发回LLM │          │ 全部结果发回 LLM  │
                                │                  │  → 生成最终回复  │          │ → 综合生成出行建议 │
                                │                  └────────┬───────┘          └────────┬─────────┘
                                │                           │                            │
                                ▼                           ▼                            ▼
                            ┌────────────────────────────────────────────────────────────────┐
                            │                    返回给用户（出行建议）                         │
                            └────────────────────────────────────────────────────────────────┘
```

> **💡 提示：** Agent 的核心优势在于**自动编排**——你不需要手动指定工具调用顺序。LLM 会根据用户问题自动决定调用哪些工具、以什么顺序调用、传什么参数。一个"去杭州旅游"的问题可能触发 3-5 次工具调用，全部由 Agent 自动完成。

---

## 前置准备

### 环境要求

- PHP 8.2+
- Composer
- Mistral API Key（[console.mistral.ai](https://console.mistral.ai) 注册获取）
- Mapbox API Token（[mapbox.com](https://www.mapbox.com) 免费注册）

### 安装依赖

```bash
composer require symfony/ai-platform symfony/ai-mistral-platform
composer require symfony/ai-agent
composer require symfony/ai-agent-mapbox       # 地理编码
composer require symfony/ai-agent-open-meteo   # 天气（免费，无需 API Key）
composer require symfony/ai-agent-clock        # 时钟
```

### 设置 API 密钥

```bash
export MISTRAL_API_KEY="your-mistral-api-key"
export MAPBOX_TOKEN="your-mapbox-token"
```

> **🔒 安全建议：** 不要把 API 密钥硬编码在源码中。推荐使用 Symfony 的 `secrets:set` 命令或 `.env.local` 文件管理密钥。Mapbox Token 有免费额度（每月 10 万次请求），OpenMeteo 完全免费无需密钥。

---

## Step 1：理解多工具编排

Agent 收到用户消息后，LLM 会**自动分析需要哪些信息**，然后按需调用工具：

```
用户："我明天要去杭州旅游"
  → ① clock()              获取当前日期（确定"明天"是哪天）
  → ② geocode("杭州")       将地名转为坐标（30.29, 120.16）
  → ③ weather_forecast(…)   用坐标查询天气预报
  → ④ AI 综合所有数据        生成穿衣建议 + 出行建议 + 行程规划
```

这就是**多工具自动编排**的核心——开发者只需注册工具，Agent 自动决定调用顺序和参数。

> **⚠️ 注意：** 工具的 `description` 至关重要。LLM 根据描述来判断何时该调用哪个工具。Mapbox 工具描述了"将地址转为坐标"，所以当用户提到地名时，LLM 会自动选择先调用 `geocode`，再用得到的坐标调用天气工具。

---

## Step 2：创建多工具 Agent

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\Clock\Clock;
use Symfony\AI\Agent\Bridge\Mapbox\Mapbox;
use Symfony\AI\Agent\Bridge\OpenMeteo\OpenMeteo;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\Mistral\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\Clock\Clock as SymfonyClock;
use Symfony\Component\HttpClient\HttpClient;

// 1. 创建平台和 HTTP 客户端
$httpClient = HttpClient::create();
$platform = PlatformFactory::create($_ENV['MISTRAL_API_KEY'], $httpClient);

// 2. 组合三个工具桥接
$mapbox = new Mapbox($httpClient, $_ENV['MAPBOX_TOKEN']);
$weather = new OpenMeteo($httpClient);
$clock = new Clock(new SymfonyClock());

// 3. 创建工具箱和处理器
$toolbox = new Toolbox([$mapbox, $weather, $clock]);
$processor = new AgentProcessor($toolbox);

// 4. 定义系统提示
$systemPrompt = new SystemPromptInputProcessor(
    '你是一个智能出行助手。当用户提到地点时：'
    . "\n1. 先用 geocode 工具将地名转为坐标"
    . "\n2. 用 weather_current 或 weather_forecast 获取天气"
    . "\n3. 综合信息给出出行建议"
    . "\n\n建议包括：穿衣建议、是否需要带伞、适合的活动、出行注意事项。"
    . "\n用中文回答。"
);

// 5. 创建 Agent
$agent = new Agent(
    $platform, 'mistral-large-latest',
    [$systemPrompt, $processor],
    [$processor],
);
```

> **💡 提示：** `AgentProcessor` 同时作为 `InputProcessor` 和 `OutputProcessor` 注册。作为 InputProcessor，它将工具定义注入到 LLM 请求中；作为 OutputProcessor，它拦截工具调用请求并执行工具，然后将结果返回给 LLM。

---

## Step 3：简单查询 —— 单城市天气

```php
<?php

$result = $agent->call(new MessageBag(
    Message::ofUser('杭州现在天气怎么样？适合户外跑步吗？'),
));

echo "助手：" . $result->getContent() . "\n\n";
```

**Agent 自动编排的调用链：**
1. `geocode("杭州")` → 获得坐标 `(30.29, 120.16)`
2. `weather_current(30.29, 120.16)` → 获得当前天气（温度、风速、天气状况）
3. LLM 基于天气数据给出跑步建议

> **💡 提示：** Mapbox 的 `geocode` 工具返回的结果包含 `relevance` 评分（0-1），表示匹配的准确度。对于中文地名，建议输入尽量具体——"杭州市西湖区"比"西湖"能得到更准确的坐标。如果需要精确到街道，可以附带省市信息以提高匹配度。

---

## Step 4：复杂查询 —— 旅行规划

```php
<?php

$result = $agent->call(new MessageBag(
    Message::ofUser(
        '我计划下周从上海出发去杭州玩 3 天。请帮我：'
        . "\n1. 查一下杭州未来 5 天的天气"
        . "\n2. 根据天气推荐每天适合去的景点"
        . "\n3. 给出穿衣和装备建议"
    ),
));

echo "=== 旅行规划 ===\n";
echo $result->getContent() . "\n";
```

**Agent 自动编排的调用链：**
1. `clock()` → 确认当前日期（确定"下周"具体是哪几天）
2. `geocode("杭州")` → 获得坐标
3. `weather_forecast(30.29, 120.16, days: 5)` → 获取未来 5 天预报
4. LLM 综合天气数据生成每天的景点推荐和穿衣建议

> **⚠️ 注意：** OpenMeteo 的 `forecast` 工具支持 1-16 天预报（`$days` 参数，默认 7）。超过 7 天的预报准确性会下降，建议在系统提示中提醒用户：7 天以上的天气预报仅供参考。

---

## Step 5：双城市对比

```php
<?php

$result = $agent->call(new MessageBag(
    Message::ofUser(
        '这周末想出去玩一天，在苏州和南京之间犹豫。'
        . '帮我比较一下这两个城市周末的天气，推荐去哪个？'
    ),
));

echo $result->getContent() . "\n";
```

**Agent 自动编排的调用链：**
1. `clock()` → 确定周末日期
2. `geocode("苏州")` → 坐标 A
3. `geocode("南京")` → 坐标 B
4. `weather_forecast(A, days: 3)` → 苏州天气
5. `weather_forecast(B, days: 3)` → 南京天气
6. LLM 对比分析，推荐更适合出行的城市

这个场景体现了多工具编排的强大之处——Agent 自动为两个城市分别执行地理编码和天气查询，然后综合对比。开发者无需写任何编排逻辑。

> **💡 提示：** 对比查询会触发 5-6 次工具调用。在高并发场景中，注意 Mapbox API 的速率限制（免费账户 600 次/分钟）。如果对比的城市是热门城市，可以考虑缓存地理编码结果（坐标不会变化），详见 Step 7 的缓存方案。

---

## Step 6：反向地理编码 —— 根据坐标找地点

```php
<?php

$result = $agent->call(new MessageBag(
    Message::ofUser(
        '我目前在坐标 (39.9042, 116.4074) 的位置，'
        . '这是哪里？附近天气怎么样？有什么推荐的？'
    ),
));

echo $result->getContent() . "\n";
```

**Agent 自动编排的调用链：**
1. `reverse_geocode(116.4074, 39.9042)` → "北京市东城区..."
2. `weather_current(39.9042, 116.4074)` → 当前天气
3. LLM 结合地点和天气给出推荐

> **⚠️ 注意：** `reverse_geocode` 工具的参数顺序是 `(longitude, latitude)`，即**经度在前、纬度在后**。这是 Mapbox API 的约定。LLM 通常会正确处理这个顺序，但如果你在系统提示中引导用户输入坐标，建议注明格式为 `(纬度, 经度)` 以符合用户习惯，LLM 会自动转换。

---

## Step 7：生产环境优化 —— 响应缓存

在生产环境中，天气数据和地理编码结果可以缓存以减少 API 调用次数和降低延迟：

```bash
composer require symfony/ai-platform-cache symfony/cache
```

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\Clock\Clock;
use Symfony\AI\Agent\Bridge\Mapbox\Mapbox;
use Symfony\AI\Agent\Bridge\OpenMeteo\OpenMeteo;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\Cache\CachePlatform;
use Symfony\AI\Platform\Bridge\Mistral\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\Cache\Adapter\TagAwareAdapter;
use Symfony\Component\Cache\Adapter\FilesystemAdapter;
use Symfony\Component\Clock\Clock as SymfonyClock;
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create();

// 创建基础平台
$mistralPlatform = PlatformFactory::create($_ENV['MISTRAL_API_KEY'], $httpClient);

// 用 CachePlatform 包装，缓存 LLM 响应
$cache = new TagAwareAdapter(new FilesystemAdapter());
$platform = new CachePlatform(
    platform: $mistralPlatform,
    cache: $cache,
    cacheTtl: 1800,  // 缓存 30 分钟
);

// 工具和 Agent 配置不变
$toolbox = new Toolbox([
    new Mapbox($httpClient, $_ENV['MAPBOX_TOKEN']),
    new OpenMeteo($httpClient),
    new Clock(new SymfonyClock()),
]);

$processor = new AgentProcessor($toolbox);
$agent = new Agent(
    $platform, 'mistral-large-latest',
    [
        new SystemPromptInputProcessor(
            '你是智能出行助手。使用地理编码和天气工具为用户提供出行建议。'
            . '回答包含：天气概况、穿衣建议、活动推荐、注意事项。用中文回答。'
        ),
        $processor,
    ],
    [$processor],
);

// 相同的查询会命中缓存，大幅降低响应时间
$result = $agent->call(new MessageBag(
    Message::ofUser('杭州现在天气怎么样？'),
));

echo $result->getContent() . "\n";
```

> **🏭 生产建议：** `CachePlatform` 是一个装饰器，包装任何 `PlatformInterface` 实现来缓存 LLM 响应。对于天气出行助手，建议根据数据时效性设置不同的缓存策略：天气数据变化快，TTL 设为 15-30 分钟；地理编码结果基本不变，可以设置更长的 TTL。缓存使用 Symfony Cache 组件的 `TagAwareAdapter`，支持按模型名标签批量清除。

> **🏭 生产建议：** 在高流量场景中，外部 API 可能超时或返回错误。建议使用 `FaultTolerantToolbox` 替代普通 `Toolbox`，它会捕获工具执行异常并将错误信息返回给 LLM，而不是中断整个请求。这样即使天气 API 暂时不可用，LLM 仍然可以基于已有信息给出部分建议。

---

## 完整示例：多轮对话旅行助手

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\Clock\Clock;
use Symfony\AI\Agent\Bridge\Mapbox\Mapbox;
use Symfony\AI\Agent\Bridge\OpenMeteo\OpenMeteo;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\Mistral\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\Clock\Clock as SymfonyClock;
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create();
$platform = PlatformFactory::create($_ENV['MISTRAL_API_KEY'], $httpClient);

$toolbox = new Toolbox([
    new Mapbox($httpClient, $_ENV['MAPBOX_TOKEN']),
    new OpenMeteo($httpClient),
    new Clock(new SymfonyClock()),
]);

$processor = new AgentProcessor($toolbox);
$agent = new Agent(
    $platform, 'mistral-large-latest',
    [
        new SystemPromptInputProcessor(
            '你是智能出行助手。使用地理编码和天气工具为用户提供出行建议。'
            . '回答包含：天气概况、穿衣建议、活动推荐、注意事项。用中文回答。'
        ),
        $processor,
    ],
    [$processor],
);

// 多轮对话示例
$questions = [
    '东京现在天气怎么样？最近想去旅游。',
    '那大阪未来一周呢？我在两个城市之间选。',
    '如果去东京，帮我规划一个 3 天的行程，考虑天气因素。',
];

$allMessages = [];

foreach ($questions as $question) {
    echo "用户：{$question}\n\n";
    $allMessages[] = Message::ofUser($question);

    $result = $agent->call(new MessageBag(...$allMessages));
    $answer = $result->getContent();

    echo "助手：{$answer}\n\n";
    echo str_repeat('─', 50) . "\n\n";
    $allMessages[] = Message::ofAssistant($answer);
}
```

---

## 其他实现方案

工具系统与平台无关，切换 AI 平台只需更换 `PlatformFactory` 和模型名，所有工具桥接无需修改。

### 方案 A：使用 OpenAI

```bash
composer require symfony/ai-openai-platform
```

```bash
export OPENAI_API_KEY="your-openai-api-key"
```

```php
<?php

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());

$agent = new Agent(
    $platform, 'gpt-4o-mini',
    [$systemPrompt, $processor],
    [$processor],
);
```

### 方案 B：使用 Anthropic

```bash
composer require symfony/ai-anthropic-platform
```

```bash
export ANTHROPIC_API_KEY="your-anthropic-api-key"
```

```php
<?php

use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;

$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY'], HttpClient::create());

$agent = new Agent(
    $platform, 'claude-sonnet-4-20250514',
    [$systemPrompt, $processor],
    [$processor],
);
```

### 平台对比

| Bridge 包 | 平台 | 推荐模型 | 安装命令 |
|-----------|------|---------|---------|
| `symfony/ai-mistral-platform` | Mistral AI | `mistral-large-latest` / `mistral-small-latest` | `composer require symfony/ai-mistral-platform` |
| `symfony/ai-openai-platform` | OpenAI | `gpt-4o-mini` / `gpt-4o` | `composer require symfony/ai-openai-platform` |
| `symfony/ai-anthropic-platform` | Anthropic | `claude-sonnet-4-20250514` | `composer require symfony/ai-anthropic-platform` |

> **💡 提示：** 如果对成本敏感，可以使用 `mistral-small-latest` 替代 `mistral-large-latest`。小模型在简单的单城市天气查询场景中表现足够好，而大模型在复杂的多城市对比和行程规划中更有优势。

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Mapbox` geocode | 地名 → 经纬度坐标（支持全球地址） |
| `Mapbox` reverse_geocode | 经纬度 → 地名（参数顺序：经度在前） |
| `OpenMeteo` weather_current | 实时天气：温度、风速、天气状况 |
| `OpenMeteo` weather_forecast | 未来 1-16 天天气预报（含每日最高/最低温度） |
| `Clock` | 当前日期时间（解析"明天""下周"等相对时间） |
| 多工具自动编排 | Agent 根据用户问题自动决定调用顺序和参数 |
| 工具链协作 | 一个问题可能触发 3-6 次工具调用（geocode → weather → 分析） |
| `CachePlatform` | 装饰器模式缓存 LLM 响应，减少重复调用 |
| 平台无关性 | 更换 `PlatformFactory` 即可切换 AI 平台，工具无需改动 |

> **🔒 安全建议：** 天气出行助手安全核查清单：① Mapbox Token 使用环境变量，不要提交到代码仓库；② 对用户输入的坐标做范围校验（纬度 -90~90，经度 -180~180）；③ 限制单个用户的请求频率，防止滥用外部 API 配额；④ 定期轮换 API Key 并监控用量。

---

## 下一步

如果你需要 AI 从多个数据源收集信息并生成结构化报告，请看 [16-automated-report-generation.md](./16-automated-report-generation.md)。
