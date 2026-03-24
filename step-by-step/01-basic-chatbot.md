# 基础问答聊天机器人

## 业务场景

你正在开发一个产品官网，需要在页面上嵌入一个 AI 聊天窗口：用户提出问题，AI 根据预设的系统提示词给出回答。

**典型应用：** 官网客服机器人、FAQ 助手、产品介绍机器人、内部知识库问答、技术文档查询

## 涉及模块

| 模块 | 包名 | 用途 |
|------|------|------|
| **Platform** | `symfony/ai-platform` | 核心抽象层——定义 `PlatformInterface`、消息系统、结果系统、元数据等，与具体 AI 平台无关 |
| **Bridge（Anthropic）** | `symfony/ai-anthropic-platform` | 本教程的主实现，提供 Anthropic Claude 的 `PlatformFactory`、`ModelClient`、`Contract` 等 |
| **Bridge（OpenAI）** | `symfony/ai-open-ai-platform` | 完整示例使用，展示平台切换的便捷性 |

## 架构概述

Symfony AI 采用 **Bridge 架构**，将核心抽象与平台实现彻底分离：

- **`PlatformInterface`** 定义了统一的 `invoke()` 方法——所有 AI 平台共享同一调用方式
- **`Contract`** 负责将平台无关的消息格式（`MessageBag`）转换为各平台的 API 请求格式（如 Anthropic 的 `messages` 格式、OpenAI 的 `chat/completions` 格式）
- **`PlatformFactory`** 是每个 Bridge 提供的便捷工厂，一行代码即可创建完整配置的 Platform 实例

这意味着你的业务代码只依赖核心抽象层（`MessageBag`、`DeferredResult`、`Metadata`），切换 AI 平台只需更换 Bridge 包和工厂类。

## 项目流程图

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

> **📝 知识扩展：** 流程中的 `Contract` 是一个常被忽略但至关重要的组件。它基于 Symfony Serializer，负责将 `Message`、`ToolCall` 等对象序列化为各平台特定的 JSON 格式。每个 Bridge 都有自己的 Contract 实现（如 `AnthropicContract`、`OpenAiContract`），你无需直接操作它——`PlatformFactory` 会自动创建并注入。

## 前置准备

### 环境要求

- PHP >= 8.2
- Composer

### 安装依赖

```bash
composer require symfony/ai-platform symfony/ai-anthropic-platform
```

`symfony/ai-platform` 是核心包，`symfony/ai-anthropic-platform` 是 Anthropic Claude 的 Bridge 包。

### 设置 API 密钥

```bash
export ANTHROPIC_API_KEY="sk-ant-your-api-key-here"
```

> **🔒 安全建议：** 永远不要将 API 密钥硬编码在源代码中。在 Symfony 项目中，应使用 `.env.local` 文件或服务器环境变量来管理密钥；在 CI/CD 中使用 Secrets 管理。API 密钥一旦泄露会被自动扫描程序利用，导致意外费用。

---

## Step 1：创建 Platform 实例

每个 Bridge 都提供了 `PlatformFactory`——一个静态工厂类，用于创建 `Platform` 实例。不同平台的工厂参数略有不同，但第一个参数通常是 API 密钥。

```php
<?php

use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;

// 通过工厂方法创建 Platform 实例
$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY']);
```

Anthropic 的 `PlatformFactory::create()` 完整签名如下：

```php
public static function create(
    string $apiKey,                          // 必需：API 密钥
    ?HttpClientInterface $httpClient = null,  // 可选：自定义 HTTP 客户端
    ModelCatalogInterface $modelCatalog = new ModelCatalog(),  // 可选：模型目录
    ?Contract $contract = null,              // 可选：自定义序列化契约
    ?EventDispatcherInterface $eventDispatcher = null,  // 可选：事件调度器
    string $cacheRetention = 'short',        // Anthropic 特有：提示缓存策略（'none'|'short'|'long'）
): Platform
```

大多数场景下只需传入 API 密钥，其余参数有合理默认值。如果需要自定义 HTTP 超时：

```php
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create(['timeout' => 60]);
$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY'], $httpClient);
```

