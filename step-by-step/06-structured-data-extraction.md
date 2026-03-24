# 结构化数据提取

## 业务场景

你在做一个智能表单系统。用户用自然语言描述需求，AI 自动将描述转为结构化的数据对象。比如用户说"我要订明天下午三点从北京飞上海的机票，一个成人"，系统需要提取出出发城市、目的城市、时间、人数等结构化字段。

**典型应用：** 智能表单填充、数据录入自动化、简历解析、订单信息提取、内容分类打标

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台，使用结构化输出能力 |
| **StructuredOutput** | 定义输出 schema，让 AI 返回 PHP 对象而非自由文本 |

## 前置准备

```bash
composer require symfony/ai-platform symfony/ai-platform-openai
composer require symfony/event-dispatcher  # 结构化输出需要事件分发器
```

---

## Step 1：理解结构化输出

普通调用 AI 返回自由文本字符串。结构化输出让 AI 返回符合预定义 schema 的 JSON，自动映射为 PHP 对象。

```
普通输出：  "北京到上海，明天下午三点，1个成人"（自由文本，需要额外解析）
结构化输出：FlightBooking { from: "北京", to: "上海", date: "2025-01-16", time: "15:00", passengers: 1 }
```

---

## Step 2：定义输出结构（PHP 类）

创建一个 PHP 类来表示你期望的输出结构。

```php
<?php

namespace App\Dto;

/**
 * 机票预订信息。
 */
final class FlightBooking
{
    /**
     * @param string   $departureCity   出发城市
     * @param string   $arrivalCity     到达城市
     * @param string   $date            出发日期（YYYY-MM-DD 格式）
     * @param string   $time            出发时间（HH:MM 格式）
     * @param int      $passengers      乘客人数
     * @param string   $cabinClass      舱位等级（economy/business/first）
     * @param string[] $specialRequests 特殊需求列表
     */
    public function __construct(
        public readonly string $departureCity,
        public readonly string $arrivalCity,
        public readonly string $date,
        public readonly string $time,
        public readonly int $passengers,
        public readonly string $cabinClass = 'economy',
        public readonly array $specialRequests = [],
    ) {
    }
}
```

---

## Step 3：使用结构化输出调用 AI

```php
<?php

require 'vendor/autoload.php';

use App\Dto\FlightBooking;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

// 1. 结构化输出需要注册 PlatformSubscriber
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

$platform = PlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);

// 2. 用户的自然语言输入
$userInput = '帮我订一张后天下午两点半从上海飞广州的机票，商务舱，2个大人，需要靠窗座位。';

$messages = new MessageBag(
    Message::forSystem('你是一个机票预订助手。从用户的描述中提取预订信息。今天是 2025-01-15。'),
    Message::ofUser($userInput),
);

// 3. 指定 response_format 为 PHP 类
$result = $platform->invoke('gpt-4o-mini', $messages, [
    'response_format' => FlightBooking::class,
]);

// 4. 获取结构化 PHP 对象
$booking = $result->asObject();

echo "出发城市：{$booking->departureCity}\n";
echo "到达城市：{$booking->arrivalCity}\n";
echo "日期：{$booking->date}\n";
echo "时间：{$booking->time}\n";
echo "人数：{$booking->passengers}\n";
echo "舱位：{$booking->cabinClass}\n";
echo "特殊需求：" . implode('、', $booking->specialRequests) . "\n";
```

**输出：**
```
出发城市：上海
到达城市：广州
日期：2025-01-17
时间：14:30
人数：2
舱位：business
特殊需求：靠窗座位
```

---

## Step 4：复杂嵌套结构

真实业务通常需要嵌套的数据结构。

```php
<?php

namespace App\Dto;

final class ContactInfo
{
    public function __construct(
        public readonly string $name,
        public readonly string $email,
        public readonly ?string $phone = null,
    ) {
    }
}

final class OrderItem
{
    /**
     * @param string $productName 商品名称
     * @param int    $quantity    数量
     * @param float  $unitPrice   单价（元）
     */
    public function __construct(
        public readonly string $productName,
        public readonly int $quantity,
        public readonly float $unitPrice,
    ) {
    }
}

final class Order
{
    /**
     * @param ContactInfo $customer     客户信息
     * @param OrderItem[] $items        订单商品列表
     * @param string      $deliveryAddr 配送地址
     * @param string      $urgency      紧急程度（normal/urgent/express）
     * @param ?string     $notes        备注
     */
    public function __construct(
        public readonly ContactInfo $customer,
        public readonly array $items,
        public readonly string $deliveryAddr,
        public readonly string $urgency = 'normal',
        public readonly ?string $notes = null,
    ) {
    }
}
```

### 从邮件中提取订单：

