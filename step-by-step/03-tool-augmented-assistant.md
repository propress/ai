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

> **💡 提示：** 本教程使用 **Google Gemini** 作为主要平台。第 01 篇使用 Anthropic/OpenAI，第 02 篇使用 Anthropic，本篇使用 Gemini 以展示不同平台的对接方式。工具系统与具体平台无关，换成其他 Bridge 只需改一行代码。

## 架构概述

Symfony AI 的工具系统由三个核心组件组成，形成一个**闭环工具调用系统**：

- **`Toolbox`**（工具容器）实现 `ToolboxInterface`，持有所有可用工具的元数据和实例。它负责根据 `ToolCall` 查找对应工具并执行。`Toolbox` 内部通过 `ReflectionToolFactory` 从 `#[AsTool]` 属性和 PHP 类型信息中自动提取工具的名称、描述和参数 JSON Schema
- **`AgentProcessor`** 同时实现 `InputProcessorInterface` 和 `OutputProcessorInterface`——输入阶段将工具定义注入 LLM 请求，输出阶段拦截 `ToolCallResult`（LLM 要求调用工具的响应）并驱动执行循环
- **`Agent`** 是整个系统的协调者，将 Platform、输入处理器管线、输出处理器管线组装在一起。调用 `$agent->call()` 时，消息依次流过输入处理器 → LLM → 输出处理器

三者协作的关键在于 `AgentProcessor` 的 **do-while 循环**：当 LLM 返回工具调用请求时，处理器执行工具、将结果追加到消息历史、再次调用 LLM——这个过程持续进行，直到 LLM 返回文本回复而非工具调用。这使得 Agent 能够完成多步推理任务（如：先搜索 → 再查详情 → 最后总结）。

> **📝 知识扩展：** `Toolbox` 的执行流程远不只是"调用方法"这么简单。完整流程为：① 根据名称查找工具元数据 → ② 分发 `ToolCallRequested` 事件（可被拦截或拒绝）→ ③ 通过 `ArgumentResolver` 反序列化参数 → ④ 分发 `ToolCallArgumentsResolved` → ⑤ 如果工具实现了 `HasSourcesInterface`，注入 `SourceCollection` → ⑥ 通过反射执行工具方法 → ⑦ 创建 `ToolResult`（含可选的来源引用）→ ⑧ 分发 `ToolCallSucceeded`；若异常则分发 `ToolCallFailed` 并抛出 `ToolExecutionException`。

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
                                 │               ┌────────────────────────────────────┐
                                 │               │ do-while 循环：结果发回 LLM，        │
                                 │               │ 若 LLM 再次请求工具则继续执行，       │
                                 │               │ 直到 LLM 返回文本回复为止            │
                                 │               └───────────────┬──────────────────┘
                                 │                               │
                                 ▼                               ▼
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
$agent = new Agent($platform, 'gemini-2.5-flash', [$processor], [$processor]);

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

### AgentProcessor 构造函数详解

`AgentProcessor` 是工具系统的核心引擎，其构造函数接受多个参数来控制工具调用行为：

