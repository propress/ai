# 第 9 章：实战场景（基础篇）

## 🎯 本章学习目标

通过 6 个基础实战场景，将 Platform 和 Chat 组件的核心概念付诸实践：从简单的问答机器人到多模态内容理解，从结构化数据提取到实时流式对话，掌握 AI 应用开发的常见模式。

---

## 1. 回顾

在前八章中，我们系统学习了 Symfony AI 的所有核心组件。本章开始进入**实战篇**，通过真实业务场景将知识融会贯通。

**本章侧重点**：Platform + Chat 组件的基础用法，不涉及 Agent 工具调用和 Store 向量检索。

---

## 2. 场景一：基础问答聊天机器人

### 2.1 业务场景

在产品官网嵌入一个 AI 聊天窗口，用户提问，AI 根据系统提示词回答。

**适用场景**：官网客服、FAQ 助手、产品介绍机器人。

### 2.2 所需依赖

```bash
composer require symfony/ai-platform symfony/ai-anthropic-platform
# 或使用 OpenAI：
# composer require symfony/ai-platform symfony/ai-open-ai-platform
```

### 2.3 核心实现

```php
<?php

use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 1. 创建平台实例
$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY']);

// 2. 构建消息
$messages = new MessageBag(
    Message::forSystem('你是一个友好的产品助手。用简洁专业的中文回答用户问题。'),
    Message::ofUser('你们的产品有哪些定价方案？'),
);

// 3. 调用 AI
$response = $platform->invoke(
    $messages,
    new Symfony\AI\Platform\Bridge\Anthropic\Claude(),
);

// 4. 获取结果
echo $response->asText();
```

### 2.4 流式响应

对于 Web 应用，流式输出能大幅提升用户体验：

```php
$response = $platform->invoke($messages, $model);

// 流式输出——逐块获取文本
foreach ($response->asStream() as $chunk) {
    echo $chunk;
    flush();
}
```

### 2.5 模型选项

```php
use Symfony\AI\Platform\Bridge\Anthropic\Claude;

$model = new Claude(Claude::CLAUDE_3_5_SONNET);

$response = $platform->invoke($messages, $model, [
    'temperature' => 0.7,      // 创造性（0-1）
    'max_tokens' => 1024,      // 最大输出 Token 数
]);
```

> 💡 **提示**：`temperature` 越低回答越确定，适合事实性问答；越高越有创造性，适合内容生成。

### 2.6 切换平台

Symfony AI 的 Bridge 架构让平台切换只需改两行代码：

```php
// 从 Anthropic 切换到 OpenAI：
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Bridge\OpenAi\GPT;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);
$response = $platform->invoke($messages, new GPT(GPT::GPT_4O));
```

---

## 3. 场景二：多轮对话与会话持久化

### 3.1 业务场景

构建一个能记住上下文的客服聊天系统。用户可以进行多轮对话，系统记住之前的交流内容。

**适用场景**：在线客服、技术支持、咨询服务。

### 3.2 所需依赖

```bash
composer require symfony/ai-platform symfony/ai-anthropic-platform \
    symfony/ai-chat symfony/ai-redis-message-store
```

### 3.3 使用 Chat 组件

```php
<?php

use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\Bridge\Redis\Store as RedisStore;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Bridge\Anthropic\Claude;
use Symfony\AI\Platform\Message;

// 1. 创建平台和消息存储
$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY']);
$store = new RedisStore(new \Redis(/* 连接配置 */));
$model = new Claude();

// 2. 创建 Chat 实例
$chat = new Chat($platform, $model, $store);

// 3. 发起新对话
$conversation = $chat->initiate(
    Message::forSystem('你是一个专业的客服助手。'),
);

// 4. 第一轮对话
$response = $chat->submit($conversation->getId(), '我想退货');
echo $response->asText();

// 5. 第二轮对话（自动携带上下文）
$response = $chat->submit($conversation->getId(), '订单号是 ORD-20240115');
echo $response->asText();
```

### 3.4 会话生命周期

```
initiate()           submit()           submit()
┌─────────┐    ┌──────────────┐   ┌──────────────┐
│ 创建会话  │──→│ 第一轮对话     │──→│ 第二轮对话    │──→ ...
│ System   │    │ +User +AI     │   │ +User +AI    │
│ Prompt   │    │ 保存到 Store   │   │ 保存到 Store  │
└─────────┘    └──────────────┘   └──────────────┘
```

### 3.5 消息存储后端对比

| 后端 | 适用场景 | 持久化 | 性能 |
|------|---------|--------|------|
| InMemory | 测试 | ❌ | ⚡ 极快 |
| Redis | 生产环境、高并发 | ✅ | ⚡ 快 |
| Doctrine DBAL | 关系数据库项目 | ✅ | 🔄 中等 |
| MongoDB | 文档型存储 | ✅ | ⚡ 快 |
| Session | 简单 Web 应用 | 📦 会话级 | ⚡ 快 |

---

## 4. 场景三：结构化数据提取

