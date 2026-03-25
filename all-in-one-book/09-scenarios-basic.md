# 第 9 章：实战场景（基础篇）

## 🎯 本章学习目标

通过 6 个基础实战场景，将 Platform 和 Chat 组件的核心概念付诸实践：从简单的问答机器人到多模态内容理解，从结构化数据提取到实时流式对话，掌握 AI 应用开发的常见模式。每个场景都包含架构概述、完整实现、错误处理和生产优化建议。

---

## 1. 回顾

在前八章中，我们系统学习了 Symfony AI 的所有核心组件。本章开始进入**实战篇**，通过真实业务场景将知识融会贯通。

**本章侧重点**：Platform + Chat 组件的基础用法，不涉及 Agent 工具调用和 Store 向量检索。

**场景总览**：

| 场景 | 核心组件 | 难度 | 典型应用 |
|------|---------|------|---------|
| 基础问答聊天机器人 | Platform | ⭐ | 官网客服、FAQ |
| 多轮对话与会话持久化 | Platform + Chat | ⭐⭐ | 在线客服 |
| 结构化数据提取 | Platform + StructuredOutput | ⭐⭐ | 表单解析、实体识别 |
| 多模态内容理解 | Platform + Content Types | ⭐⭐ | 图片/文档/音频分析 |
| 多语言翻译助手 | Platform + Prompt Engineering | ⭐⭐ | 文档翻译、本地化 |
| 实时流式对话 | Platform + SSE | ⭐⭐⭐ | ChatGPT 式界面 |

---

## 2. 场景一：基础问答聊天机器人

### 2.1 业务场景

在产品官网嵌入一个 AI 聊天窗口，用户提问，AI 根据预设的系统提示词给出回答。

**典型应用**：官网客服机器人、FAQ 助手、产品介绍机器人、内部知识库问答。

### 2.2 架构概述

```
┌──────────┐     ┌──────────────┐     ┌──────────────────┐     ┌──────────────┐
│  用户输入  │ ──▶│  构建 MessageBag│ ──▶│ Platform::invoke() │ ──▶│ Contract 序列化│
│ (文本问题) │     │ (System+User) │     │    (分发事件)      │     │  (转换格式)    │
└──────────┘     └──────────────┘     └──────────────────┘     └──────┬───────┘
                                                                       │
                                                                       ▼
┌──────────┐     ┌──────────────┐     ┌──────────────────┐     ┌──────────────┐
│  显示回复  │ ◀──│  asText() 或   │ ◀──│  DeferredResult    │ ◀──│  AI 平台 API  │
│ (前端渲染) │     │  asStream()   │     │   (延迟求值)       │     │  (推理生成)    │
└──────────┘     └──────────────┘     └──────────────────┘     └──────────────┘
```

> 📝 **知识扩展**：流程中的 `Contract` 是一个常被忽略但至关重要的组件。它基于 Symfony Serializer，负责将 `Message`、`ToolCall` 等对象序列化为各平台特定的 JSON 格式。每个 Bridge 都有自己的 Contract 实现（如 `AnthropicContract`、`OpenAiContract`），你无需直接操作它——`PlatformFactory` 会自动创建并注入。

### 2.3 所需依赖

```bash
composer require symfony/ai-platform symfony/ai-anthropic-platform
# 或使用 OpenAI：
# composer require symfony/ai-platform symfony/ai-open-ai-platform
```

### 2.4 核心实现

```php
<?php

use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 1. 创建平台实例
$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY']);

// 2. 构建消息——MessageBag 是消息的有序容器
$messages = new MessageBag(
    Message::forSystem('你是一个友好的产品助手。用简洁专业的中文回答用户问题。'),
    Message::ofUser('你们的产品有哪些定价方案？'),
);

// 3. 调用 AI——invoke() 返回 DeferredResult（延迟求值）
$response = $platform->invoke('claude-3-5-sonnet-latest', $messages);

// 4. 获取文本结果——此时才真正解析 HTTP 响应
echo $response->asText();
```

### 2.5 流式响应

对于 Web 应用，流式输出能大幅提升用户体验——首字节通常在 200ms 内返回，而非等待完整回复的 3-10 秒：

```php
$response = $platform->invoke($model, $messages);

// 流式输出——逐块获取文本
foreach ($response->asStream() as $chunk) {
    echo $chunk;
    flush();
}
```