```php
$processor = new AgentProcessor(
    toolbox: $toolbox,              // 必需：ToolboxInterface 实例
    resultConverter: new ToolResultConverter(), // 工具结果 → 字符串的转换器
    eventDispatcher: $dispatcher,   // 可选：Symfony 事件调度器，用于监听工具生命周期
    excludeToolMessages: false,     // 为 true 时，工具调用消息不会追加到后续 LLM 请求中
    includeSources: false,          // 为 true 时，收集工具的来源引用（需工具实现 HasSourcesInterface）
    maxToolCalls: null,             // 安全限制：最大工具调用轮数，超出抛出 MaxIterationsExceededException
);
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `$toolbox` | `ToolboxInterface` | 工具容器，提供工具列表和执行能力 |
| `$resultConverter` | `ToolResultConverter` | 将 `ToolResult` 转换为字符串传回 LLM。默认实现对大多数场景足够 |
| `$eventDispatcher` | `?EventDispatcherInterface` | 传入后可监听 `ToolCallRequested` 等事件 |
| `$excludeToolMessages` | `bool` | 优化选项——当工具返回大量数据时，排除历史工具消息可减少 Token 消耗 |
| `$includeSources` | `bool` | 启用后，工具执行时的来源引用会被收集到元数据中 |
| `$maxToolCalls` | `?int` | 防止 Agent 陷入无限工具调用循环的安全阀 |

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
$agent = new Agent($platform, 'gemini-2.5-flash', [$processor], [$processor]);

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

> **💡 提示：** 框架提供了丰富的内置工具桥接：Clock、Wikipedia、OpenMeteo（天气）、Brave/Tavily/SerpApi（搜索）、YouTube（视频字幕）、Firecrawl（网页抓取）、Filesystem（10 种文件操作）、SimilaritySearch（向量搜索）、Mapbox（地理编码）等。每个桥接都是独立的 Composer 包，按需安装即可。

### 使用 `$options['tools']` 限定可用工具

当 Toolbox 中注册了多个工具时，你可以在每次调用中通过 `$options['tools']` 限制 LLM 只能使用特定工具：

```php
// Toolbox 中有 clock、wikipedia_search、wikipedia_article 三个工具
// 但此次调用只允许使用 clock
$result = $agent->call($messages, ['tools' => ['clock']]);

// 另一次调用只允许使用 wikipedia 相关工具
$result = $agent->call($messages, ['tools' => ['wikipedia_search', 'wikipedia_article']]);
```

> **📝 知识扩展：** `AgentProcessor` 在 `processInput()` 中处理工具筛选。当 `$options['tools']` 是一个纯字符串数组时，它会从完整工具列表中过滤出匹配的子集：`array_filter($toolMap, fn (Tool $tool) => in_array($tool->getName(), $options['tools'], true))`。筛选后的工具列表替换原始列表传给 LLM，LLM 只会看到被允许的工具定义。这在多场景共用同一 Agent 时非常有用——同一个 Toolbox 可以在不同业务场景中暴露不同工具子集。

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

`#[AsTool]` 属性的完整签名：

```php
#[\Attribute(\Attribute::TARGET_CLASS | \Attribute::IS_REPEATABLE)]
final class AsTool
{
    public function __construct(
        public readonly string $name,        // 工具名称——LLM 通过它来调用工具
        public readonly string $description, // 工具描述——LLM 依赖它判断何时调用
        public readonly string $method = '__invoke', // 要调用的方法，默认 __invoke
    )
}
```

> **⚠️ 注意：** `description` 参数非常重要——LLM 依赖它来判断何时调用该工具。描述要清晰、具体，说明工具的功能和适用场景。模糊的描述会导致 LLM 错误地调用或遗漏该工具。

> **📝 知识扩展：** `ReflectionToolFactory` 是将 PHP 类转换为 LLM 可理解的工具定义的核心。它通过 PHP 反射机制提取工具元数据：① 读取 `#[AsTool]` 属性获取工具名称、描述和方法名 → ② 反射方法签名获取参数的 PHP 类型（`string`、`int`、`array` 等）→ ③ 解析 PHPDoc 中的 `@param` 标签获取参数描述 → ④ 识别 `#[With]` 属性提取额外约束（如枚举值、正则模式）→ ⑤ 将以上信息组装为 JSON Schema 格式的参数定义。例如 `public function __invoke(string $orderId): string` 加上 `@param string $orderId 用户的订单号` 会生成 `{"type": "object", "properties": {"orderId": {"type": "string", "description": "用户的订单号"}}, "required": ["orderId"]}`。这意味着你只需要写好类型声明和 PHPDoc，无需手动编写 JSON Schema。

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
$agent = new Agent($platform, 'gemini-2.5-flash', [$processor], [$processor]);

