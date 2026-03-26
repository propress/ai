# 第 11 章：实战 —— 结构化数据提取

## 学习目标

通过本章实战，你将掌握：
- 使用 StructuredOutput 从非结构化文本提取结构化数据
- 定义 DTO 并利用 `#[With]` 属性引导 AI 精准提取
- 处理嵌套 DTO 和对象数组
- 配合 Symfony Validator 验证提取结果
- 实现批量提取模式

## 前置知识

<!-- 建议在正式学习前回顾以下章节 -->
- Platform 组件基础用法
- JSON Schema 与 StructuredOutput 概念
- Symfony Serializer 基础

## 业务场景描述

从非结构化文本中提取结构化信息——例如从客户反馈中提取情感、类别、关键词，或从商品描述中提取属性。AI 返回的不是自由文本，而是符合预定义 Schema 的 PHP 对象。

**典型应用**：表单解析、情感分析、实体识别、数据录入自动化、简历解析。

## 架构概述

```php
结构化输出的内部流程
══════════════════

用户代码                              Platform 内部
────────                              ────────────

$options = [                          1. 检测到 'response_format' => ProductInfo::class
    'response_format' => ProductInfo::class    2. 使用 JsonSchemaFactory 生成 JSON Schema
];                                    3. 将 Schema 注入 AI 请求（response_format）
                                      4. AI 返回符合 Schema 的 JSON
$response = $platform->invoke(        5. ResultEvent 订阅器拦截结果
    $messages, $model, $options       6. 使用 Symfony Serializer 反序列化为 PHP 对象
);                                    7. 通过 $response->asObject() 获取对象

/** @var ProductInfo $dto */
$dto = $response->asObject();

                  JSON Schema（自动生成）
                  ══════════════════════
                  {
                    "type": "object",
                    "properties": {
                      "name": {
                        "type": "string",
                        "description": "商品名称"
                      },
                      "price": {
                        "type": "number",
                        "description": "价格，单位人民币"
                      },
                      "sentiment": {
                        "type": "string",
                        "enum": ["positive", "neutral", "negative"]
                      }
                    },
                    "required": ["name", "price", "sentiment"]
                  }
```

## 环境准备

```bash
composer require symfony/ai-platform symfony/ai-open-ai-platform
```

## 核心实现

### 定义数据结构（DTO）

```php
<?php

namespace App\Dto;

use Symfony\AI\Platform\Contract\JsonSchema\Attribute\With;

class ProductInfo
{
    /**
     * @param string $name 商品名称
     * @param float $price 价格（人民币）
     * @param string $category 商品分类
     * @param string[] $features 核心特性列表
     * @param Sentiment $sentiment 用户情感倾向
     */
    public function __construct(
        #[With(description: '商品名称')]
        public readonly string $name,
        #[With(description: '价格，单位人民币')]
        public readonly float $price,
        #[With(description: '商品分类，如电子产品、服装、食品等')]
        public readonly string $category,
        #[With(description: '商品的核心特性列表')]
        public readonly array $features,
        #[With(description: '描述中的情感倾向')]
        public readonly Sentiment $sentiment,
    ) {}
}

enum Sentiment: string
{
    case POSITIVE = 'positive';
    case NEUTRAL = 'neutral';
    case NEGATIVE = 'negative';
}
```

> **知识扩展：`#[With]` 属性的重要性**
>
> `#[With(description: '...')]` 属性的 `description` 参数**直接影响提取质量**。AI 会参考这些描述来决定如何填充每个字段。好的描述应该包含：
> - 字段的含义和用途
> - 数据格式要求（如「单位人民币」「格式为 YYYY-MM-DD」）
> - 可接受值的范围或示例
>
> ```php
> // 差的描述——AI 不知道期望什么格式
> #[With(description: '日期')]
> public readonly string $date,
>
> // 好的描述——AI 能准确提取
> #[With(description: '事件发生日期，格式为 YYYY-MM-DD，如 2024-01-15')]
> public readonly string $date,
> ```

### 使用 StructuredOutput

```php
<?php

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Bridge\OpenAi\Gpt;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);

$messages = new MessageBag(
    Message::forSystem('从用户输入中提取商品信息。如果信息不完整，尽量推断。'),
    Message::ofUser(
        '这款华为 Mate 60 Pro 手机真不错，5999 元的价格很合理，'
        .'拍照效果特别好，电池续航也很长，就是重量稍微有点重。'
    ),
);

$response = $platform->invoke(
    'gpt-4o',
    $messages,
    ['response_format' => ProductInfo::class],  // 指定输出类型
);

/** @var ProductInfo $product */
$product = $response->asObject();

echo $product->name;       // "华为 Mate 60 Pro"
echo $product->price;      // 5999.0
echo $product->category;   // "电子产品"
echo $product->sentiment;  // Sentiment::POSITIVE
print_r($product->features);
// ["拍照效果好", "电池续航长", "价格合理"]
```

### 嵌套 DTO 和对象数组

StructuredOutput 完整支持嵌套对象和对象数组：