> **🏭 生产建议：** 在 Symfony 项目中推荐通过 AI Bundle（`symfony/ai-bundle`）来自动配置 Platform，通过依赖注入获取实例，而不是手动调用工厂方法。AI Bundle 会自动处理 HTTP 客户端、事件调度器等的注入。

---

## Step 2：构建消息并发送

LLM 的交互基于消息列表。Symfony AI 提供了两个核心类来管理消息：

- **`Message`** — 静态工厂类，创建不同角色的消息
- **`MessageBag`** — 消息容器，持有完整的对话历史

```php
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 构建消息：系统提示 + 用户问题
$messages = new MessageBag(
    Message::forSystem('你是一个专业的产品客服。你的任务是回答用户关于我们 SaaS 产品的问题。回答要简洁、专业、友好。如果不确定答案，请诚实地告诉用户。'),
    Message::ofUser('你们的产品支持哪些编程语言？'),
);

// 调用 AI 平台
$result = $platform->invoke('claude-sonnet-4-20250514', $messages);

// 获取文本回复
echo $result->asText();
// 输出示例："我们的产品目前支持 PHP、Python、JavaScript、Go 等主流编程语言..."
```

### 消息角色详解

`Message` 类提供了四种工厂方法，对应 LLM 对话中的不同角色：

| 工厂方法 | 创建的类型 | 角色 | 用途 |
|---------|-----------|------|------|
| `Message::forSystem(string\|Template $content)` | `SystemMessage` | 系统 | 定义 AI 的角色、行为规则和知识背景 |
| `Message::ofUser(string\|ContentInterface ...$content)` | `UserMessage` | 用户 | 用户发送的问题或指令，支持多模态内容 |
| `Message::ofAssistant(?string $content, ?array $toolCalls)` | `AssistantMessage` | 助手 | AI 的回复（用于构建多轮对话上下文）|
| `Message::ofToolCall(ToolCall $toolCall, string $content)` | `ToolCallMessage` | 工具 | 工具调用的执行结果（后续章节讲解） |

> **📝 知识扩展：** `Message::ofUser()` 接受可变参数（`...$content`），支持同时传入文本和多媒体内容（如图片、音频、PDF）。这是 Symfony AI 支持多模态输入的基础——在后续章节中，你会看到如何传入图片让 AI 进行视觉分析。

### 使用 Template 构建动态系统提示词

当系统提示词需要包含动态变量时，可以使用 `Template`：

```php
use Symfony\AI\Platform\Message\Template;

$messages = new MessageBag(
    Message::forSystem(Template::string(
        '你是 {product_name} 的客服。当前用户等级：{user_level}。请根据用户等级提供对应的服务。'
    )),
    Message::ofUser('我想升级套餐'),
);

// 调用时通过 template_vars 传入变量值
$result = $platform->invoke('claude-sonnet-4-20250514', $messages, [
    'template_vars' => [
        'product_name' => 'CloudFlow',
        'user_level' => 'Pro',
    ],
]);
```

`Template::string()` 使用简单的 `{variable}` 占位符替换。如果需要更复杂的表达式逻辑，可以使用 `Template::expression()`，它基于 Symfony ExpressionLanguage 组件。

### MessageBag 的操作方法

`MessageBag` 不只是一个简单数组，它提供了丰富的操作方法：

```php
// 每个 MessageBag 都有唯一的 UUID v7 标识符
echo $messages->getId();  // 如："01914a5c-7d3e-7..."

// 获取特定角色的消息
$system = $messages->getSystemMessage();   // 获取系统消息（如果有）
$user = $messages->getUserMessage();       // 获取最后一条用户消息

// 检查消息中是否包含多媒体内容
$hasImages = $messages->containsImage();
$hasAudio = $messages->containsAudio();
```

`MessageBag` 同时提供了**不可变**和**可变**两种操作方式：

```php
// with() —— 不可变：返回新 MessageBag，原实例不受影响
$newBag = $messages->with(Message::ofUser('追加问题'));

// merge() —— 不可变：合并两个 MessageBag，返回新实例
$merged = $messages->merge($otherMessages);

// withSystemMessage() —— 不可变：替换系统消息
$newBag = $messages->withSystemMessage(
    new SystemMessage('新的系统提示词')
);

// withoutSystemMessage() —— 不可变：移除系统消息
$noSystem = $messages->withoutSystemMessage();

// add() —— 可变：直接修改当前实例
$messages->add(Message::ofUser('你好'));
```