> 📝 **知识扩展：流式响应的内部机制**
>
> `asStream()` 返回一个 PHP `Generator`，内部通过 HTTP 分块传输（chunked transfer）逐步接收 AI 平台的响应。每收到一个 token（或几个 token），Generator 就 `yield` 一次。这意味着：
> - Generator 只能迭代**一次**——如果需要重复使用，先 `asText()` 保存完整内容
> - 流式模式下 `getMetadata()` 可能不完整（Token 计数在流结束后才确定）
> - 流式模式与结构化输出（`output`）不兼容——结构化输出需要完整 JSON 后才能解析

### 2.6 模型选项

```php
$model = 'claude-3-5-sonnet-latest';

$response = $platform->invoke($model, $messages, [
    'temperature' => 0.7,      // 创造性（0.0-1.0）
    'max_tokens' => 1024,      // 最大输出 Token 数
]);
```

**temperature 参数指南**：

| temperature | 效果 | 适用场景 |
|-------------|------|---------|
| 0.0-0.2 | 输出高度确定、一致 | 事实问答、数据提取、代码生成 |
| 0.3-0.6 | 平衡准确性和多样性 | 通用对话、客服 |
| 0.7-1.0 | 输出多样、有创造性 | 内容创作、头脑风暴 |

### 2.7 切换平台

Symfony AI 的 Bridge 架构让平台切换只需改两行代码——业务逻辑完全不变：

```php
// 从 Anthropic 切换到 OpenAI：
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);
$response = $platform->invoke('gpt-4o', $messages);

// 从 OpenAI 切换到 Gemini：
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;

$platform = PlatformFactory::create($_ENV['GOOGLE_API_KEY']);
$response = $platform->invoke('gemini-2.5-flash', $messages);

// 从云端切换到本地 Ollama：
use Symfony\AI\Platform\Bridge\Ollama\PlatformFactory;

$platform = PlatformFactory::create();  // 默认 localhost:11434
$response = $platform->invoke('llama3.1', $messages);
```

### 2.8 错误处理

```php
use Symfony\AI\Platform\Exception\AuthenticationException;
use Symfony\AI\Platform\Exception\RateLimitExceededException;
use Symfony\AI\Platform\Exception\ContentFilterException;

try {
    $response = $platform->invoke($model, $messages);
    echo $response->asText();
} catch (AuthenticationException $e) {
    // API Key 无效或过期
    $logger->critical('AI 认证失败', ['error' => $e->getMessage()]);
    echo '系统配置错误，请联系管理员。';
} catch (RateLimitExceededException $e) {
    // 超出 API 调用限额
    $retryAfter = $e->retryAfter;  // 可选：建议等待秒数
    $logger->warning('AI 限速', ['retry_after' => $retryAfter]);
    echo '服务繁忙，请稍后重试。';
} catch (ContentFilterException $e) {
    // 触发内容安全策略
    echo '您的问题触发了安全过滤，请重新措辞。';
} catch (\Throwable $e) {
    $logger->error('AI 调用失败', ['exception' => $e]);
    echo '抱歉，AI 服务暂时不可用。';
}
```

> 📝 **知识扩展：Platform 异常体系**
>
> Symfony AI 定义了完整的异常层次，让你可以针对不同类型的错误做出精确响应：
>
> | 异常类 | 场景 | 处理建议 |
> |--------|------|---------|
> | `AuthenticationException` | API Key 无效/过期 | 告警 + 快速失败 |
> | `RateLimitExceededException` | 超出配额（可含 `retryAfter`） | 退避重试 |
> | `ContentFilterException` | 触发安全策略 | 通知用户修改输入 |
> | `ExceedContextSizeException` | 输入超过模型上下文窗口 | 截断消息历史 |
> | `BadRequestException` | 请求格式错误 | 检查参数 |
> | `MissingModelSupportException` | 模型不支持请求的功能 | 换用其他模型 |
> | `ModelNotFoundException` | 模型名称不存在 | 检查模型名 |

### 2.9 使用事件监控

```php
use Symfony\AI\Platform\Event\InvocationEvent;
use Symfony\AI\Platform\Event\ResultEvent;

// 在调用前拦截——可以修改模型、输入或选项
$dispatcher->addListener(InvocationEvent::class, function (InvocationEvent $event) {
    $this->logger->info('AI 调用开始', [
        'model' => $event->model,
    ]);
});

// 在结果返回后——记录 Token 使用量和耗时
$dispatcher->addListener(ResultEvent::class, function (ResultEvent $event) {
    $metadata = $event->deferredResult->getMetadata();
    $this->logger->info('AI 调用完成', [
        'model' => $event->model,
        'input_tokens' => $metadata->getInputTokens(),
        'output_tokens' => $metadata->getOutputTokens(),
    ]);
});
```