### 4.1 业务场景

从非结构化文本中提取结构化信息——例如从客户反馈中提取情感、类别、关键词，或从商品描述中提取属性。

**适用场景**：表单解析、情感分析、实体识别、数据录入自动化。

### 4.2 所需依赖

```bash
composer require symfony/ai-platform symfony/ai-open-ai-platform
```

### 4.3 定义数据结构（DTO）

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

### 4.4 使用 StructuredOutput

```php
<?php

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Bridge\OpenAi\GPT;
use Symfony\AI\Platform\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);

$messages = new MessageBag(
    Message::forSystem('从用户输入中提取商品信息。'),
    Message::ofUser(
        '这款华为 Mate 60 Pro 手机真不错，5999 元的价格很合理，'
        .'拍照效果特别好，电池续航也很长，就是重量稍微有点重。'
    ),
);

$response = $platform->invoke(
    $messages,
    new GPT(GPT::GPT_4O),
    ['output' => ProductInfo::class],  // 指定输出类型
);

/** @var ProductInfo $product */
$product = $response->unwrap();

echo $product->name;       // "华为 Mate 60 Pro"
echo $product->price;      // 5999.0
echo $product->category;   // "电子产品"
echo $product->sentiment;  // Sentiment::POSITIVE
```

### 4.5 嵌套 DTO

StructuredOutput 支持嵌套对象：

```php
class OrderExtraction
{
    public function __construct(
        public readonly string $orderNumber,
        public readonly CustomerInfo $customer,
        /** @var OrderItem[] */
        public readonly array $items,
        public readonly float $totalAmount,
    ) {}
}

class CustomerInfo
{
    public function __construct(
        public readonly string $name,
        public readonly string $phone,
        public readonly ?string $email = null,
    ) {}
}

class OrderItem
{
    public function __construct(
        public readonly string $productName,
        public readonly int $quantity,
        public readonly float $unitPrice,
    ) {}
}
```

### 4.6 配合 Validator

```php
use Symfony\Component\Validator\Constraints as Assert;

class ProductInfo
{
    public function __construct(
        #[Assert\NotBlank]
        public readonly string $name,
        #[Assert\Positive]
        public readonly float $price,
        // ...
    ) {}
}

// 提取后验证
$product = $response->unwrap();
$violations = $validator->validate($product);

if (count($violations) > 0) {
    // AI 提取的数据不符合约束，可以重试或人工复核
}
```

> 💡 **提示**：`#[With]` 属性的 `description` 参数非常重要——它告诉 AI 每个字段应该包含什么内容，直接影响提取质量。

---

## 5. 场景四：多模态内容理解

### 5.1 业务场景

让 AI 理解图片、PDF、音频等非文本内容——例如分析产品图片、解析合同文档、转录会议录音。

**适用场景**：商品图片分析、文档审查、音频转录、视频理解。

### 5.2 所需依赖

```bash
composer require symfony/ai-platform symfony/ai-open-ai-platform
# 视频理解需要 Gemini：
# composer require symfony/ai-gemini-platform
```

### 5.3 图片分析

```php
use Symfony\AI\Platform\Message;
use Symfony\AI\Platform\Message\Content\Image;

// 从文件加载
$messages = new MessageBag(
    Message::ofUser(
        Image::fromFile('/path/to/product.jpg'),
        '请描述这张产品图片的内容，包括颜色、材质和适用场景。',
    ),
);

// 从 URL 加载
$messages = new MessageBag(
    Message::ofUser(
        Image::fromUrl('https://example.com/photo.png'),
        '这张图片中有什么？',
    ),
);

$response = $platform->invoke($messages, new GPT(GPT::GPT_4O));
echo $response->asText();
```

### 5.4 PDF 文档处理

```php
use Symfony\AI\Platform\Message\Content\Document;

$messages = new MessageBag(
    Message::ofUser(
        Document::fromFile('/path/to/contract.pdf'),
        '请总结这份合同的主要条款和关键日期。',
    ),
);

$response = $platform->invoke($messages, $model);
echo $response->asText();
```

### 5.5 音频处理

```php
use Symfony\AI\Platform\Message\Content\Audio;

// 音频转文字（Speech-to-Text）
$messages = new MessageBag(
    Message::ofUser(
        Audio::fromFile('/path/to/meeting.mp3'),
        '请转录这段录音内容。',
    ),
);

$response = $platform->invoke($messages, $model);
echo $response->asText();
```

### 5.6 视频理解（Gemini）

```php
use Symfony\AI\Platform\Bridge\Google\PlatformFactory;
use Symfony\AI\Platform\Bridge\Google\Gemini;
use Symfony\AI\Platform\Message\Content\Video;

$platform = PlatformFactory::create($_ENV['GOOGLE_API_KEY']);

$messages = new MessageBag(
    Message::ofUser(
        Video::fromFile('/path/to/demo.mp4'),
        '请分析这段视频的内容，列出关键时间点和对应的事件。',
    ),
);

$response = $platform->invoke($messages, new Gemini(Gemini::GEMINI_2_FLASH));
echo $response->asText();
```