> **💡 提示：** 在需要从同一上下文分叉出不同对话分支时（如 A/B 测试不同的系统提示词），不可变方法 `with()` 和 `withSystemMessage()` 特别有用。可变方法 `add()` 则适合在循环中逐步构建对话。

---

## Step 3：理解 DeferredResult 与 Metadata

`$platform->invoke()` 的返回值不是直接的文本字符串，而是一个 `DeferredResult` 对象。这是 Symfony AI 的一个重要设计——**延迟求值（Lazy Evaluation）**。

### 延迟求值

```php
// invoke() 返回后，HTTP 请求可能还未完成
$result = $platform->invoke('claude-sonnet-4-20250514', $messages);

// 直到调用 as*() 方法时，才真正消费响应
$text = $result->asText();  // 此刻消费 HTTP 响应并解析结果
```

`DeferredResult` 提供了多种输出方法，适用于不同场景：

| 方法 | 返回类型 | 用途 |
|------|---------|------|
| `asText()` | `string` | 获取纯文本回复（最常用） |
| `asStream()` | `Generator` | 流式输出，逐块获取内容 |
| `asObject()` | `object` | 结构化输出，返回 PHP 对象 |
| `asToolCalls()` | `ToolCall[]` | 获取工具调用请求 |
| `asBinary()` | `string` | 获取二进制内容（如生成的图片） |
| `asFile(string $path)` | `void` | 将二进制内容直接写入文件 |
| `asDataUri(?string $mimeType)` | `string` | 获取 Base64 Data URI |
| `asVectors()` | `Vector[]` | 获取文本嵌入向量 |
| `asReranking()` | `RerankingEntry[]` | 获取重排序结果 |

> **⚠️ 注意：** `as*()` 方法只能调用一次有效——底层 HTTP 响应在首次消费后即关闭。如果需要同时获取文本和元数据，先调用 `asText()`，再调用 `getMetadata()`。

### Metadata 与 Token 用量

每次调用都会产生元数据，最重要的是 Token 用量——它直接关系到 API 费用：

```php
$result = $platform->invoke('claude-sonnet-4-20250514', $messages);
$text = $result->asText();

echo $text;

// 获取 Token 用量
$tokenUsage = $result->getMetadata()->get('token_usage');

if (null !== $tokenUsage) {
    echo sprintf(
        "\n[Token 用量] 输入: %d, 输出: %d, 合计: %d\n",
        $tokenUsage->getPromptTokens(),
        $tokenUsage->getCompletionTokens(),
        $tokenUsage->getTotalTokens(),
    );
}
```

`TokenUsage` 还提供了更精细的消耗信息：

```php
$tokenUsage->getThinkingTokens();       // 思考 Token（支持 thinking 的模型）
$tokenUsage->getCachedTokens();          // 命中缓存的 Token 数
$tokenUsage->getCacheCreationTokens();   // 缓存创建消耗的 Token
$tokenUsage->getCacheReadTokens();       // 缓存读取的 Token
$tokenUsage->getRemainingTokens();       // 剩余配额（部分平台返回）
```

> **🏭 生产建议：** 建议在生产环境中记录每次请求的 Token 用量，用于成本监控和异常检测。可以结合事件系统（Step 6）自动化收集，而不是在业务代码中手动获取。

---

## Step 4：多轮对话——维护上下文

在真实场景中，用户会连续提问。LLM 本身没有记忆能力，每次请求都是独立的——你需要将完整的对话历史传给它，它才能理解上下文。

`MessageBag` 提供了不可变的 `with()` 方法来优雅地追加消息：