---

## 3. 场景二：多轮对话与会话持久化

### 3.1 业务场景

构建一个能记住上下文的客服聊天系统。用户可以进行多轮对话，系统记住之前的交流内容。传统聊天机器人每次请求都是独立的——Chat 组件解决了这个问题。

**典型应用**：在线客服、技术支持、咨询服务、智能导购。

### 3.2 架构概述

```
                                 Chat 组件的内部流程
                                 ═══════════════════

  initiate(systemPrompt)                    submit(conversationId, userMessage)
  ┌──────────────────┐                      ┌──────────────────────────────┐
  │ 1. 创建 Conversation                     │ 1. 从 Store 加载消息历史       │
  │    (UUID v7 作 ID)                       │ 2. 追加 UserMessage           │
  │ 2. 保存 SystemMessage                   │ 3. 调用 Platform::invoke()    │
  │    到 Store                              │ 4. 追加 AssistantMessage      │
  │ 3. 返回 Conversation                    │ 5. 保存到 Store               │
  └──────────────────┘                      │ 6. 返回 DeferredResult        │
                                            └──────────────────────────────┘

  Store 中的消息序列（每次 submit 后）：
  ┌──────────────────────────────────────────────────────────────────┐
  │ [System] 你是专业客服助手                                         │
  │ [User]   我想退货                                                │
  │ [AI]     好的，请提供订单号                                       │
  │ [User]   订单号是 ORD-20240115                                   │
  │ [AI]     已查到您的订单，退货申请已提交...                          │
  └──────────────────────────────────────────────────────────────────┘
```

### 3.3 所需依赖

```bash
composer require symfony/ai-platform symfony/ai-anthropic-platform \
    symfony/ai-chat symfony/ai-redis-message-store
```

### 3.4 完整实现

```php
<?php

use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\Bridge\Redis\MessageStore;
use Symfony\AI\Agent\Agent;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 1. 创建平台和消息存储
$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY']);
$store = new MessageStore(new \Redis(/* 连接配置 */));
$model = 'claude-3-5-sonnet-latest';

// 2. 创建 Agent 和 Chat 实例
//    注意：Chat 要求 store 同时实现 MessageStoreInterface & ManagedStoreInterface
$agent = new Agent($platform, $model);
$chat = new Chat($agent, $store);

// 3. 发起新对话——initiate() 返回 void
$chat->initiate(new MessageBag(
    Message::forSystem('你是一个专业的客服助手。回答简洁、礼貌。'),
));

// 4. 第一轮对话——submit() 接收 UserMessage，返回 AssistantMessage
$response = $chat->submit(Message::ofUser('我想退货'));
echo $response->getContent();
// "好的，请提供您的订单号，我来帮您查询退货资格。"

// 5. 第二轮对话——AI 自动获得之前的上下文
$response = $chat->submit(Message::ofUser('订单号是 ORD-20240115'));
echo $response->getContent();
// "已查到订单 ORD-20240115，购买于 2024-01-15，在 30 天退货期内..."

// 6. 第三轮——继续追问
$response = $chat->submit(Message::ofUser('运费谁承担？'));
echo $response->getContent();
// "根据我们的退货政策，质量问题免费退货；非质量问题需承担 ¥15 运费。"
```

> 📝 **知识扩展：Chat 的交叉类型约束**
>
> `Chat::__construct()` 对 store 参数使用了 PHP 交叉类型（intersection type）：
> ```php
> public function __construct(
>     private readonly AgentInterface $agent,
>     private readonly MessageStoreInterface&ManagedStoreInterface $store,
> )
> ```
> 这意味着 store 必须**同时**实现两个接口：`MessageStoreInterface`（读写消息）和 `ManagedStoreInterface`（管理会话生命周期）。所有内置的 Chat Bridge（Redis、Doctrine、MongoDB 等）都满足此要求。
> 同时注意 Chat 接收的是 `AgentInterface`，而非直接的 `PlatformInterface`——这使得 Chat 可以利用 Agent 的完整能力（工具调用、处理器管线等）。

### 3.5 消息存储后端完整对比