// 用户问订单状态
$result = $agent->call(new MessageBag(
    Message::forSystem('你是一个电商客服。帮用户查询订单状态。'),
    Message::ofUser('我的订单号是 ORD-001，到哪了？'),
));
echo $result->getContent() . "\n";
// 输出示例："您的订单 ORD-001 已发货，快递单号是 SF1234567890，预计 2025 年 1 月 20 日到达。"
```

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
$agent = new Agent($platform, 'gemini-2.5-flash', [$processor], [$processor]);

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
// AI 可能调用 check_stock 和 get_product_detail 两个工具——这就是多步工具调用
```

> **💡 提示：** `#[AsTool]` 是一个可重复属性（`IS_REPEATABLE`），所以同一个类上可以标记任意多个工具。每个标记通过 `method` 参数指向不同的方法。不指定 `method` 时默认调用 `__invoke`。

> **📝 知识扩展：** 当 LLM 在一次回复中请求调用 `check_stock` 和 `get_product_detail` 两个工具时，`AgentProcessor` 的 do-while 循环会这样运作：① LLM 返回包含两个 `ToolCall` 的 `ToolCallResult` → ② 循环体 `foreach` 逐个执行工具，将每个结果以 `Message::ofToolCall()` 追加到消息历史 → ③ 分发 `ToolCallsExecuted` 事件 → ④ 再次调用 LLM，传入包含两条工具结果的完整消息历史 → ⑤ LLM 综合两个工具的返回信息生成最终文本回复 → ⑥ 返回的不再是 `ToolCallResult`，循环结束。如果 LLM 看到工具结果后仍需要更多信息（如"库存为 0，去查推荐替代品"），它可以再次请求工具调用——循环继续。`$maxToolCalls` 正是为防止这种链式调用无限进行而设。

---

## Step 5：使用 FaultTolerantToolbox 处理工具错误

在生产环境中，工具可能会失败（API 超时、服务不可用等）。`FaultTolerantToolbox` 会捕获异常并让 AI 优雅地处理错误。

```php
use Symfony\AI\Agent\Toolbox\FaultTolerantToolbox;

// 用 FaultTolerantToolbox 包装，工具出错时不会中断对话
$faultTolerantToolbox = new FaultTolerantToolbox($toolbox);
$processor = new AgentProcessor($faultTolerantToolbox);
$agent = new Agent($platform, 'gemini-2.5-flash', [$processor], [$processor]);

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

| 事件 | 触发时机 | 关键方法 | 用途 |
|------|---------|---------|------|
| `ToolCallRequested` | 工具即将被调用 | `deny()`, `setResult()`, `getToolCall()` | 安全拦截、审计日志、拒绝危险操作、跳过执行直接返回结果 |
| `ToolCallArgumentsResolved` | 参数已反序列化 | `getArguments()`, `getMetadata()` | 参数验证、调试 |
| `ToolCallSucceeded` | 工具执行成功 | `getResult()`, `getMetadata()` | 成功日志、性能监控 |
| `ToolCallFailed` | 工具执行失败 | `getException()`, `getMetadata()` | 错误报警、异常追踪 |
| `ToolCallsExecuted` | 一批工具调用完成 | `setResult()`, `hasResult()`, `getToolResults()` | 批量后处理、**覆盖 LLM 返回结果** |

### 示例：日志与安全拦截

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Event\ToolCallRequested;
use Symfony\AI\Agent\Toolbox\Event\ToolCallSucceeded;
use Symfony\AI\Agent\Toolbox\Event\ToolCallFailed;
use Symfony\AI\Agent\Toolbox\Event\ToolCallsExecuted;
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

// ToolCallsExecuted：可覆盖后续 LLM 调用的结果
$dispatcher->addListener(ToolCallsExecuted::class, function (ToolCallsExecuted $event) {
    // 检查所有工具结果，根据业务逻辑决定是否直接返回
    $results = $event->getToolResults();
    echo "[日志] 本轮执行了 " . count($results) . " 个工具\n";

    // 通过 setResult() 可跳过后续 LLM 调用，直接返回指定结果
    // $event->setResult($customResult);
});

// 将 dispatcher 传给 AgentProcessor
$processor = new AgentProcessor($toolbox, eventDispatcher: $dispatcher);
$agent = new Agent($platform, 'gemini-2.5-flash', [$processor], [$processor]);
```