```php
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// === 第一轮对话 ===
$messages = new MessageBag(
    Message::forSystem('你是一个专业的 SaaS 产品客服。回答要简洁专业。'),
    Message::ofUser('你们的产品多少钱？'),
);

$result = $platform->invoke('claude-sonnet-4-20250514', $messages);
$answer1 = $result->asText();
echo "客服：{$answer1}\n";

// === 第二轮对话 ===
// with() 返回新的 MessageBag，原 $messages 不受影响（不可变设计）
$messages = $messages
    ->with(Message::ofAssistant($answer1))
    ->with(Message::ofUser('专业版和企业版有什么区别？'));

$result = $platform->invoke('claude-sonnet-4-20250514', $messages);
$answer2 = $result->asText();
echo "客服：{$answer2}\n";

// === 第三轮对话 ===
$messages = $messages
    ->with(Message::ofAssistant($answer2))
    ->with(Message::ofUser('企业版可以试用吗？'));

$result = $platform->invoke('claude-sonnet-4-20250514', $messages);
echo "客服：" . $result->asText() . "\n";
```

`with()` 是不可变操作——它会 `clone` 当前 `MessageBag` 并在副本上追加消息，返回新实例。这在需要从同一上下文分叉出不同对话分支时很有用。

如果你需要原地修改（例如在循环中构建对话），可以使用 `add()` 方法：

```php
$messages = new MessageBag(Message::forSystem('你是一个客服。'));

// add() 是可变操作，直接修改当前实例
$messages->add(Message::ofUser('你好'));
```

> **⚠️ 注意：** 每轮对话都会将完整历史发送给 AI，Token 消耗随对话轮数线性增长。10 轮深度对话可能消耗数千 Token。在生产环境中，务必设置对话轮数上限或实现消息截断策略。

---

## Step 5：流式输出

在网页聊天中，用户希望看到 AI 逐字打出回答，而不是等待数秒后一次性显示。这里用 Gemini 来演示流式输出，展示跨平台的一致性：

```bash
composer require symfony/ai-gemini-platform
```

```php
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['GEMINI_API_KEY']);

$messages = new MessageBag(
    Message::forSystem('你是一个产品客服，请详细解释我们的退款政策。'),
    Message::ofUser('如果我购买后不满意，可以退款吗？'),
);

// 通过 stream 选项开启流式输出
$result = $platform->invoke('gemini-2.5-flash', $messages, [
    'stream' => true,
]);

// 逐块输出
foreach ($result->asStream() as $chunk) {
    echo $chunk;
    flush();
}
echo "\n";

// 流式输出完成后，仍可获取 Token 用量
$tokenUsage = $result->getMetadata()->get('token_usage');
```

> **💡 提示：** 在 Web 应用中，流式输出通常通过 Server-Sent Events（SSE）推送给前端。在 `asStream()` 循环中，将每个 `$chunk` 以 `data: {chunk}\n\n` 格式输出并调用 `flush()` 即可。Symfony AI 的 Chat 组件（`symfony/ai-chat`）提供了开箱即用的 SSE 集成。

---

## Step 6：模型选项与参数控制

### 通过选项数组传参

你可以通过 `invoke()` 的第三个参数控制模型行为：

```php
$result = $platform->invoke('claude-sonnet-4-20250514', $messages, [
    'max_output_tokens' => 200,  // 限制最大输出长度
    'temperature' => 0.3,        // 降低随机性，让回答更确定
]);
```

### 通过模型名称的查询字符串传参

Symfony AI 支持将参数附加在模型名称上，这在快速测试时特别方便：

```php
// 等价于上面的选项数组写法
$result = $platform->invoke(
    'claude-sonnet-4-20250514?max_output_tokens=200&temperature=0.3',
    $messages,
);
```

查询字符串会被自动解析：数字转换为 `int` 或 `float`，`true`/`false` 转换为布尔值。如果同时使用查询字符串和选项数组，选项数组的值优先。

### 常用参数说明

| 参数 | 类型 | 说明 |
|------|------|------|
| `temperature` | `float` | 控制输出随机性，0.0 最确定，2.0 最随机。客服场景建议 0.3 以下 |
| `max_output_tokens` | `int` | 限制输出 Token 上限，防止过长回复增加成本 |
| `stream` | `bool` | 是否开启流式输出 |
| `top_p` | `float` | 核采样参数，与 temperature 二选一调节 |

> **💡 提示：** 不同平台支持的参数名称可能略有差异（如 OpenAI 用 `max_output_tokens`，部分旧 API 用 `max_tokens`）。Symfony AI 的 Contract 层会自动处理这些差异，你只需使用统一的参数名。

---

## Step 7：事件系统——日志与监控