| 后端 | Composer 包 | 持久化 | 性能 | 搜索 | 适用场景 |
|------|------------|--------|------|------|---------|
| **InMemory** | 内置 | ❌ | ⚡ 极快 | ❌ | 单元测试 |
| **Redis** | `symfony/ai-redis-message-store` | ✅ | ⚡ 快 | ❌ | 高并发生产环境 |
| **Doctrine DBAL** | `symfony/ai-doctrine-message-store` | ✅ | 🔄 中 | SQL | 已有关系数据库的项目 |
| **MongoDB** | `symfony/ai-mongo-db-message-store` | ✅ | ⚡ 快 | ✅ | 文档型数据项目 |
| **Session** | 内置 | 📦 会话 | ⚡ 快 | ❌ | 简单 Web 应用 |
| **Cache** | 内置 | ⏰ TTL | ⚡ 快 | ❌ | 短期对话 |
| **Cloudflare** | `symfony/ai-cloudflare-message-store` | ✅ | 🌐 CDN | ❌ | 边缘部署 |
| **Meilisearch** | `symfony/ai-meilisearch-message-store` | ✅ | ⚡ 快 | ✅ | 需要搜索对话历史 |
| **SurrealDB** | `symfony/ai-surreal-db-message-store` | ✅ | ⚡ 快 | ✅ | SurrealDB 项目 |

### 3.6 在 Symfony 控制器中使用

```php
<?php

namespace App\Controller;

use Symfony\AI\Chat\ChatInterface;
use Symfony\AI\Platform\Message\Message;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Routing\Attribute\Route;

class CustomerServiceController
{
    public function __construct(
        private readonly ChatInterface $chat,
    ) {}

    #[Route('/api/chat/start', methods: ['POST'])]
    public function startConversation(): JsonResponse
    {
        $conversation = $this->chat->initiate(
            Message::forSystem('你是专业客服，回答简洁有礼貌。'),
        );

        return new JsonResponse([
            'conversation_id' => $conversation->getId()->toString(),
        ]);
    }

    #[Route('/api/chat/message', methods: ['POST'])]
    public function sendMessage(Request $request): JsonResponse
    {
        $conversationId = $request->getPayload()->getString('conversation_id');
        $message = $request->getPayload()->getString('message');

        try {
            $response = $this->chat->submit($conversationId, $message);

            return new JsonResponse([
                'reply' => $response->asText(),
                'metadata' => [
                    'input_tokens' => $response->getMetadata()->getInputTokens(),
                    'output_tokens' => $response->getMetadata()->getOutputTokens(),
                ],
            ]);
        } catch (\Throwable $e) {
            return new JsonResponse(
                ['error' => '回复生成失败，请重试'],
                500,
            );
        }
    }
}
```

### 3.7 上下文长度管理

多轮对话的消息历史会持续增长，最终超过模型的上下文窗口。你需要制定截断策略：

```php
// 策略 1：限制保留的消息轮数
// 在自定义 Chat 子类中或通过事件处理器实现
$maxRounds = 20;  // 保留最近 20 轮对话

// 策略 2：使用摘要压缩历史
// 当历史超过阈值时，让 AI 生成摘要替换旧消息
$summary = $platform->invoke(new MessageBag(
    Message::forSystem('请用一段话总结以下对话的关键信息。'),
    Message::ofUser($oldHistoryText),
), $model)->asText();

// 然后用摘要替换原始历史
```

---

## 4. 场景三：结构化数据提取

### 4.1 业务场景

从非结构化文本中提取结构化信息——例如从客户反馈中提取情感、类别、关键词，或从商品描述中提取属性。AI 返回的不是自由文本，而是符合预定义 Schema 的 PHP 对象。

**典型应用**：表单解析、情感分析、实体识别、数据录入自动化、简历解析。

### 4.2 架构概述

```
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

### 4.3 所需依赖

```bash
composer require symfony/ai-platform symfony/ai-open-ai-platform
```

### 4.4 定义数据结构（DTO）

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

> 📝 **知识扩展：`#[With]` 属性的重要性**
>
> `#[With(description: '...')]` 属性的 `description` 参数**直接影响提取质量**。AI 会参考这些描述来决定如何填充每个字段。好的描述应该包含：
> - 字段的含义和用途
> - 数据格式要求（如「单位人民币」「格式为 YYYY-MM-DD」）
> - 可接受值的范围或示例
>
> ```php
> // ❌ 差的描述——AI 不知道期望什么格式
> #[With(description: '日期')]
> public readonly string $date,
>
> // ✅ 好的描述——AI 能准确提取
> #[With(description: '事件发生日期，格式为 YYYY-MM-DD，如 2024-01-15')]
> public readonly string $date,
> ```

### 4.5 使用 StructuredOutput

