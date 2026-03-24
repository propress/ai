# 工具增强型 AI 助手

## 业务场景

你在开发一个内部运维助手。员工可以向 AI 提问，AI 不仅能聊天，还能实时查询天气、搜索网页、查看当前时间等。当用户的问题需要实时数据时，AI 会自动调用对应工具获取信息，然后基于工具返回的数据生成回答。

**典型应用：** 企业内部助手、智能运维、个人效率助手、带搜索能力的聊天机器人

## 涉及模块

| 模块 | Composer 包 | 用途 |
|------|------------|------|
| **Platform** | `symfony/ai-platform` + `symfony/ai-gemini-platform` | 连接 AI 平台，发送消息与接收回复 |
| **Agent** | `symfony/ai-agent` | 管理工具调用流程，自动判断何时需要调用工具 |
| **Agent Bridge（各工具桥接）** | `symfony/ai-agent-clock-bridge` 等 | 提供具体工具实现（时钟、搜索、维基百科等） |

> **💡 提示：** 本教程使用 **Google Gemini** 作为主要平台。第 01 篇使用 OpenAI，第 02 篇使用 Anthropic，本篇使用 Gemini 以展示不同平台的对接方式。工具系统与具体平台无关，换成其他 Bridge 只需改一行代码。

## 项目流程图

```
┌──────────┐      ┌───────────────┐      ┌──────────────────┐      ┌────────────┐
│  用户输入  │ ──▶ │ InputProcessor │ ──▶ │ Agent 调用 LLM    │ ──▶ │ Gemini API │
│ (文本问题) │      │ (注入工具列表)  │      │ (携带工具定义)     │      │ (AI 推理)   │
└──────────┘      └───────────────┘      └──────────────────┘      └─────┬──────┘
                                                                          │
                                                                          ▼
                                                                  ┌──────────────┐
                                                            ┌──── │ LLM 返回结果   │
                                                            │     └──────────────┘
                                                            │
                                    ┌───────────────────────┼───────────────────────┐
                                    │                       │                       │
                                    ▼                       ▼                       ▼
                           ┌──────────────┐       ┌───────────────┐       ┌──────────────┐
                           │ 直接文本回复   │       │ 请求调用工具    │       │ 请求调用多个   │
                           │ (无需工具)     │       │ (单个工具)     │       │ 工具(并行)     │
                           └──────┬───────┘       └───────┬───────┘       └──────┬───────┘
                                  │                       │                       │
                                  │                       ▼                       ▼
                                  │               ┌───────────────┐       ┌───────────────┐
                                  │               │ AgentProcessor │       │ AgentProcessor │
                                  │               │ 拦截 + 执行工具 │       │ 逐个执行工具    │
                                  │               └───────┬───────┘       └───────┬───────┘
                                  │                       │                       │
                                  │                       ▼                       ▼
                                  │               ┌───────────────┐       ┌───────────────┐
                                  │               │ 工具结果发回LLM │       │ 全部结果发回LLM │
                                  │               │ → 生成最终回复  │       │ → 生成最终回复  │
                                  │               └───────┬───────┘       └───────┬───────┘
                                  │                       │                       │
                                  ▼                       ▼                       ▼
                              ┌──────────────────────────────────────────────────────┐
                              │              OutputProcessor (后处理回复)              │
                              └──────────────────────────┬───────────────────────────┘
                                                         │
                                                         ▼
                                                  ┌──────────────┐
                                                  │  返回给用户    │
                                                  └──────────────┘
```

用户全程无感知，只看到最终回答。

> **💡 提示：** `AgentProcessor` 同时实现了 `InputProcessorInterface` 和 `OutputProcessorInterface`。输入阶段注入工具定义，输出阶段拦截工具调用并执行。这就是为什么它同时出现在 Agent 的输入和输出处理器列表中。

---

## 前置准备

### 环境要求