Symfony AI 内置了事件系统，在平台调用的关键节点分发事件。这对于日志记录、性能监控和调试非常有价值。

### 两个核心事件

| 事件 | 触发时机 | 可变性 | 用途 |
|------|---------|--------|------|
| `InvocationEvent` | `invoke()` 处理**之前** | 可修改模型、输入、选项 | 记录请求日志、动态切换模型、注入默认选项 |
| `ResultEvent` | 结果创建**之后** | 可修改 DeferredResult | 记录响应日志、包装结果、性能追踪 |

### 实现日志监听

要使用事件系统，需要在创建 Platform 时传入 `EventDispatcherInterface`：

```php
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Event\InvocationEvent;
use Symfony\AI\Platform\Event\ResultEvent;
use Symfony\Component\EventDispatcher\EventDispatcher;

// 创建事件调度器并注册监听器
$dispatcher = new EventDispatcher();

// InvocationEvent：请求发出前触发
$dispatcher->addListener(InvocationEvent::class, function (InvocationEvent $event) {
    echo sprintf("[LOG] 调用模型: %s\n", $event->getModel()->getName());

    // 事件是可变的——可以在发送前修改选项
    $options = $event->getOptions();
    $options['temperature'] = $options['temperature'] ?? 0.7;
    $event->setOptions($options);
});

// ResultEvent：获得结果后触发
$dispatcher->addListener(ResultEvent::class, function (ResultEvent $event) {
    $tokenUsage = $event->getDeferredResult()->getMetadata()->get('token_usage');
    if (null !== $tokenUsage) {
        echo sprintf("[LOG] Token 消耗: %d\n", $tokenUsage->getTotalTokens());
    }
});

// 将调度器传入工厂方法
$platform = PlatformFactory::create(
    $_ENV['ANTHROPIC_API_KEY'],
    eventDispatcher: $dispatcher,
);
```

> **🏭 生产建议：** 在 Symfony 项目中使用 AI Bundle 时，事件调度器会自动注入，你只需创建 EventSubscriber 或 EventListener 服务即可。结合 Monolog 可以轻松将 AI 调用日志写入文件、发送到 ELK 等日志系统。

> **📝 知识扩展：** `InvocationEvent` 的可变性是一个强大的特性。你可以利用它在全局层面实现功能，例如：根据用户等级自动切换模型（`$event->setModel(...)`）、在所有请求中注入默认选项、或者在开发环境中将请求重定向到更便宜的模型。

---

## 完整示例：命令行多轮聊天机器人

以下是一个整合了上述所有概念的完整示例，使用 OpenAI 实现——展示平台切换的便捷性：

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Event\InvocationEvent;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\EventDispatcher\EventDispatcher;

// --- 配置事件监听 ---
$dispatcher = new EventDispatcher();
$dispatcher->addListener(InvocationEvent::class, function (InvocationEvent $event) {
    echo sprintf("  [调用模型: %s]\n", $event->getModel()->getName());
});

// --- 创建 Platform ---
$platform = PlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    eventDispatcher: $dispatcher,
);

// --- 初始化对话 ---
$messages = new MessageBag(
    Message::forSystem(
        '你是一个友好的 SaaS 产品客服。产品名称是 CloudFlow。'
        . '主要功能：项目管理、团队协作、自动化工作流。'
        . '定价：基础版 ¥99/月，专业版 ¥299/月，企业版联系销售。'
        . '回答要简洁、友好、准确。如果用户的问题超出产品范围，礼貌地引导回来。'
    ),
);

$totalTokens = 0;

echo "=== CloudFlow 客服 ===\n";
echo "输入 'quit' 退出\n\n";

// 模拟用户的多轮提问
$userQuestions = [
    '你们的产品是做什么的？',
    '专业版有什么功能？',
    '可以先试用吗？',
    '好的，怎么注册试用？',
];