```php
<?php

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Bridge\OpenAi\GPT;
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

### 4.6 嵌套 DTO 和对象数组

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

### 4.7 配合 Symfony Validator

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

### 4.8 批量提取

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

---

## 5. 场景四：多模态内容理解

### 5.1 业务场景

让 AI 理解图片、PDF、音频等非文本内容。例如：分析产品图片生成描述、解析合同关键条款、转录会议录音。

**典型应用**：商品图片自动标注、文档审查、音频转录、视频内容分析。

### 5.2 架构概述

```
多模态消息的构建方式
══════════════════

                 UserMessage 可以包含多个 Content 部分
                 ════════════════════════════════════

Message::ofUser(                    ┌──────────────────────┐
    Image::fromFile('photo.jpg'),   │  Content Part 1      │
    Audio::fromFile('audio.mp3'),   │  [Image: base64...]  │
    '请分析以上内容',                │                      │
)                                   │  Content Part 2      │
                                    │  [Audio: base64...]  │
                                    │                      │
                                    │  Content Part 3      │
                                    │  [Text: "请分析..."] │
                                    └──────────────────────┘

        Content 类型层次
        ════════════════

        Content (interface)
        ├── Text         —— 纯文本
        ├── Image        —— 图片（fromFile / fromUrl / fromBase64）
        ├── Audio        —— 音频文件
        ├── Document     —— 文档（PDF 等）
        └── Video        —— 视频文件
```

### 5.3 所需依赖

```bash
composer require symfony/ai-platform symfony/ai-open-ai-platform
# 视频理解需要 Gemini：
# composer require symfony/ai-gemini-platform
```

### 5.4 图片分析

```php
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\Content\Image;
use Symfony\AI\Platform\Message\MessageBag;

// 方式 1：从本地文件加载（自动 base64 编码）
$messages = new MessageBag(
    Message::ofUser(
        Image::fromFile('/path/to/product.jpg'),
        '请描述这张产品图片的内容，包括颜色、材质和适用场景。',
    ),
);

// 方式 2：从 URL 加载（平台直接请求 URL，无需本地下载）
$messages = new MessageBag(
    Message::ofUser(
        Image::fromUrl('https://example.com/photo.png'),
        '这张图片中有什么？',
    ),
);

// 方式 3：从 base64 数据加载
$messages = new MessageBag(
    Message::ofUser(
        Image::fromBase64($base64Data, 'image/jpeg'),
        '分析这张图片。',
    ),
);

$response = $platform->invoke('gpt-4o', $messages);
echo $response->asText();
```

### 5.5 多图片对比

```php
// 同时分析多张图片
$messages = new MessageBag(
    Message::ofUser(
        Image::fromFile('/path/to/product_v1.jpg'),
        Image::fromFile('/path/to/product_v2.jpg'),
        '请对比这两张产品图片的区别，列出设计变更点。',
    ),
);
```

### 5.6 图片 + 结构化输出

```php
class ProductImageAnalysis
{
    public function __construct(
        #[With(description: '图片中产品的主色调')]
        public readonly string $primaryColor,
        #[With(description: '产品材质判断')]
        public readonly string $material,
        #[With(description: '适用场景列表')]
        public readonly array $useCases,
        #[With(description: '图片质量评分 1-10')]
        public readonly int $imageQuality,
        #[With(description: '是否包含品牌 Logo')]
        public readonly bool $hasBrandLogo,
    ) {}
}

$response = $platform->invoke($model, $messages, [
    'response_format' => ProductImageAnalysis::class,
]);

$analysis = $response->asObject();
echo "主色调: {$analysis->primaryColor}";
echo "材质: {$analysis->material}";
```

### 5.7 PDF 文档处理

```php
use Symfony\AI\Platform\Message\Content\Document;

$messages = new MessageBag(
    Message::ofUser(
        Document::fromFile('/path/to/contract.pdf'),
        '请总结这份合同的主要条款和关键日期。特别注意：'
        .'1. 合同期限和终止条件'
        .'2. 付款条款和金额'
        .'3. 违约责任',
    ),
);

$response = $platform->invoke($model, $messages);
echo $response->asText();
```

### 5.8 音频处理

```php
use Symfony\AI\Platform\Message\Content\Audio;

// 音频转录 + 分析
$messages = new MessageBag(
    Message::ofUser(
        Audio::fromFile('/path/to/meeting.mp3'),
        '请完成以下任务：'
        .'1. 转录这段会议录音的完整内容'
        .'2. 提取关键决议事项'
        .'3. 标注每个发言人的主要观点',
    ),
);

$response = $platform->invoke($model, $messages);
echo $response->asText();
```

### 5.9 视频理解（Gemini）

```php
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Video;