> ⚠️ **注意**：不是所有平台都支持所有模态。使用前请检查平台和模型的能力。GPT-4o 支持图片和音频，Gemini 原生支持视频。

---

## 6. 场景五：多语言翻译助手

### 6.1 业务场景

构建一个支持多语言翻译的 AI 助手，能保持术语一致性，支持多轮修正。

**适用场景**：文档翻译、本地化、多语言客服。

### 6.2 核心实现

```php
<?php

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Bridge\OpenAi\GPT;
use Symfony\AI\Platform\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);

$systemPrompt = <<<PROMPT
你是一个专业的技术文档翻译专家。

翻译规则：
1. 保持技术术语的准确性
2. 代码块和变量名不翻译
3. 保持 Markdown 格式
4. 术语表：
   - Platform → 平台
   - Agent → 智能代理
   - Store → 存储
   - Bridge → 桥接器
   - Toolbox → 工具箱
PROMPT;

$messages = new MessageBag(
    Message::forSystem($systemPrompt),
    Message::ofUser(
        "请将以下英文翻译为中文：\n\n"
        ."The Agent component provides a framework for building AI agents "
        ."that can use tools to accomplish tasks. The Toolbox manages all "
        ."available tools and their execution."
    ),
);

$response = $platform->invoke($messages, new GPT(GPT::GPT_4O));
echo $response->asText();
```

### 6.3 结构化翻译结果

```php
class TranslationResult
{
    public function __construct(
        #[With(description: '翻译后的文本')]
        public readonly string $translation,
        #[With(description: '翻译质量评分 1-10')]
        public readonly int $qualityScore,
        #[With(description: '翻译中的注意事项')]
        public readonly array $notes,
    ) {}
}

$response = $platform->invoke($messages, $model, [
    'output' => TranslationResult::class,
]);

$result = $response->unwrap();
echo $result->translation;    // 翻译文本
echo $result->qualityScore;   // 9
```

---

## 7. 场景六：实时流式对话

### 7.1 业务场景

构建一个实时流式对话的 Web 应用，使用 Server-Sent Events（SSE）将 AI 回复逐字推送到前端。

**适用场景**：ChatGPT 式交互界面、实时客服、在线教育。

### 7.2 Symfony 控制器

```php
<?php

namespace App\Controller;

use Symfony\AI\Platform\PlatformInterface;
use Symfony\AI\Platform\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\StreamedResponse;
use Symfony\Component\Routing\Attribute\Route;

class ChatController
{
    #[Route('/api/chat', methods: ['POST'])]
    public function chat(
        Request $request,
        PlatformInterface $platform,
    ): StreamedResponse {
        $userMessage = $request->getPayload()->getString('message');

        $messages = new MessageBag(
            Message::forSystem('你是一个友好的中文助手。'),
            Message::ofUser($userMessage),
        );

        return new StreamedResponse(function () use ($platform, $messages) {
            $response = $platform->invoke($messages, $this->model);

            header('Content-Type: text/event-stream');
            header('Cache-Control: no-cache');

            foreach ($response->asStream() as $chunk) {
                echo "data: ".json_encode(['text' => $chunk])."\n\n";
                flush();
            }

            echo "data: [DONE]\n\n";
            flush();
        });
    }
}
```

### 7.3 前端 JavaScript

```javascript
const eventSource = new EventSource('/api/chat', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ message: '你好！' }),
});

eventSource.onmessage = (event) => {
    if (event.data === '[DONE]') {
        eventSource.close();
        return;
    }
    const data = JSON.parse(event.data);
    document.getElementById('response').textContent += data.text;
};
```

### 7.4 结合 Chat 持久化

```php
// 使用 Chat 组件管理多轮对话 + 流式输出
$chat = new Chat($platform, $model, $store);

// 如果是新对话
$conversation = $chat->initiate(Message::forSystem('你是一个友好的助手。'));

// 流式提交
$response = $chat->submit($conversation->getId(), $userMessage);

foreach ($response->asStream() as $chunk) {
    // SSE 推送到前端
}
```

---

## 8. 本章小结

通过六个基础场景，我们掌握了以下核心模式：

| 模式 | 场景 | 关键 API |
|------|------|---------|
| **简单问答** | 聊天机器人 | `Platform::invoke()` + `MessageBag` |
| **多轮对话** | 客服系统 | `Chat::initiate()` + `submit()` |
| **结构化输出** | 数据提取 | `StructuredOutput` + DTO |
| **多模态** | 内容理解 | `Image`/`Document`/`Audio`/`Video` |
| **翻译** | 多语言 | System Prompt 工程 |
| **流式输出** | 实时对话 | `asStream()` + SSE |

---

## 9. 下一步

在 [第 10 章](10-scenarios-intermediate.md) 中，我们将进入进阶场景——涉及 Agent 工具调用、RAG 知识库检索、多智能体协作等更复杂的模式。