foreach ($userQuestions as $question) {
    echo "用户：{$question}\n";

    // 使用 with() 不可变地追加用户消息
    $messages = $messages->with(Message::ofUser($question));

    // 发送完整对话历史给 AI
    $result = $platform->invoke('gpt-4.1-mini', $messages, [
        'temperature' => 0.7,
        'max_output_tokens' => 500,
    ]);

    $answer = $result->asText();
    echo "客服：{$answer}\n";

    // 追踪 Token 用量
    $tokenUsage = $result->getMetadata()->get('token_usage');
    if (null !== $tokenUsage) {
        $totalTokens += $tokenUsage->getTotalTokens();
        echo sprintf(
            "  [本轮 Token: %d | 累计: %d]\n",
            $tokenUsage->getTotalTokens(),
            $totalTokens,
        );
    }

    echo "\n";

    // 将 AI 回复加入历史，保持上下文
    $messages = $messages->with(Message::ofAssistant($answer));
}

echo sprintf("=== 对话结束，共消耗 %d Tokens ===\n", $totalTokens);
```

---

## 替代实现方案

### DeepSeek——低成本方案

DeepSeek 提供与 OpenAI 兼容的 API，价格显著更低，适合预算敏感的场景：

```bash
composer require symfony/ai-deepseek-platform
```

```php
use Symfony\AI\Platform\Bridge\DeepSeek\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['DEEPSEEK_API_KEY']);

$messages = new MessageBag(
    Message::forSystem('你是一个友好的客服。'),
    Message::ofUser('你好，请介绍一下你们的产品'),
);

// deepseek-chat 适合常规对话，deepseek-reasoner 适合复杂推理
$result = $platform->invoke('deepseek-chat', $messages);
echo $result->asText();
```

### Ollama——本地私有化部署

如果你对数据隐私有严格要求，或者想避免 API 费用，Ollama 让你在本地运行开源模型：

```bash
composer require symfony/ai-ollama-platform
```

```php
use Symfony\AI\Platform\Bridge\Ollama\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// Ollama 的工厂参数不同：第一个参数是服务端点，API 密钥可选
$platform = PlatformFactory::create('http://localhost:11434');

$messages = new MessageBag(
    Message::forSystem('你是一个友好的客服。'),
    Message::ofUser('你好，请介绍一下你们的产品'),
);

// 使用本地部署的模型
$result = $platform->invoke('llama3.2', $messages);
echo $result->asText();
```

> **💡 提示：** Ollama 需要先在本地安装并运行（`ollama serve`），然后通过 `ollama pull llama3.2` 下载模型。中文场景推荐使用 `qwen2.5` 系列模型。

### 生产环境增强

对于生产环境，Symfony AI 还提供了两个实用的包装平台：

- **`CachePlatform`**：缓存相同请求的响应，减少重复调用的 API 费用和延迟
- **`FailoverPlatform`**：在多个平台之间自动故障转移——当主平台不可用时自动切换到备用平台

这些高级主题将在后续章节详细讲解。

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `PlatformFactory::create()` | 每个 Bridge 的工厂方法，创建完整配置的 `Platform` 实例 |
| `Contract` | 序列化层，将平台无关的消息转换为各 AI 平台的 API 格式 |
| `Message::forSystem()` | 系统提示词，定义 AI 角色和行为。支持 `Template` 对象实现动态内容 |
| `Message::ofUser()` | 用户消息，支持传入多个 `ContentInterface` 实现多模态输入 |
| `Message::ofAssistant()` | 助手消息，用于在多轮对话中填充 AI 的历史回复 |
| `MessageBag` | 消息容器，具有 UUID v7 标识符。`with()` 不可变追加，`merge()` 不可变合并，`add()` 可变追加 |
| `DeferredResult` | 延迟求值——`invoke()` 返回后不立即请求，直到调用 `asText()` 等方法 |
| `Metadata` | 结果元数据容器，`get('token_usage')` 获取 `TokenUsage` 对象 |
| `InvocationEvent` / `ResultEvent` | 请求前/后的事件，用于日志、监控和请求修改。事件对象可变，可动态修改模型和选项 |
| 查询字符串选项 | `'model?temperature=0.3'` 语法，快速设置模型参数 |
| `Template::string()` / `Template::expression()` | 系统提示词模板，前者用 `{variable}` 占位符，后者用 Symfony 表达式语言 |

## 下一步

这个例子中，对话历史是在内存中手动维护的。如果用户刷新页面或关闭浏览器，对话就丢失了。要实现持久化对话，请看 [02-multi-turn-conversation.md](./02-multi-turn-conversation.md)。