```php
class OrderExtraction
{
    public function __construct(
        #[With(description: '订单编号，格式为 ORD-XXXXXXXX')]
        public readonly string $orderNumber,
        #[With(description: '客户信息')]
        public readonly CustomerInfo $customer,
        /** @var OrderItem[] */
        #[With(description: '订单商品列表')]
        public readonly array $items,
        #[With(description: '订单总金额，单位人民币')]
        public readonly float $totalAmount,
    ) {}
}

class CustomerInfo
{
    public function __construct(
        #[With(description: '客户姓名')]
        public readonly string $name,
        #[With(description: '联系电话')]
        public readonly string $phone,
        #[With(description: '邮箱地址，可选')]
        public readonly ?string $email = null,
    ) {}
}

class OrderItem
{
    public function __construct(
        #[With(description: '商品名称')]
        public readonly string $productName,
        #[With(description: '购买数量')]
        public readonly int $quantity,
        #[With(description: '单价，单位人民币')]
        public readonly float $unitPrice,
    ) {}
}
```

### 配合 Symfony Validator

结构化输出解决了格式问题，但 AI 提取的**内容**可能不准确。配合 Validator 可以进一步保证数据质量：

```php
use Symfony\Component\Validator\Constraints as Assert;

class ProductInfo
{
    public function __construct(
        #[Assert\NotBlank(message: '商品名称不能为空')]
        #[With(description: '商品名称')]
        public readonly string $name,

        #[Assert\Positive(message: '价格必须为正数')]
        #[Assert\LessThan(value: 1000000, message: '价格超出合理范围')]
        #[With(description: '价格，单位人民币')]
        public readonly float $price,

        #[Assert\Choice(choices: ['电子产品', '服装', '食品', '家居', '其他'])]
        #[With(description: '商品分类，可选值：电子产品、服装、食品、家居、其他')]
        public readonly string $category,
        // ...
    ) {}
}

// 提取后验证
$product = $response->asObject();
$violations = $validator->validate($product);

if (count($violations) > 0) {
    // AI 提取的数据不符合约束
    foreach ($violations as $violation) {
        $logger->warning('结构化输出验证失败', [
            'field' => $violation->getPropertyPath(),
            'message' => $violation->getMessage(),
            'value' => $violation->getInvalidValue(),
        ]);
    }
    // 可以选择重试（换个 temperature）或人工复核
}
```

### 批量提取

处理大量文本时的批量提取模式：

```php
// 批量情感分析
class FeedbackAnalysis
{
    public function __construct(
        #[With(description: '客户反馈的情感倾向')]
        public readonly Sentiment $sentiment,
        #[With(description: '反馈涉及的主题标签列表')]
        public readonly array $topics,
        #[With(description: '紧急程度 1-5，5 最紧急')]
        public readonly int $urgency,
        #[With(description: '关键诉求的一句话摘要')]
        public readonly string $summary,
    ) {}
}

$feedbacks = ['反馈1...', '反馈2...', '反馈3...'];
$results = [];

foreach ($feedbacks as $feedback) {
    $messages = new MessageBag(
        Message::forSystem('分析以下客户反馈。'),
        Message::ofUser($feedback),
    );

    $response = $platform->invoke($model, $messages, [
        'response_format' => FeedbackAnalysis::class,
        'temperature' => 0.1,  // 低 temperature 确保一致性
    ]);

    $results[] = $response->asObject();
}
```

## 运行与验证

<!-- 运行示例并验证输出的步骤 -->

## 错误处理

### 通用错误处理模式

每个场景在生产环境中都应该包含完善的错误处理：

```php
use Symfony\AI\Platform\Exception\AuthenticationException;
use Symfony\AI\Platform\Exception\RateLimitExceededException;
use Symfony\AI\Platform\Exception\ContentFilterException;
use Symfony\AI\Platform\Exception\ExceedContextSizeException;

function safeInvoke(
    PlatformInterface $platform,
    MessageBag $messages,
    string $model,
    array $options = [],
): string {
    try {
        $response = $platform->invoke($model, $messages, $options);
        return $response->asText();
    } catch (AuthenticationException) {
        throw new \RuntimeException('AI 服务认证失败，请检查 API Key 配置');
    } catch (RateLimitExceededException $e) {
        // 可以实现退避重试
        sleep($e->getRetryAfter() ?? 5);
        return safeInvoke($platform, $messages, $model, $options);
    } catch (ExceedContextSizeException) {
        // 输入过长，截断消息历史
        $truncated = new MessageBag(...array_slice($messages->getMessages(), -5));
        return safeInvoke($platform, $truncated, $model, $options);
    } catch (ContentFilterException) {
        return '您的输入触发了安全过滤，请修改后重试。';
    } catch (\Throwable $e) {
        $this->logger->error('AI 调用异常', ['exception' => $e]);
        return '抱歉，AI 服务暂时不可用。';
    }
}
```

### Token 成本优化

| 优化策略 | 做法 | 效果 |
|---------|------|------|
| **精简系统提示** | 200 字而非 1000 字 | 减少 30-50% 输入 Token |
| **限制上下文** | 只保留最近 N 轮对话 | 避免上下文爆炸 |
| **选择合适模型** | 简单任务用 mini 模型 | 降低 80-90% 成本 |
| **缓存重复请求** | CachePlatform | 重复请求零成本 |
| **设置 max_tokens** | 限制输出长度 | 控制输出成本 |

## 生产环境注意事项

<!-- 部署到生产环境时的配置和优化建议 -->

## 扩展方向

<!-- 基于本场景的进一步扩展思路 -->

## 完整源代码

<!-- 完整可运行的源代码汇总 -->

## 下一步

下一章我们将学习 [第 12 章：实战 —— 多模态内容理解](12-scenario-multimodal.md)，让 AI 理解图片、PDF、音频等非文本内容。