$platform = PlatformFactory::create($_ENV['GOOGLE_API_KEY']);

$messages = new MessageBag(
    Message::ofUser(
        Video::fromFile('/path/to/demo.mp4'),
        '请分析这段视频的内容，列出关键时间点和对应的事件。',
    ),
);

$response = $platform->invoke('gemini-2.5-flash', $messages);
echo $response->asText();
```

### 5.10 平台多模态能力对比

| 内容类型 | OpenAI GPT-4o | Anthropic Claude | Google Gemini | Ollama（本地） |
|---------|:-------------:|:----------------:|:-------------:|:------------:|
| **文本** | ✅ | ✅ | ✅ | ✅ |
| **图片** | ✅ | ✅ | ✅ | ⚠️ 部分模型 |
| **PDF 文档** | ✅ | ✅ | ✅ | ❌ |
| **音频** | ✅ | ❌ | ✅ | ❌ |
| **视频** | ❌ | ❌ | ✅ | ❌ |

> ⚠️ **注意**：使用前请确认目标平台和模型支持你需要的模态。如果不支持，`Platform` 会抛出 `MissingModelSupportException`。

---

## 6. 场景五：多语言翻译助手

### 6.1 业务场景

构建一个支持多语言翻译的 AI 助手，能保持术语一致性，支持上下文感知翻译和多轮修正。

**典型应用**：技术文档翻译、产品本地化、多语言客服、学术论文翻译。

### 6.2 架构概述

翻译助手的核心是精心设计的 System Prompt——它定义了翻译规则、术语表和质量标准。这是 **Prompt Engineering** 的经典应用：

```
┌─────────────────────────────────────────┐
│          System Prompt（翻译指令）         │
│                                         │
│  角色定义：专业技术文档翻译专家              │
│  翻译规则：7 条核心规则                    │
│  术语表：Platform→平台, Agent→智能代理...  │
│  质量标准：信达雅                          │
│  禁止事项：不翻译代码、变量名               │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│          User Message（待翻译内容）        │
│                                         │
│  "The Agent component provides a        │
│   framework for building AI agents..."  │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│          AI 翻译结果                      │
│                                         │
│  "Agent 组件为构建 AI 智能代理提供了       │
│   一个完整的框架..."                      │
│  （术语一致、格式保留、代码块不翻译）       │
└─────────────────────────────────────────┘
```

### 6.3 核心实现

```php
<?php

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Bridge\OpenAi\GPT;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);

$systemPrompt = <<<PROMPT
你是一个专业的技术文档翻译专家，专注于将英文技术文档翻译为中文。

翻译规则：
1. 保持技术术语的准确性——参照术语表
2. 代码块（```）中的内容不翻译
3. 变量名、类名、方法名不翻译
4. 保持 Markdown 格式完整
5. 长句拆分为短句，增强可读性
6. 保留原文中的链接和引用
7. 不确定的翻译加注释标注

术语表（必须严格遵守）：
   - Platform → 平台
   - Agent → 智能代理
   - Store → 存储
   - Bridge → 桥接器
   - Toolbox → 工具箱
   - Processor → 处理器
   - MessageBag → 消息包
   - Embedding → 嵌入向量
PROMPT;

$messages = new MessageBag(
    Message::forSystem($systemPrompt),
    Message::ofUser(
        "请将以下英文翻译为中文：\n\n"
        ."The Agent component provides a framework for building AI agents "
        ."that can use tools to accomplish tasks. The Toolbox manages all "
        ."available tools and their execution. When the Agent receives a "
        ."user message, it sends it to the LLM along with the tool definitions."
    ),
);

$response = $platform->invoke('gpt-4o', $messages, [
    'temperature' => 0.3,  // 低 temperature 保证翻译一致性
]);

echo $response->asText();
```

### 6.4 结构化翻译结果

```php
class TranslationResult
{
    public function __construct(
        #[With(description: '翻译后的完整文本')]
        public readonly string $translation,
        #[With(description: '翻译质量自评分 1-10，10 为最高')]
        public readonly int $qualityScore,
        #[With(description: '翻译过程中的注意事项和不确定之处')]
        public readonly array $notes,
        #[With(description: '使用到的术语映射列表')]
        public readonly array $glossaryUsed,
    ) {}
}

$response = $platform->invoke($model, $messages, [
    'response_format' => TranslationResult::class,
    'temperature' => 0.3,
]);

$result = $response->asObject();
echo $result->translation;     // 翻译文本
echo $result->qualityScore;    // 9
print_r($result->notes);       // ["Agent 在此上下文中也可译为'代理']
print_r($result->glossaryUsed);// ["Agent→智能代理", "Toolbox→工具箱"]
```

