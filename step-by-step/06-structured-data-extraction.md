# 结构化数据提取

## 业务场景

你在做一个智能表单系统。用户用自然语言描述需求，AI 自动将描述转为结构化的数据对象。比如用户说"我要订明天下午三点从北京飞上海的机票，一个成人"，系统需要提取出出发城市、目的城市、时间、人数等结构化字段。

**典型应用：** 智能表单填充、数据录入自动化、简历解析、订单信息提取、内容分类打标

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台，使用结构化输出能力 |
| **StructuredOutput** | 自动将 PHP 类转为 JSON Schema，让 AI 返回可直接映射为 PHP 对象的数据 |
| **ResponseFormatFactory** | 从 PHP 类定义自动生成 JSON Schema |
| **PropertyDescriber** | 描述器链，用于分析 PHP 类属性并生成精确的 schema 描述 |

## 前置准备

```bash
composer require symfony/ai-platform symfony/ai-platform-deepseek
composer require symfony/event-dispatcher  # 结构化输出需要事件分发器
```

> [!TIP]
> 本教程使用 **DeepSeek** 作为主要平台（`deepseek-chat` 模型），性价比极高，非常适合结构化数据提取任务。如需切换其他平台，只需替换 `PlatformFactory` 和对应的 API Key 即可。

---

## 整体流程

```
┌──────────────────┐
│   用户自然语言     │
│   输入文本         │
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────────────┐
│  PlatformSubscriber::processInput()  │
│  ┌────────────────────────────────┐  │
│  │  ResponseFormatFactory         │  │
│  │  ┌──────────────────────────┐  │  │
│  │  │ PropertyDescriber 链      │  │  │
│  │  │ • TypeInfoDescriber      │  │  │
│  │  │ • PropertyInfoDescriber  │  │  │
│  │  │ • ValidatorConstraints   │  │  │
│  │  │   Describer              │  │  │
│  │  └──────────────────────────┘  │  │
│  │  PHP 类 ──→ JSON Schema       │  │
│  └────────────────────────────────┘  │
└────────┬─────────────────────────────┘
         │ 携带 JSON Schema 的请求
         ▼
┌──────────────────┐
│   AI 平台         │
│  (DeepSeek 等)    │
│  返回符合 Schema  │
│  的 JSON 数据     │
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────────────┐
│  PlatformSubscriber::processResult() │
│  JSON ──→ PHP 对象（反序列化）        │
└────────┬─────────────────────────────┘
         │
         ▼
┌──────────────────┐
│  业务层验证       │
│  (Symfony         │
│   Validator)      │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  结构化 PHP 对象  │
│  可直接使用       │
└──────────────────┘
```

---

## Step 1：理解结构化输出

普通调用 AI 返回自由文本字符串。结构化输出让 AI 返回符合预定义 schema 的 JSON，自动映射为 PHP 对象。

```
普通输出：  "北京到上海，明天下午三点，1个成人"（自由文本，需要额外解析）
结构化输出：FlightBooking { from: "北京", to: "上海", date: "2025-01-16", time: "15:00", passengers: 1 }
```

**背后的机制：**

1. `PlatformSubscriber` 监听平台调用事件
2. 当检测到 `response_format` 选项时，`ResponseFormatFactory` 将你的 PHP 类转为 JSON Schema
3. **PropertyDescriber 链**（`TypeInfoDescriber`、`PropertyInfoDescriber` 等）逐个属性分析类型、PHPDoc 注释、验证约束，生成精确的 schema
4. AI 平台收到 schema 后，严格按格式返回 JSON
5. 响应被自动反序列化为 PHP 对象

> [!NOTE]
> 结构化输出依赖 `EventDispatcher` + `PlatformSubscriber` 模式。`PlatformSubscriber` 会在请求发送前注入 JSON Schema，并在响应返回后执行反序列化。这是整个流程的核心枢纽。

---

## Step 2：定义输出结构（PHP 类）

创建一个 PHP 类来表示你期望的输出结构。**PHPDoc 注释非常关键**——`PropertyDescriber` 系统会读取它们来生成 schema 中的 `description` 字段，帮助 AI 理解每个属性的含义。

```php
<?php

namespace App\Dto;

/**
 * 机票预订信息。
 */
final class FlightBooking
{
    /**
     * @param string   $departureCity   出发城市（中文城市名）
     * @param string   $arrivalCity     到达城市（中文城市名）
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

> [!TIP]
> **PHPDoc 就是你的 Prompt 工程。** `ResponseFormatFactory` 通过 `PropertyDescriber` 链把 `@param` 中的描述文字原样传入 JSON Schema 的 `description` 字段。写得越清晰，AI 提取的结果越准确。比如写 `出发日期（YYYY-MM-DD 格式）` 比单纯写 `日期` 效果好得多。

---

## Step 3：使用结构化输出调用 AI

```php
<?php