- PHP 8.2+
- Composer
- Google Gemini API 密钥（[获取地址](https://aistudio.google.com/apikey)）

### 安装依赖

```bash
composer require symfony/ai-agent symfony/ai-gemini-platform symfony/ai-agent-clock-bridge symfony/ai-agent-wikipedia-bridge
```

### 设置 API 密钥

```bash
export GEMINI_API_KEY="your-gemini-api-key"
```

> **🔒 安全建议：** 不要把 API 密钥硬编码在源码中。使用环境变量或 Symfony Secrets 管理敏感信息。

---

## Step 1：使用内置工具 —— Clock

最简单的工具示例：让 AI 能告诉用户当前时间。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\Clock\Clock;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\Clock\Clock as SymfonyClock;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create($_ENV['GEMINI_API_KEY'], HttpClient::create());

// 1. 创建工具实例
$clock = new Clock(new SymfonyClock());

// 2. 把工具放入 Toolbox
$toolbox = new Toolbox([$clock]);

// 3. 创建 AgentProcessor（它负责工具调用的拦截和处理）
$processor = new AgentProcessor($toolbox);

// 4. 创建 Agent，传入 processor 作为输入和输出处理器
$agent = new Agent($platform, 'gemini-2.0-flash', [$processor], [$processor]);

// 5. 用户提问 —— 涉及时间，AI 会自动调用 clock 工具
$messages = new MessageBag(
    Message::ofUser('现在几点了？'),
);
$result = $agent->call($messages);

echo $result->getContent() . "\n";
// 输出示例："现在是 2025 年 1 月 15 日下午 3 点 42 分。"
```

**发生了什么？**
1. 用户问"现在几点了？"
2. LLM 判断需要调用 `clock` 工具
3. Agent 自动调用 Clock 类，获取当前时间
4. LLM 根据时间数据生成自然语言回复

> **💡 提示：** `AgentProcessor` 的构造函数还接受可选参数：`$eventDispatcher` 用于监听工具调用事件，`$maxToolCalls` 用于限制单次对话中的工具调用次数，避免无限循环。

---

## Step 2：组合多个工具

一个 Agent 可以同时拥有多个工具。LLM 会根据用户的问题自动选择合适的工具。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\Clock\Clock;
use Symfony\AI\Agent\Bridge\Wikipedia\Wikipedia;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\Clock\Clock as SymfonyClock;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create($_ENV['GEMINI_API_KEY'], HttpClient::create());

// 组合多个工具
$clock = new Clock(new SymfonyClock());
$wikipedia = new Wikipedia(HttpClient::create());

$toolbox = new Toolbox([$clock, $wikipedia]);
$processor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'gemini-2.0-flash', [$processor], [$processor]);

// === 场景 1：问时间 → AI 调用 clock 工具 ===
$result = $agent->call(new MessageBag(
    Message::ofUser('现在是什么时间？'),
));
echo "回答 1：" . $result->getContent() . "\n\n";

// === 场景 2：问知识 → AI 调用 wikipedia 工具 ===
$result = $agent->call(new MessageBag(
    Message::ofUser('Symfony 框架是什么？请用中文简单介绍。'),
));
echo "回答 2：" . $result->getContent() . "\n\n";

// === 场景 3：普通闲聊 → AI 不调用任何工具，直接回复 ===
$result = $agent->call(new MessageBag(
    Message::ofUser('你好，今天心情怎么样？'),
));
echo "回答 3：" . $result->getContent() . "\n\n";
```

> **💡 提示：** 框架提供了丰富的内置工具桥接：Clock、Wikipedia、OpenMeteo（天气）、Brave/Tavily/SerpApi（搜索）、YouTube（视频字幕）、Firecrawl/Scraper（网页抓取）、Filesystem（文件操作）、SimilaritySearch（向量搜索）等。每个桥接都是独立的 Composer 包，按需安装即可。

---

## Step 3：创建自定义工具

在真实业务中，你需要创建自己的工具。比如：查询订单状态、查询库存、发送通知等。

**关键：** 使用 `#[AsTool]` 属性标记类，Agent 会自动识别。

```php
<?php

namespace App\Tool;

use Symfony\AI\Agent\Toolbox\Attribute\AsTool;

/**
 * 查询订单状态的工具。
 * LLM 会看到工具名和描述，根据用户问题决定是否调用。
 */
#[AsTool('query_order_status', description: '根据订单号查询订单的当前状态')]
class OrderStatusTool
{
    /**
     * @param string $orderId 用户的订单号
     */
    public function __invoke(string $orderId): string
    {
        // 在真实应用中，这里查询数据库或调用订单服务 API
        // 此处用模拟数据演示
        $orders = [
            'ORD-001' => ['status' => '已发货', 'tracking' => 'SF1234567890', 'eta' => '2025-01-20'],
            'ORD-002' => ['status' => '处理中', 'tracking' => null, 'eta' => '2025-01-22'],
            'ORD-003' => ['status' => '已送达', 'tracking' => 'YT9876543210', 'eta' => null],
        ];

        if (!isset($orders[$orderId])) {
            return "未找到订单号 {$orderId}，请检查订单号是否正确。";
        }

        $order = $orders[$orderId];
        $info = "订单 {$orderId} 的状态：{$order['status']}";

        if (null !== $order['tracking']) {
            $info .= "，快递单号：{$order['tracking']}";
        }
        if (null !== $order['eta']) {
            $info .= "，预计到达：{$order['eta']}";
        }

        return $info;
    }
}
```

> **⚠️ 注意：** `#[AsTool]` 的 `description` 参数非常重要——LLM 依赖它来判断何时调用该工具。描述要清晰、具体，说明工具的功能和适用场景。模糊的描述会导致 LLM 错误地调用或遗漏该工具。

### 使用自定义工具：

```php
<?php

require 'vendor/autoload.php';

use App\Tool\OrderStatusTool;
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create($_ENV['GEMINI_API_KEY'], HttpClient::create());

$toolbox = new Toolbox([new OrderStatusTool()]);
$processor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'gemini-2.0-flash', [$processor], [$processor]);

// 用户问订单状态
$result = $agent->call(new MessageBag(
    Message::forSystem('你是一个电商客服。帮用户查询订单状态。'),
    Message::ofUser('我的订单号是 ORD-001，到哪了？'),
));
echo $result->getContent() . "\n";
// 输出示例："您的订单 ORD-001 已发货，快递单号是 SF1234567890，预计 2025 年 1 月 20 日到达。"
```

> **💡 提示：** 工具方法的参数通过 PHP 类型声明和 PHPDoc 注释自动映射。`ToolCallArgumentResolver` 使用反射和 Symfony Serializer 将 LLM 传递的 JSON 参数反序列化为正确的 PHP 类型（`string`、`int`、`DateTime`、甚至自定义对象）。

---

## Step 4：一个工具类提供多个工具方法

有时一个业务模块包含多个操作。可以在一个类上标记多个 `#[AsTool]`，指定不同方法。

```php
<?php

namespace App\Tool;

use Symfony\AI\Agent\Toolbox\Attribute\AsTool;

#[AsTool('search_products', description: '根据关键词搜索商品', method: 'search')]
#[AsTool('get_product_detail', description: '根据商品ID获取商品详细信息', method: 'detail')]
#[AsTool('check_stock', description: '检查某商品的库存数量', method: 'stock')]
class ProductTool
{
    /**
     * @param string $keyword 搜索关键词
     */
    public function search(string $keyword): string
    {
        // 模拟搜索结果
        return "搜索'{$keyword}'的结果：1) 无线蓝牙耳机(P001) ¥199 2) 降噪耳机(P002) ¥599 3) 运动耳机(P003) ¥129";
    }

    /**
     * @param string $productId 商品ID
     */
    public function detail(string $productId): string
    {
        $products = [
            'P001' => '无线蓝牙耳机 - ¥199 - 蓝牙5.3，续航8小时，IPX4防水',
            'P002' => '降噪耳机 - ¥599 - 主动降噪，续航30小时，Hi-Res认证',
        ];

        return $products[$productId] ?? "未找到商品 {$productId}";
    }

    /**
     * @param string $productId 商品ID
     */
    public function stock(string $productId): string
    {
        $stocks = ['P001' => 156, 'P002' => 23, 'P003' => 0];
        $qty = $stocks[$productId] ?? -1;

        if (-1 === $qty) {
            return "未找到商品 {$productId}";
        }

        return 0 === $qty
            ? "商品 {$productId} 已售罄"
            : "商品 {$productId} 库存：{$qty} 件";
    }
}
```

### 使用多方法工具的对话：

```php
$toolbox = new Toolbox([new ProductTool()]);
$processor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'gemini-2.0-flash', [$processor], [$processor]);

// 模拟用户购物对话
$messages = new MessageBag(
    Message::forSystem('你是一个电商购物助手。帮用户找商品、查库存。'),
    Message::ofUser('我想买一个蓝牙耳机，有什么推荐的？'),
);

$result = $agent->call($messages);
echo "助手：" . $result->getContent() . "\n\n";
// AI 会自动调用 search_products 工具，搜索"蓝牙耳机"

// 继续问细节
$messages = new MessageBag(
    Message::forSystem('你是一个电商购物助手。'),
    Message::ofUser('P002 降噪耳机还有货吗？详细参数是什么？'),
);

$result = $agent->call($messages);
echo "助手：" . $result->getContent() . "\n";
// AI 会调用 check_stock 和 get_product_detail 两个工具
```

> **💡 提示：** `#[AsTool]` 是一个可重复属性（`IS_REPEATABLE`），所以同一个类上可以标记任意多个工具。每个标记通过 `method` 参数指向不同的方法。不指定 `method` 时默认调用 `__invoke`。

---

## Step 5：使用 FaultTolerantToolbox 处理工具错误

在生产环境中，工具可能会失败（API 超时、服务不可用等）。`FaultTolerantToolbox` 会捕获异常并让 AI 优雅地处理错误。

```php
use Symfony\AI\Agent\Toolbox\FaultTolerantToolbox;

// 用 FaultTolerantToolbox 包装，工具出错时不会中断对话
$faultTolerantToolbox = new FaultTolerantToolbox($toolbox);
$processor = new AgentProcessor($faultTolerantToolbox);
$agent = new Agent($platform, 'gemini-2.0-flash', [$processor], [$processor]);

// 即使工具调用失败，AI 也会告诉用户"暂时无法查询"而不是崩溃
```

> **🏭 生产建议：** 生产环境强烈建议使用 `FaultTolerantToolbox`。它的工作原理是：当工具抛出异常时，将错误信息转换为 `ToolResult` 返回给 LLM，LLM 会据此生成友好的提示（如"暂时无法查询"）。如果工具不存在，它还会返回可用工具列表帮助 LLM 重新选择。

---

## Step 6：使用工具事件监听

`AgentProcessor` 支持通过 Symfony EventDispatcher 监听工具调用的全生命周期，便于日志记录、安全审计和调试。

### 工具事件生命周期

```
ToolCallRequested → ToolCallArgumentsResolved → ToolCallSucceeded / ToolCallFailed → ToolCallsExecuted
```

| 事件 | 触发时机 | 用途 |
|------|---------|------|
| `ToolCallRequested` | 工具即将被调用 | 安全拦截、审计日志、拒绝危险操作 |
| `ToolCallArgumentsResolved` | 参数已反序列化 | 参数验证、调试 |
| `ToolCallSucceeded` | 工具执行成功 | 成功日志、性能监控 |
| `ToolCallFailed` | 工具执行失败 | 错误报警、异常追踪 |
| `ToolCallsExecuted` | 一批工具调用完成 | 批量后处理、覆盖返回结果 |

### 示例：日志与安全拦截

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Event\ToolCallRequested;
use Symfony\AI\Agent\Toolbox\Event\ToolCallSucceeded;
use Symfony\AI\Agent\Toolbox\Event\ToolCallFailed;
use Symfony\Component\EventDispatcher\EventDispatcher;

$dispatcher = new EventDispatcher();

// 记录所有工具调用
$dispatcher->addListener(ToolCallRequested::class, function (ToolCallRequested $event) {
    $toolCall = $event->getToolCall();
    echo "[日志] 工具请求：{$toolCall->getName()}，参数：" . json_encode($toolCall->getArguments()) . "\n";

    // 安全拦截：拒绝危险操作
    if ('delete_user' === $toolCall->getName()) {
        $event->deny('不允许通过 AI 助手删除用户');
    }
});

// 记录成功
$dispatcher->addListener(ToolCallSucceeded::class, function (ToolCallSucceeded $event) {
    echo "[日志] 工具成功：{$event->getMetadata()->name}\n";
});

// 记录失败
$dispatcher->addListener(ToolCallFailed::class, function (ToolCallFailed $event) {
    echo "[告警] 工具失败：{$event->getMetadata()->name}，原因：{$event->getException()->getMessage()}\n";
});

// 将 dispatcher 传给 AgentProcessor
$processor = new AgentProcessor($toolbox, eventDispatcher: $dispatcher);
$agent = new Agent($platform, 'gemini-2.0-flash', [$processor], [$processor]);
```

> **🔒 安全建议：** `ToolCallRequested` 事件的 `deny()` 方法是工具安全的第一道防线。对于敏感操作（删除数据、发送邮件、执行命令等），务必通过事件监听器进行权限校验，在调用执行前拦截非法操作。被拒绝的调用会将拒绝原因作为结果返回给 LLM。

---

## Step 7：InputProcessor 与 OutputProcessor 管线

`Agent` 的输入和输出通过处理器管线依次处理。`AgentProcessor` 只是其中之一，你还可以组合其他处理器来增强功能。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;

$toolbox = new Toolbox([$clock, $wikipedia]);
$agentProcessor = new AgentProcessor($toolbox);

// SystemPromptInputProcessor：自动注入系统提示词
// 还可以将工具列表自动附加到系统提示中，增强 LLM 的工具理解
$systemPromptProcessor = new SystemPromptInputProcessor(
    '你是一个智能运维助手，可以查时间和搜索知识。用中文简洁回答。',
    $toolbox, // 可选：自动将工具描述附加到 system prompt
);

// 组合多个处理器
$agent = new Agent(
    $platform,
    'gemini-2.0-flash',
    inputProcessors: [$systemPromptProcessor, $agentProcessor],  // 输入管线
    outputProcessors: [$agentProcessor],                         // 输出管线
);

// 现在不需要在 MessageBag 中手动添加 system message
$result = $agent->call(new MessageBag(
    Message::ofUser('现在几点了？'),
));
```

> **💡 提示：** `InputProcessorInterface` 和 `OutputProcessorInterface` 构成了 Agent 的处理管线。输入阶段可用的处理器还有：`ModelOverrideInputProcessor`（动态切换模型）、`MemoryInputProcessor`（注入对话记忆）等。你也可以实现自己的处理器，例如添加用户上下文、进行输入过滤。

---

## 完整示例：带工具的运维助手

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\Clock\Clock;
use Symfony\AI\Agent\Bridge\Wikipedia\Wikipedia;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Event\ToolCallRequested;
use Symfony\AI\Agent\Toolbox\Event\ToolCallSucceeded;
use Symfony\AI\Agent\Toolbox\Event\ToolCallFailed;
use Symfony\AI\Agent\Toolbox\FaultTolerantToolbox;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\Clock\Clock as SymfonyClock;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create();
$platform = PlatformFactory::create($_ENV['GEMINI_API_KEY'], $httpClient);

// 1. 注册工具
$toolbox = new Toolbox([
    new Clock(new SymfonyClock()),
    new Wikipedia($httpClient),
    // 在真实场景中加入你的自定义工具：
    // new OrderStatusTool(),
    // new ProductTool(),
]);

// 2. 用 FaultTolerantToolbox 包装（生产推荐）
$faultTolerantToolbox = new FaultTolerantToolbox($toolbox);

// 3. 配置事件监听（可选，用于日志和安全审计）
$dispatcher = new EventDispatcher();
$dispatcher->addListener(ToolCallRequested::class, function (ToolCallRequested $event) {
    echo "  → 调用工具：{$event->getToolCall()->getName()}\n";
});
$dispatcher->addListener(ToolCallSucceeded::class, function (ToolCallSucceeded $event) {
    echo "  ✓ 工具成功：{$event->getMetadata()->name}\n";
});
$dispatcher->addListener(ToolCallFailed::class, function (ToolCallFailed $event) {
    echo "  ✗ 工具失败：{$event->getMetadata()->name}\n";
});

// 4. 创建 AgentProcessor
$processor = new AgentProcessor($faultTolerantToolbox, eventDispatcher: $dispatcher);

// 5. 创建 Agent
$agent = new Agent($platform, 'gemini-2.0-flash', [$processor], [$processor]);

// 6. 模拟多轮对话
$questions = [
    '现在几点了？',                              // → 调用 clock 工具
    'PHP 是什么编程语言？帮我查一下。',            // → 调用 wikipedia 工具
    '谢谢你的帮助！',                             // → 不调用工具，直接回复
];

$allMessages = [
    Message::forSystem('你是一个全能助手，可以查时间、搜索知识。用中文简洁回答。'),
];

foreach ($questions as $question) {
    echo "用户：{$question}\n";

    $allMessages[] = Message::ofUser($question);
    $result = $agent->call(new MessageBag(...$allMessages));
    $answer = $result->getContent();

    echo "助手：{$answer}\n\n";
    $allMessages[] = Message::ofAssistant($answer);
}
```

---

## 其他实现方案

工具系统与平台无关，切换 AI 平台只需更换 `PlatformFactory` 和模型名。

### 方案 A：使用 OpenAI

```bash
composer require symfony/ai-openai-platform
```

```bash
export OPENAI_API_KEY="your-openai-api-key"
```

```php
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());
$agent = new Agent($platform, 'gpt-4o-mini', [$processor], [$processor]);
```

### 方案 B：使用 Anthropic

```bash
composer require symfony/ai-anthropic-platform
```

```bash
export ANTHROPIC_API_KEY="your-anthropic-api-key"
```

```php
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;