### 6.5 多轮修正翻译

配合 Chat 组件实现带上下文的翻译修正：

```php
// 发起翻译对话
$conversation = $chat->initiate(
    Message::forSystem($systemPrompt),
);

// 第一轮：翻译
$chat->submit($conversation->getId(), '翻译以下内容：...');

// 第二轮：修正——AI 记得之前的翻译
$chat->submit($conversation->getId(), '把 "智能代理" 改为 "Agent"，保留英文原文');

// 第三轮：继续修正
$chat->submit($conversation->getId(), '第二段的语句太长了，拆成两句');
```

---

## 7. 场景六：实时流式对话

### 7.1 业务场景

构建一个 ChatGPT 式的实时流式对话 Web 应用，使用 Server-Sent Events（SSE）将 AI 回复逐字推送到前端。

**典型应用**：ChatGPT 式交互界面、实时客服、在线教育、代码辅助。

### 7.2 架构概述

```
完整的流式对话架构
══════════════════

  浏览器                  Symfony 控制器              AI 平台
  ──────                  ──────────────              ──────
    │                         │                         │
    │  POST /api/chat         │                         │
    │  {message: "你好"}      │                         │
    │ ───────────────────────→│                         │
    │                         │  invoke(model, messages) │
    │                         │────────────────────────→│
    │                         │                         │
    │                         │  HTTP chunked response   │
    │  SSE: data: {"text":"你"}│◀════════════════════════│ token: "你"
    │ ◀──────────────────────│                         │
    │                         │                         │
    │  SSE: data: {"text":"好"}│◀════════════════════════│ token: "好"
    │ ◀──────────────────────│                         │
    │                         │                         │
    │  SSE: data: {"text":"！"}│◀════════════════════════│ token: "！"
    │ ◀──────────────────────│                         │
    │                         │                         │
    │  SSE: data: [DONE]      │◀════════════════════════│ [end]
    │ ◀──────────────────────│                         │
    │                         │                         │
    │  close connection       │                         │
```

### 7.3 Symfony 控制器

```php
<?php

namespace App\Controller;

use Symfony\AI\Platform\PlatformInterface;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\StreamedResponse;
use Symfony\Component\Routing\Attribute\Route;

class StreamingChatController
{
    public function __construct(
        private readonly PlatformInterface $platform,
    ) {}

    #[Route('/api/chat/stream', methods: ['POST'])]
    public function stream(Request $request): StreamedResponse
    {
        $userMessage = $request->getPayload()->getString('message');

        $messages = new MessageBag(
            Message::forSystem('你是一个友好的中文助手。回答详细且有条理。'),
            Message::ofUser($userMessage),
        );

        return new StreamedResponse(
            function () use ($messages) {
                // 关键：在回调中设置 SSE 头
                header('Content-Type: text/event-stream');
                header('Cache-Control: no-cache');
                header('X-Accel-Buffering: no');  // 禁用 Nginx 缓冲

                $response = $this->platform->invoke('gpt-4o', $messages);

                foreach ($response->asStream() as $chunk) {
                    // SSE 格式：data: {json}\n\n
                    echo 'data: '.json_encode(
                        ['text' => $chunk],
                        JSON_UNESCAPED_UNICODE,
                    )."\n\n";

                    // 立即发送到客户端
                    if (ob_get_level() > 0) {
                        ob_flush();
                    }
                    flush();
                }

                // 发送结束信号
                echo "data: [DONE]\n\n";
                flush();
            },
            200,
            [
                'Content-Type' => 'text/event-stream',
                'Cache-Control' => 'no-cache',
                'Connection' => 'keep-alive',
            ],
        );
    }
}
```

### 7.4 前端 JavaScript（使用 Fetch API）

```javascript
// 使用 Fetch API + ReadableStream（比 EventSource 更灵活）
async function streamChat(message) {
    const responseDiv = document.getElementById('response');
    responseDiv.textContent = '';

    const response = await fetch('/api/chat/stream', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ message }),
    });

    const reader = response.body.getReader();
    const decoder = new TextDecoder();

    while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        const text = decoder.decode(value);
        const lines = text.split('\n\n');

        for (const line of lines) {
            if (!line.startsWith('data: ')) continue;
            const data = line.slice(6);  // 去掉 "data: " 前缀

            if (data === '[DONE]') return;

            try {
                const parsed = JSON.parse(data);
                responseDiv.textContent += parsed.text;
            } catch (e) {
                // 忽略解析错误（可能是不完整的 chunk）
            }
        }
    }
}

// 绑定表单
document.getElementById('chat-form').addEventListener('submit', (e) => {
    e.preventDefault();
    const input = document.getElementById('user-input');
    streamChat(input.value);
    input.value = '';
});
```

