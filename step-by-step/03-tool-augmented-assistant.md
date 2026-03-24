# 工具增强型 AI 助手

## 业务场景

你在开发一个内部运维助手。员工可以向 AI 提问，AI 不仅能聊天，还能实时查询天气、搜索网页、查看当前时间等。当用户的问题需要实时数据时，AI 会自动调用对应工具获取信息，然后基于工具返回的数据生成回答。

**典型应用：** 企业内部助手、智能运维、个人效率助手、带搜索能力的聊天机器人

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台 |
| **Agent** | 管理工具调用流程，自动判断何时需要调用工具 |
| **Agent Bridge（各工具桥接）** | 提供具体工具实现（时钟、搜索、维基百科等） |

## 工具调用流程

理解工具调用的完整流程很重要：

```
1. 用户发消息 → Agent 转发给 LLM
2. LLM 判断是否需要工具 →
   a. 不需要：直接回复文本
   b. 需要：返回"请调用 XX 工具，参数是 YY"
3. Agent 自动执行工具调用，获取结果
4. Agent 把工具结果发回 LLM
5. LLM 根据工具结果生成最终回复
6. Agent 返回回复给用户
```

用户全程无感知，只看到最终回答。

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
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\Clock\Clock as SymfonyClock;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());

// 1. 创建工具实例
$clock = new Clock(new SymfonyClock());

// 2. 把工具放入 Toolbox
$toolbox = new Toolbox([$clock]);

// 3. 创建 AgentProcessor（它负责工具调用的拦截和处理）
$processor = new AgentProcessor($toolbox);

// 4. 创建 Agent，传入 processor 作为输入和输出处理器
$agent = new Agent($platform, 'gpt-4o-mini', [$processor], [$processor]);

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
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\Clock\Clock as SymfonyClock;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());

// 组合多个工具
$clock = new Clock(new SymfonyClock());
$wikipedia = new Wikipedia(HttpClient::create());

$toolbox = new Toolbox([$clock, $wikipedia]);
$processor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'gpt-4o-mini', [$processor], [$processor]);

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

### 使用自定义工具：

```php
<?php

require 'vendor/autoload.php';

use App\Tool\OrderStatusTool;
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());

$toolbox = new Toolbox([new OrderStatusTool()]);
$processor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'gpt-4o-mini', [$processor], [$processor]);

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
$agent = new Agent($platform, 'gpt-4o-mini', [$processor], [$processor]);

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

---

## Step 5：使用 FaultTolerantToolbox 处理工具错误

在生产环境中，工具可能会失败（API 超时、服务不可用等）。`FaultTolerantToolbox` 会捕获异常并让 AI 优雅地处理错误。

```php
use Symfony\AI\Agent\Toolbox\FaultTolerantToolbox;

// 用 FaultTolerantToolbox 包装，工具出错时不会中断对话
$faultTolerantToolbox = new FaultTolerantToolbox($toolbox);
$processor = new AgentProcessor($faultTolerantToolbox);
$agent = new Agent($platform, 'gpt-4o-mini', [$processor], [$processor]);

// 即使工具调用失败，AI 也会告诉用户"暂时无法查询"而不是崩溃
```

---

## 完整示例：带工具的运维助手

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\Clock\Clock;
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

$toolbox = new Toolbox([
    new Clock(new SymfonyClock()),
    new Wikipedia($httpClient),
    // 在真实场景中加入你的自定义工具：
    // new OrderStatusTool(),
    // new ProductTool(),
]);
$processor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'gpt-4o-mini', [$processor], [$processor]);

// 模拟多轮对话，展示不同类型问题的处理
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

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `#[AsTool]` | PHP 属性，标记一个类/方法为 AI 可用工具 |
| `Toolbox` | 工具容器，管理所有可用工具 |
| `AgentProcessor` | 拦截 AI 的工具调用请求并执行 |
| `Agent` | 智能体，整合 Platform 和工具调用流程 |
| `FaultTolerantToolbox` | 容错工具箱，工具出错不会中断对话 |
| 工具参数 | 通过方法参数和 PHPDoc 注释定义，AI 自动识别 |
| 多方法工具 | 一个类通过多个 `#[AsTool]` 提供多个工具 |

## 下一步

工具让 AI 能获取实时数据，但如果你有大量的文档（产品手册、FAQ 文档、法律条文等）需要 AI 基于这些文档回答，就需要 RAG（检索增强生成）。请看 [04-rag-knowledge-base.md](./04-rag-knowledge-base.md)。
