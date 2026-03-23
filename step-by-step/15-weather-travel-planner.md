# 智能天气出行助手

## 业务场景

你在做一个旅行服务 App。用户输入出发地和目的地，AI 自动查询目的地坐标、获取实时天气和未来几天的天气预报，综合给出穿衣建议、出行建议和行程规划。这是一个将多种工具（地理编码、天气、时钟）组合使用的典型场景。

**典型应用：** 旅行助手、出行规划、户外活动推荐、物流天气预警

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台 |
| **Agent** | 智能体框架，自动编排多工具调用 |
| **Agent Bridge Mapbox** | 地理编码：地址 → 经纬度坐标 |
| **Agent Bridge OpenMeteo** | 天气查询：实时天气 + 未来预报 |
| **Agent Bridge Clock** | 当前时间，确保建议时效性 |

## 前置准备

```bash
composer require symfony/ai-platform symfony/ai-platform-openai
composer require symfony/ai-agent
composer require symfony/ai-agent-mapbox     # 地理编码
composer require symfony/ai-agent-open-meteo  # 天气
composer require symfony/ai-agent-clock       # 时钟
```

需要 Mapbox API Token（免费注册）：
```bash
export MAPBOX_TOKEN="your-mapbox-token"
```

---

## Step 1：理解工具链

AI 会自动编排以下调用链：

```
用户："我明天要去杭州旅游" 
  → clock：获取当前日期（确定"明天"是哪天）
  → geocode：将"杭州"转为坐标（30.29, 120.16）
  → weather_forecast：用坐标查询天气预报
  → AI 综合所有数据生成出行建议
```

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
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\Clock\Clock as SymfonyClock;
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create();
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], $httpClient);

// 组合三个工具
$mapbox = new Mapbox($httpClient, $_ENV['MAPBOX_TOKEN']);
$weather = new OpenMeteo($httpClient);
$clock = new Clock(new SymfonyClock());

$toolbox = new Toolbox([$mapbox, $weather, $clock]);
$processor = new AgentProcessor($toolbox);

$systemPrompt = new SystemPromptInputProcessor(
    '你是一个智能出行助手。当用户提到地点时：'
    . "\n1. 先用 geocode 工具将地名转为坐标"
    . "\n2. 用 weather_current 或 weather_forecast 获取天气"
    . "\n3. 综合信息给出出行建议"
    . "\n\n建议包括：穿衣建议、是否需要带伞、适合的活动、出行注意事项。"
    . "\n用中文回答。"
);

$agent = new Agent(
    $platform, 'gpt-4o-mini',
    [$systemPrompt, $processor],
    [$processor],
);
```

---

## Step 3：简单查询 —— 单城市天气

```php
<?php

// 查询单个城市
$result = $agent->call(new MessageBag(
    Message::ofUser('杭州现在天气怎么样？适合户外跑步吗？'),
));

echo "助手：" . $result->getContent() . "\n\n";
```

**AI 自动调用：**
1. `geocode("杭州")` → 获得坐标 (30.29, 120.16)
2. `weather_current(30.29, 120.16)` → 获得当前天气
3. 基于天气数据给出跑步建议

---

## Step 4：复杂查询 —— 旅行规划

```php
<?php

// 旅行规划（涉及多地点和预报）
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

**AI 自动调用：**
1. `clock()` → 确认当前日期
2. `geocode("杭州")` → 获得坐标
3. `weather_forecast(30.29, 120.16, days: 5)` → 获取 5 天预报
4. 综合天气数据生成每天的景点推荐和穿衣建议

---

## Step 5：双城市对比

```php
<?php

// 对比两个城市的天气做出行决策
$result = $agent->call(new MessageBag(
    Message::ofUser(
        '这周末想出去玩一天，在苏州和南京之间犹豫。'
        . '帮我比较一下这两个城市周末的天气，推荐去哪个？'
    ),
));

echo $result->getContent() . "\n";
```

**AI 自动调用：**
1. `clock()` → 确定周末日期
2. `geocode("苏州")` → 坐标 A
3. `geocode("南京")` → 坐标 B
4. `weather_forecast(A, days: 3)` → 苏州天气
5. `weather_forecast(B, days: 3)` → 南京天气
6. 对比分析，推荐更适合出行的城市

---

## Step 6：反向地理编码 —— 根据坐标找地点

```php
<?php

// 用户分享了一个坐标（比如从地图 App 复制的）
$result = $agent->call(new MessageBag(
    Message::ofUser(
        '我目前在坐标 (39.9042, 116.4074) 的位置，'
        . '这是哪里？附近天气怎么样？有什么推荐的？'
    ),
));

echo $result->getContent() . "\n";
```

**AI 自动调用：**
1. `reverse_geocode(116.4074, 39.9042)` → "北京市东城区..."
2. `weather_current(39.9042, 116.4074)` → 当前天气
3. 结合地点和天气给出推荐

---

## 完整示例：旅行出行助手

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
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\Clock\Clock as SymfonyClock;
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create();
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], $httpClient);

$toolbox = new Toolbox([
    new Mapbox($httpClient, $_ENV['MAPBOX_TOKEN']),
    new OpenMeteo($httpClient),
    new Clock(new SymfonyClock()),
]);

$processor = new AgentProcessor($toolbox);
$agent = new Agent(
    $platform, 'gpt-4o-mini',
    [
        new SystemPromptInputProcessor(
            '你是智能出行助手。使用地理编码和天气工具为用户提供出行建议。'
            . '回答包含：天气概况、穿衣建议、活动推荐、注意事项。用中文回答。'
        ),
        $processor,
    ],
    [$processor],
);

// 模拟多轮对话
$questions = [
    '东京现在天气怎么样？最近想去旅游。',
    '那大阪未来一周呢？我在两个城市之间选。',
    '如果去东京，帮我规划一个 3 天的行程，考虑天气因素。',
];

$allMessages = [
    Message::forSystem('你可以使用工具获取信息。'),
];

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

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Mapbox` geocode | 地名 → 经纬度坐标 |
| `Mapbox` reverseGeocode | 经纬度 → 地名 |
| `OpenMeteo` current | 实时天气（温度、风速、天气状况） |
| `OpenMeteo` forecast | 未来 1-16 天天气预报 |
| `Clock` | 当前日期时间（用于计算"明天""下周"等） |
| 工具链自动编排 | AI 自动决定调用顺序和参数 |
| 多工具协作 | 一个问题可能触发 3-5 次工具调用 |

## 下一步

如果你需要 AI 从多个数据源收集信息并生成结构化报告，请看 [16-automated-report-generation.md](./16-automated-report-generation.md)。