> **🔒 安全建议：** `ToolCallRequested` 事件的 `deny()` 方法是工具安全的第一道防线。对于敏感操作（删除数据、发送邮件、执行命令等），务必通过事件监听器进行权限校验，在调用执行前拦截非法操作。被拒绝的调用会将拒绝原因作为结果返回给 LLM。

---

## Step 7：工具来源引用（HasSourcesInterface）

某些工具会提供来源引用——例如 Wikipedia 工具返回答案的同时标注文章链接，Clock 标注时间来源。框架通过 `HasSourcesInterface` 和 `SourceCollection` 统一管理这些引用。

```php
// 启用来源收集
$processor = new AgentProcessor(
    $toolbox,
    includeSources: true, // 关键：开启来源收集
);
$agent = new Agent($platform, 'gemini-2.5-flash', [$processor], [$processor]);

$result = $agent->call(new MessageBag(
    Message::ofUser('Symfony 框架是谁创建的？'),
));

echo $result->getContent() . "\n";

// 来源引用会收集在 AgentProcessor 的元数据中
// 在 Symfony Bundle 环境中可通过服务注入获取
```

> **📝 知识扩展：** `HasSourcesInterface` 只定义了一个方法：`setSourceCollection(SourceCollection $sourceMap): void`。内置桥接如 Clock 和 Wikipedia 都实现了此接口（通过 `HasSourcesTrait`）。当 `$includeSources` 为 `true` 时，`Toolbox::execute()` 在调用工具方法前会注入一个空的 `SourceCollection`，工具在执行过程中通过 `$this->addSource(new Source(...))` 向其中添加来源。执行完成后，`ToolResult` 会携带这些来源，`AgentProcessor` 在循环中通过 `$toolResult->getSources()` 收集并合并所有工具的来源引用。这为构建带引用的 AI 回答提供了基础。

---

## Step 8：InputProcessor 与 OutputProcessor 管线

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
    'gemini-2.5-flash',
    inputProcessors: [$systemPromptProcessor, $agentProcessor],  // 输入管线
    outputProcessors: [$agentProcessor],                         // 输出管线
);

// 现在不需要在 MessageBag 中手动添加 system message
$result = $agent->call(new MessageBag(
    Message::ofUser('现在几点了？'),
));
```

> **📝 知识扩展：** 当 `SystemPromptInputProcessor` 接收到一个 `ToolboxInterface` 实例时，它会自动将所有工具的名称和描述附加到系统提示词末尾。内部实现是遍历 `$toolbox->getTools()` 生成格式化文本：`## {tool_name}\n{tool_description}`，并以 `# Tools\nThe following tools are available...` 为标题拼接到系统消息中。这在某些模型上能显著提升工具选择准确率，因为工具信息同时出现在系统提示和函数定义两个位置。

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

// 4. 创建 AgentProcessor（配置安全限制和来源收集）
$processor = new AgentProcessor(
    $faultTolerantToolbox,
    eventDispatcher: $dispatcher,
    includeSources: true,  // 收集工具的来源引用
    maxToolCalls: 10,      // 最多 10 轮工具调用，防止无限循环
);

// 5. 创建 Agent
$agent = new Agent($platform, 'gemini-2.5-flash', [$processor], [$processor]);

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

### 方案 A：使用 Anthropic

```bash
composer require symfony/ai-anthropic-platform
```

```bash
export ANTHROPIC_API_KEY="your-anthropic-api-key"
```

```php
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;

$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY']);
$agent = new Agent($platform, 'claude-sonnet-4-20250514', [$processor], [$processor]);
```

### 方案 B：使用 OpenAI

```bash
composer require symfony/ai-openai-platform
```

```bash
export OPENAI_API_KEY="your-openai-api-key"
```

```php
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);
$agent = new Agent($platform, 'gpt-4.1-mini', [$processor], [$processor]);
```