$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY'], HttpClient::create());
$agent = new Agent($platform, 'claude-sonnet-4-20250514', [$processor], [$processor]);
```

### 平台对比

| Bridge 包 | 平台 | 推荐模型 | 安装命令 |
|-----------|------|---------|---------|
| `symfony/ai-gemini-platform` | Google Gemini | `gemini-2.0-flash` / `gemini-2.5-pro` | `composer require symfony/ai-gemini-platform` |
| `symfony/ai-openai-platform` | OpenAI | `gpt-4o-mini` / `gpt-4o` | `composer require symfony/ai-openai-platform` |
| `symfony/ai-anthropic-platform` | Anthropic | `claude-sonnet-4-20250514` | `composer require symfony/ai-anthropic-platform` |

> **💡 提示：** 所有平台的工具定义格式由框架自动转换，你的 `#[AsTool]` 标记的工具代码无需任何修改即可在不同平台间切换。

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `#[AsTool]` | PHP 属性，标记一个类/方法为 AI 可用工具。支持 `name`、`description`、`method` 参数 |
| `Toolbox` | 工具容器，管理所有可用工具，实现 `ToolboxInterface` |
| `AgentProcessor` | 同时实现输入处理器和输出处理器，输入时注入工具定义，输出时拦截并执行工具调用 |
| `Agent` | 智能体，整合 Platform、输入处理器管线和输出处理器管线 |
| `FaultTolerantToolbox` | 容错工具箱装饰器，捕获异常转为友好错误信息返回给 LLM |
| `ToolCallArgumentResolver` | 使用反射和 Symfony Serializer 将 LLM 的 JSON 参数反序列化为 PHP 类型 |
| 工具事件 | `ToolCallRequested` → `ToolCallArgumentsResolved` → `ToolCallSucceeded`/`ToolCallFailed` → `ToolCallsExecuted` |
| `InputProcessorInterface` | 输入管线接口，用于在消息发送前注入系统提示、工具定义、记忆等 |
| `OutputProcessorInterface` | 输出管线接口，用于在 LLM 响应后执行工具调用、后处理等 |
| 多方法工具 | 一个类通过多个 `#[AsTool]` 提供多个工具，各指向不同方法 |

> **🔒 安全建议：** 工具执行安全核查清单：① 通过 `ToolCallRequested` 事件拦截敏感操作；② 工具内部验证参数合法性，不要信任 LLM 传入的参数；③ 数据库操作使用参数化查询防止注入；④ 文件操作限制路径范围；⑤ 使用 `$maxToolCalls` 防止 Agent 陷入无限工具调用循环。

---

## 下一步

工具让 AI 能获取实时数据，但如果你有大量的文档（产品手册、FAQ 文档、法律条文等）需要 AI 基于这些文档回答，就需要 RAG（检索增强生成）。请看 [04-rag-knowledge-base.md](./04-rag-knowledge-base.md)。