### 7.5 结合 Chat 组件实现多轮流式对话

```php
class MultiRoundStreamController
{
    public function __construct(
        private readonly ChatInterface $chat,
    ) {}

    #[Route('/api/chat/stream/{conversationId}', methods: ['POST'])]
    public function stream(string $conversationId, Request $request): StreamedResponse
    {
        $userMessage = $request->getPayload()->getString('message');

        return new StreamedResponse(function () use ($conversationId, $userMessage) {
            header('Content-Type: text/event-stream');
            header('Cache-Control: no-cache');

            // submit() 返回的 DeferredResult 也支持 asStream()
            $response = $this->chat->submit($conversationId, $userMessage);

            foreach ($response->asStream() as $chunk) {
                echo 'data: '.json_encode(
                    ['text' => $chunk],
                    JSON_UNESCAPED_UNICODE,
                )."\n\n";

                if (ob_get_level() > 0) {
                    ob_flush();
                }
                flush();
            }

            echo "data: [DONE]\n\n";
            flush();
        });
    }
}
```

### 7.6 生产环境注意事项

**Nginx 配置**——必须禁用缓冲以实现真正的流式传输：

```nginx
location /api/chat/stream {
    proxy_pass http://php_upstream;
    proxy_buffering off;           # 禁用代理缓冲
    proxy_cache off;               # 禁用缓存
    proxy_read_timeout 300s;       # 延长超时（AI 生成可能需要较长时间）
    chunked_transfer_encoding on;
}
```

**PHP-FPM 配置**——确保输出不被缓冲：

```ini
; php.ini
output_buffering = Off
implicit_flush = On
```

**超时和断连处理**：

```php
// 设置脚本执行超时（防止长时间连接）
set_time_limit(120);

// 检测客户端是否断开
if (connection_aborted()) {
    // 客户端已断开，停止生成
    return;
}
```

---

## 8. 错误处理与生产优化最佳实践

### 8.1 通用错误处理模式

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
        sleep($e->retryAfter ?? 5);
        return safeInvoke($platform, $messages, $model, $options);
    } catch (ExceedContextSizeException) {
        // 输入过长，截断消息历史
        $truncated = new MessageBag(...array_slice($messages->all(), -5));
        return safeInvoke($platform, $truncated, $model, $options);
    } catch (ContentFilterException) {
        return '您的输入触发了安全过滤，请修改后重试。';
    } catch (\Throwable $e) {
        $this->logger->error('AI 调用异常', ['exception' => $e]);
        return '抱歉，AI 服务暂时不可用。';
    }
}
```

### 8.2 Token 成本优化

| 优化策略 | 做法 | 效果 |
|---------|------|------|
| **精简系统提示** | 200 字而非 1000 字 | 减少 30-50% 输入 Token |
| **限制上下文** | 只保留最近 N 轮对话 | 避免上下文爆炸 |
| **选择合适模型** | 简单任务用 mini 模型 | 降低 80-90% 成本 |
| **缓存重复请求** | CachePlatform | 重复请求零成本 |
| **设置 max_tokens** | 限制输出长度 | 控制输出成本 |

---

## 9. 本章小结

通过六个基础场景，我们掌握了以下核心模式和关键注意事项：

| 模式 | 场景 | 关键 API | 生产要点 |
|------|------|---------|---------|
| **简单问答** | 聊天机器人 | `Platform::invoke()` + `MessageBag` | 异常体系、事件监控 |
| **多轮对话** | 客服系统 | `Chat::initiate()` + `submit()` | 上下文长度管理、Store 选型 |
| **结构化输出** | 数据提取 | `StructuredOutput` + DTO + `#[With]` | Validator 验证、description 质量 |
| **多模态** | 内容理解 | `Image`/`Document`/`Audio`/`Video` | 平台能力差异、文件大小限制 |
| **翻译** | 多语言 | System Prompt Engineering | 术语一致性、低 temperature |
| **流式输出** | 实时对话 | `asStream()` + SSE | Nginx/PHP 缓冲配置、断连处理 |

---

## 10. 下一步

在 [第 10 章](10-scenarios-intermediate.md) 中，我们将进入进阶场景——涉及 Agent 工具调用循环、RAG 知识库检索管线、多智能体协作编排等更复杂的模式。