### 平台对比

| Bridge 包 | 平台 | 推荐模型 | 安装命令 |
|-----------|------|---------|---------|
| `symfony/ai-gemini-platform` | Google Gemini | `gemini-2.5-flash` / `gemini-2.5-pro` | `composer require symfony/ai-gemini-platform` |
| `symfony/ai-anthropic-platform` | Anthropic | `claude-sonnet-4-20250514` | `composer require symfony/ai-anthropic-platform` |
| `symfony/ai-openai-platform` | OpenAI | `gpt-4.1-mini` / `gpt-4.1` | `composer require symfony/ai-openai-platform` |
| `symfony/ai-mistral-platform` | Mistral | `mistral-large-latest` | `composer require symfony/ai-mistral-platform` |

> **💡 提示：** 所有平台的工具定义格式由框架自动转换，你的 `#[AsTool]` 标记的工具代码无需任何修改即可在不同平台间切换。

---

## 关键知识点总结

| # | 概念 | 说明 |
|---|------|------|
| 1 | `#[AsTool]` | PHP 属性，标记一个类/方法为 AI 可用工具。支持 `name`、`description`、`method` 参数，可重复标记 |
| 2 | `Toolbox` / `ToolboxInterface` | 工具容器，管理所有可用工具，提供 `getTools()` 返回工具列表、`execute()` 执行工具调用 |
| 3 | `AgentProcessor` | 同时实现输入处理器和输出处理器，输入时注入工具定义，输出时驱动 do-while 循环执行工具调用 |
| 4 | `Agent` | 智能体，整合 Platform、输入处理器管线和输出处理器管线，`call()` 是核心入口 |
| 5 | `ReflectionToolFactory` | 通过 PHP 反射从类型声明、PHPDoc `@param` 和 `#[With]` 属性提取参数信息，生成 JSON Schema |
| 6 | do-while 工具调用循环 | `AgentProcessor` 持续执行工具调用直到 LLM 返回文本而非 `ToolCallResult`，支持多步推理 |
| 7 | `$options['tools']` | 在 `$agent->call()` 时传入工具名称数组，限制本次调用可用的工具子集 |
| 8 | `HasSourcesInterface` / `SourceCollection` | 工具实现此接口可提供来源引用，`$includeSources` 开启后 `ToolResult` 携带来源信息 |
| 9 | `ToolCallsExecuted` 事件 | 一批工具执行完成后触发，`setResult()` 可跳过后续 LLM 调用直接返回指定结果 |
| 10 | `$maxToolCalls` | 安全限制，超出最大工具调用轮数时抛出 `MaxIterationsExceededException`，防止无限循环 |
| 11 | `$excludeToolMessages` | 优化选项，为 `true` 时工具调用消息不追加到后续请求，减少 Token 消耗 |
| 12 | `$includeSources` | 开启后收集所有工具的 `SourceCollection`，用于构建带引用的回答 |
| 13 | `FaultTolerantToolbox` | 容错装饰器，捕获异常转为友好错误信息返回给 LLM，工具不存在时返回可用列表 |
| 14 | 工具事件生命周期 | `ToolCallRequested`（可拦截）→ `ToolCallArgumentsResolved` → `ToolCallSucceeded`/`ToolCallFailed` → `ToolCallsExecuted`（可覆盖结果） |

> **🔒 安全建议：** 工具执行安全核查清单：① 通过 `ToolCallRequested` 事件拦截敏感操作；② 工具内部验证参数合法性，不要信任 LLM 传入的参数；③ 数据库操作使用参数化查询防止注入；④ 文件操作限制路径范围；⑤ 使用 `$maxToolCalls` 防止 Agent 陷入无限工具调用循环。

---

## 下一步

工具让 AI 能获取实时数据，但如果你有大量的文档（产品手册、FAQ 文档、法律条文等）需要 AI 基于这些文档回答，就需要 RAG（检索增强生成）。请看 [04-rag-knowledge-base.md](./04-rag-knowledge-base.md)。