require 'vendor/autoload.php';

use App\Dto\FlightBooking;
use Symfony\AI\Platform\Bridge\DeepSeek\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

// 1. 结构化输出需要注册 PlatformSubscriber
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

$platform = PlatformFactory::create(
    $_ENV['DEEPSEEK_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);

// 2. 用户的自然语言输入
$userInput = '帮我订一张后天下午两点半从上海飞广州的机票，商务舱，2个大人，需要靠窗座位。';

$messages = new MessageBag(
    Message::forSystem('你是一个机票预订助手。从用户的描述中提取预订信息。今天是 2025-01-15。'),
    Message::ofUser($userInput),
);

// 3. 指定 response_format 为 PHP 类，触发结构化输出
$result = $platform->invoke('deepseek-chat', $messages, [
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

> [!TIP]
> **使用 OpenAI 替代？** 只需更换 `PlatformFactory` 和 API Key：
> ```php
> use Symfony\AI\Platform\Bridge\OpenAI\PlatformFactory;
>
> $platform = PlatformFactory::create(
>     $_ENV['OPENAI_API_KEY'],
>     HttpClient::create(),
>     eventDispatcher: $dispatcher,
> );
>
> $result = $platform->invoke('gpt-4o-mini', $messages, [
>     'response_format' => FlightBooking::class,
> ]);
> ```

> [!WARNING]
> 结构化输出**不支持流式（streaming）模式**。`PlatformSubscriber` 会在检测到同时使用 `response_format` 和流式输出时抛出异常。如果需要流式展示，考虑先获取完整结构化结果，再渐进式地渲染到界面。

---

## Step 4：复杂嵌套结构

真实业务通常需要嵌套的数据结构。`ResponseFormatFactory` 会递归遍历所有嵌套类型，通过 `ObjectDescriberInterface` 自动展开子对象，为每一层生成完整的 JSON Schema。

```php
<?php

namespace App\Dto;

/**
 * 客户联系方式。
 */
final class ContactInfo
{
    /**
     * @param string  $name  客户姓名
     * @param string  $email 邮箱地址
     * @param ?string $phone 联系电话（可选）
     */
    public function __construct(
        public readonly string $name,
        public readonly string $email,
        public readonly ?string $phone = null,
    ) {
    }
}

/**
 * 订单中的单个商品。
 */
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

/**
 * 完整的采购订单。
 */
final class Order
{
    /**
     * @param ContactInfo $customer     客户信息
     * @param OrderItem[] $items        订单商品列表
     * @param string      $deliveryAddr 配送地址（完整地址，含省市区）
     * @param string      $urgency      紧急程度（normal/urgent/express）
     * @param ?string     $notes        备注信息（可选）
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

> [!NOTE]
> **嵌套 DTO 设计原则：** 将可复用的子结构（如 `ContactInfo`）拆分为独立类。`ResponseFormatFactory` 通过 `PropertyDescriber` 链自动识别类型化的属性（如 `ContactInfo $customer`）和类型化数组（如 `OrderItem[] $items`），递归生成嵌套的 JSON Schema。确保数组属性使用 `@param Type[]` 注解，这是 `PropertyDescriber` 识别数组元素类型的关键。

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

$result = $platform->invoke('deepseek-chat', $messages, [
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

## Step 5：使用 Symfony Validator 验证提取结果

AI 提取的数据不一定总是正确的——日期格式可能异常、数值可能越界、必填字段可能缺失。通过 Symfony Validator 的约束注解，你可以在两个层面获得保障：

1. **Schema 层面**：`ValidatorConstraintsDescriber` 会把验证约束（如 `@Assert\Length`、`@Assert\Choice`）自动转换为 JSON Schema 中的 `minLength`、`enum` 等约束，引导 AI 生成合规的数据
2. **业务层面**：对 AI 返回的对象执行验证，拦截不符合业务规则的数据

```bash
composer require symfony/validator
```

```php
<?php

namespace App\Dto;

use Symfony\Component\Validator\Constraints as Assert;

/**
 * 经过验证的机票预订信息。
 */
final class ValidatedFlightBooking
{
    /**
     * @param string   $departureCity   出发城市（中文城市名）
     * @param string   $arrivalCity     到达城市（中文城市名）
     * @param string   $date            出发日期（YYYY-MM-DD 格式）
     * @param string   $time            出发时间（HH:MM 格式）
     * @param int      $passengers      乘客人数
     * @param string   $cabinClass      舱位等级
     * @param string[] $specialRequests 特殊需求列表
     */
    public function __construct(
        #[Assert\NotBlank]
        #[Assert\Length(min: 2, max: 20)]
        public readonly string $departureCity,

        #[Assert\NotBlank]
        #[Assert\Length(min: 2, max: 20)]
        public readonly string $arrivalCity,

        #[Assert\NotBlank]
        #[Assert\Regex(pattern: '/^\d{4}-\d{2}-\d{2}$/')]
        public readonly string $date,

        #[Assert\NotBlank]
        #[Assert\Regex(pattern: '/^\d{2}:\d{2}$/')]
        public readonly string $time,

        #[Assert\Range(min: 1, max: 9)]
        public readonly int $passengers,

        #[Assert\Choice(choices: ['economy', 'business', 'first'])]
        public readonly string $cabinClass = 'economy',

        public readonly array $specialRequests = [],
    ) {
    }
}
```

> [!NOTE]
> **双重保障机制：** 当安装了 `symfony/validator` 后，`ValidatorConstraintsDescriber`（`PropertyDescriber` 链中的一环）会自动读取 `#[Assert\*]` 注解，将它们映射到 JSON Schema 的对应约束中。例如 `#[Assert\Choice]` 变成 `enum`，`#[Assert\Length(min: 2)]` 变成 `minLength: 2`。这意味着 AI 在生成响应时就已经受到约束引导。

### 验证 AI 返回的数据：

```php
use Symfony\Component\Validator\Validation;

$result = $platform->invoke('deepseek-chat', $messages, [
    'response_format' => ValidatedFlightBooking::class,
]);

$booking = $result->asObject();

// 使用 Symfony Validator 进行业务层验证
$validator = Validation::createValidatorBuilder()
    ->enableAttributeMapping()
    ->getValidator();

$violations = $validator->validate($booking);

if (count($violations) > 0) {
    echo "AI 提取的数据存在问题：\n";
    foreach ($violations as $violation) {
        echo "  - {$violation->getPropertyPath()}: {$violation->getMessage()}\n";
    }
    // 可以根据业务需求：重试、使用默认值、或提示用户确认
} else {
    echo "数据验证通过，可以安全使用。\n";
}
```

> [!TIP]
> **验证策略建议：** 对于关键业务数据（如金额、日期），始终在拿到 AI 结果后执行显式验证。不要完全依赖 JSON Schema 约束——AI 可能在边界情况下返回格式正确但语义错误的数据（如把"后天"解析为过去的日期）。

---

## Step 6：错误处理——当 AI 返回异常数据

即使使用了结构化输出，AI 仍可能返回无法正确反序列化的数据。常见原因包括：模型不完全遵守 schema、网络中断导致 JSON 截断、字段类型不匹配等。

```php
use Symfony\AI\Platform\Exception\PlatformException;

try {
    $result = $platform->invoke('deepseek-chat', $messages, [
        'response_format' => FlightBooking::class,
    ]);

    $booking = $result->asObject();
} catch (PlatformException $e) {
    // 平台级错误：API 认证失败、请求超时、模型不可用等
    echo "平台调用失败：{$e->getMessage()}\n";
    // 建议：记录日志，触发告警，使用降级策略
} catch (\Throwable $e) {
    // 反序列化失败：JSON 格式异常、字段类型不匹配等
    echo "数据解析失败：{$e->getMessage()}\n";
    // 建议：可尝试获取原始文本进行手动解析
    $rawText = $result->asText();
    echo "原始返回：{$rawText}\n";
}
```

> [!WARNING]
> **生产环境必须做错误处理。** AI 返回的结构化数据不能等同于可信的用户输入。建议建立三道防线：
> 1. **try-catch** 捕获反序列化异常
> 2. **Symfony Validator** 验证业务规则
> 3. **重试机制** 在首次失败时自动重新请求（尤其对复杂嵌套结构）

### 带重试的健壮提取模式：

```php
function extractWithRetry(
    object $platform,
    string $model,
    MessageBag $messages,
    string $dtoClass,
    int $maxRetries = 2,
): object {
    $lastException = null;

    for ($attempt = 0; $attempt <= $maxRetries; ++$attempt) {
        try {
            $result = $platform->invoke($model, $messages, [
                'response_format' => $dtoClass,
            ]);

            return $result->asObject();
        } catch (\Throwable $e) {
            $lastException = $e;
            // 可在此添加日志记录
        }
    }

    throw $lastException;
}

$booking = extractWithRetry($platform, 'deepseek-chat', $messages, FlightBooking::class);
```

---

## Step 7：使用 JSON Schema（不用 PHP 类）

对于简单的一次性提取任务，你也可以直接传入 JSON Schema 而无需定义 PHP 类：

```php
$result = $platform->invoke('deepseek-chat', $messages, [
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

> [!TIP]
> **何时用 PHP 类 vs JSON Schema？** 推荐优先使用 PHP 类——`ResponseFormatFactory` + `PropertyDescriber` 链会自动处理类型映射、PHPDoc 描述提取、Validator 约束转换等工作，大幅减少手写 schema 的出错概率。直接传 JSON Schema 适合快速原型验证或动态 schema 场景。

---

## 完整示例：智能简历解析系统

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\DeepSeek\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;
use Symfony\Component\Validator\Constraints as Assert;
use Symfony\Component\Validator\Validation;

/**
 * 单段工作经历。
 */
final class WorkExperience
{
    /**
     * @param string  $company     公司名称
     * @param string  $position    职位名称
     * @param string  $startDate   入职日期（YYYY-MM 或 YYYY.MM 格式）
     * @param ?string $endDate     离职日期（在职则为 null）
     * @param string  $description 工作内容概述
     */
    public function __construct(
        public readonly string $company,
        public readonly string $position,
        public readonly string $startDate,
        public readonly ?string $endDate,
        public readonly string $description,
    ) {
    }
}

/**
 * 结构化简历数据。
 */
final class Resume
{
    /**
     * @param string           $name        姓名
     * @param string           $email       邮箱地址
     * @param ?string          $phone       联系电话
     * @param int              $yearsOfExp  工作年限（整数）
     * @param string[]         $skills      技术技能列表
     * @param WorkExperience[] $experience  工作经历（按时间倒序）
     * @param string           $education   最高学历（如：北京大学 计算机硕士）
     */
    public function __construct(
        #[Assert\NotBlank]
        public readonly string $name,

        #[Assert\NotBlank]
        #[Assert\Email]
        public readonly string $email,

        public readonly ?string $phone,

        #[Assert\Range(min: 0, max: 50)]
        public readonly int $yearsOfExp,

        public readonly array $skills,
        public readonly array $experience,
        public readonly string $education,
    ) {
    }
}

// 初始化平台
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$platform = PlatformFactory::create(
    $_ENV['DEEPSEEK_API_KEY'],
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

try {
    $result = $platform->invoke('deepseek-chat', $messages, [
        'response_format' => Resume::class,
    ]);

    $resume = $result->asObject();

    // 验证提取结果
    $validator = Validation::createValidatorBuilder()
        ->enableAttributeMapping()
        ->getValidator();
    $violations = $validator->validate($resume);

    if (count($violations) > 0) {
        echo "⚠ 提取结果存在问题：\n";
        foreach ($violations as $violation) {
            echo "  - {$violation->getPropertyPath()}: {$violation->getMessage()}\n";
        }
    }

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
} catch (\Throwable $e) {
    echo "简历解析失败：{$e->getMessage()}\n";
}
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `PlatformSubscriber` | 结构化输出的事件订阅器，必须注册到 EventDispatcher |
| `ResponseFormatFactory` | 从 PHP 类自动生成 JSON Schema，是 `PlatformSubscriber` 内部调用的核心组件 |
| `PropertyDescriber` 链 | 分析 PHP 属性类型、PHPDoc、Validator 约束来生成精确 schema |
| `ValidatorConstraintsDescriber` | PropertyDescriber 链的一环，自动把 `#[Assert\*]` 转为 JSON Schema 约束 |
| `response_format` | 指定输出格式，可以是 PHP 类名或 JSON Schema 数组 |
| `$result->asObject()` | 将 AI 输出反序列化为 PHP 对象 |
| PHPDoc `@param` 注释 | 被 PropertyDescriber 读取，生成 schema 中的 `description` 字段 |
| 嵌套 DTO | 支持对象嵌套和类型化数组（`Type[]`），自动递归生成 schema |

> [!TIP]
> **成本优化建议：** 结构化数据提取任务通常不需要最强大的推理模型。`deepseek-chat` 在提取任务上的表现优异，且成本远低于大型推理模型。对于简单的字段提取（如姓名、日期、金额），优先选择轻量级模型可以大幅降低 API 开销。只在复杂的语义理解场景（如需要推理或跨段落关联）时才考虑使用 `deepseek-reasoner` 等推理模型。

## 下一步

除了文本，AI 还可以处理图片、音频、PDF 等多模态内容。请看 [07-multimodal-content-understanding.md](./07-multimodal-content-understanding.md)。