```php
$email = <<<TEXT
张经理您好，

我是李明（liming@example.com，电话 138-0000-1234），需要订购以下物品：
- A4 复印纸 20 箱，每箱 35 元
- 黑色签字笔 50 支，每支 3 元
- 文件夹 100 个，每个 5 元

配送到：北京市朝阳区建国路88号 3层
比较急，希望今天能发货。

谢谢！
TEXT;

$messages = new MessageBag(
    Message::forSystem('从邮件内容中提取订单信息。'),
    Message::ofUser($email),
);

$result = $platform->invoke('gpt-4o-mini', $messages, [
    'response_format' => Order::class,
]);

$order = $result->asObject();

echo "客户：{$order->customer->name}（{$order->customer->email}）\n";
echo "电话：{$order->customer->phone}\n";
echo "地址：{$order->deliveryAddr}\n";
echo "紧急程度：{$order->urgency}\n";
echo "商品列表：\n";

$total = 0;
foreach ($order->items as $item) {
    $subtotal = $item->quantity * $item->unitPrice;
    $total += $subtotal;
    echo "  - {$item->productName} × {$item->quantity} = ¥{$subtotal}\n";
}
echo "总计：¥{$total}\n";
```

---

## Step 5：使用 JSON Schema（不用 PHP 类）

如果不想定义 PHP 类，也可以直接传 JSON Schema：

```php
$result = $platform->invoke('gpt-4o-mini', $messages, [
    'response_format' => [
        'type' => 'json_schema',
        'json_schema' => [
            'name' => 'sentiment_analysis',
            'strict' => true,
            'schema' => [
                'type' => 'object',
                'properties' => [
                    'sentiment' => [
                        'type' => 'string',
                        'enum' => ['positive', 'negative', 'neutral'],
                        'description' => '情感倾向',
                    ],
                    'confidence' => [
                        'type' => 'number',
                        'description' => '置信度 0-1',
                    ],
                    'keywords' => [
                        'type' => 'array',
                        'items' => ['type' => 'string'],
                        'description' => '关键情感词',
                    ],
                ],
                'required' => ['sentiment', 'confidence', 'keywords'],
                'additionalProperties' => false,
            ],
        ],
    ],
]);

$data = json_decode($result->asText(), true);
echo "情感：{$data['sentiment']}，置信度：{$data['confidence']}\n";
```

---

## 完整示例：智能简历解析系统

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

// 定义简历数据结构
class WorkExperience
{
    public function __construct(
        public readonly string $company,
        public readonly string $position,
        public readonly string $startDate,
        public readonly ?string $endDate,
        public readonly string $description,
    ) {
    }
}

class Resume
{
    /**
     * @param string           $name        姓名
     * @param string           $email       邮箱
     * @param ?string          $phone       电话
     * @param int              $yearsOfExp  工作年限
     * @param string[]         $skills      技能列表
     * @param WorkExperience[] $experience  工作经历
     * @param string           $education   最高学历
     */
    public function __construct(
        public readonly string $name,
        public readonly string $email,
        public readonly ?string $phone,
        public readonly int $yearsOfExp,
        public readonly array $skills,
        public readonly array $experience,
        public readonly string $education,
    ) {
    }
}

// 初始化
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$platform = PlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);

// 模拟收到的简历文本
$resumeText = <<<TEXT
个人简历

姓名：王晓明
邮箱：wangxm@example.com
电话：139-1234-5678

教育背景：
北京大学 计算机科学与技术 硕士 2018-2020

工作经历：

1. 字节跳动 - 高级后端工程师（2022.03 - 至今）
   负责推荐系统后端架构设计与优化，日处理请求量 5 亿+。
   主导了系统从单体到微服务的架构迁移。

2. 阿里巴巴 - 后端工程师（2020.07 - 2022.02）
   参与淘宝商品搜索引擎开发。负责搜索排序算法的工程化实现。

技能：PHP, Go, Python, MySQL, Redis, Elasticsearch, Kubernetes, Docker
TEXT;

$messages = new MessageBag(
    Message::forSystem('从简历文本中提取结构化信息。今天是 2025-01-15。'),
    Message::ofUser($resumeText),
);

$result = $platform->invoke('gpt-4o-mini', $messages, [
    'response_format' => Resume::class,
]);

$resume = $result->asObject();

echo "=== 简历解析结果 ===\n\n";
echo "姓名：{$resume->name}\n";
echo "邮箱：{$resume->email}\n";
echo "电话：{$resume->phone}\n";
echo "工作年限：{$resume->yearsOfExp} 年\n";
echo "学历：{$resume->education}\n";
echo "技能：" . implode(', ', $resume->skills) . "\n\n";

echo "工作经历：\n";
foreach ($resume->experience as $exp) {
    $endDate = $exp->endDate ?? '至今';
    echo "  [{$exp->startDate} ~ {$endDate}] {$exp->company} - {$exp->position}\n";
    echo "    {$exp->description}\n\n";
}
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `PlatformSubscriber` | 结构化输出的事件订阅器，必须注册到 EventDispatcher |
| `response_format` | 指定输出格式，可以是 PHP 类名或 JSON Schema |
| `$result->asObject()` | 将 AI 输出反序列化为 PHP 对象 |
| PHPDoc 注释 | AI 通过属性注释理解每个字段的含义 |
| 嵌套结构 | 支持对象嵌套和数组，自动递归映射 |

## 下一步

除了文本，AI 还可以处理图片、音频、PDF 等多模态内容。请看 [07-multimodal-content-understanding.md](./07-multimodal-content-understanding.md)。
