# 基础问答聊天机器人

## 业务场景

你正在开发一个产品官网，需要在页面上嵌入一个 AI 聊天窗口，用户可以询问关于产品的问题，AI 根据预设的系统提示词来回答。

**典型应用：** 官网客服机器人、FAQ 助手、产品介绍机器人

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台（OpenAI / Anthropic / Gemini 等），发送消息并获取回复 |
| **Bridge（OpenAI）** | 本教程的主实现，通过 `PlatformFactory` 连接 OpenAI |

## 项目流程图

```
┌──────────┐      ┌───────────────┐      ┌──────────────┐      ┌──────────┐
│  用户输入  │ ──▶ │  构建 MessageBag │ ──▶ │ Platform.invoke │ ──▶ │ OpenAI API│
│ (文本问题) │      │ (System+User)  │      │  (发送请求)     │      │ (AI 推理)  │
└──────────┘      └───────────────┘      └──────────────┘      └─────┬────┘
                                                                      │
┌──────────┐      ┌───────────────┐      ┌──────────────┐            │
│  显示回复  │ ◀── │   asText() 或   │ ◀── │ DeferredResult │ ◀──────┘
│ (前端渲染) │      │   asStream()   │      │   (延迟结果)    │
└──────────┘      └───────────────┘      └──────────────┘
```

## 前置准备

### 环境要求

- PHP >= 8.2
- Composer

### 安装依赖

```bash
composer require symfony/ai-platform symfony/ai-open-ai-platform
```

> **💡 提示：** `symfony/ai-platform` 是核心包，`symfony/ai-open-ai-platform` 是 OpenAI 的 Bridge 包。Symfony AI 采用 Bridge 架构——核心代码与平台实现分离，切换 AI 平台只需替换 Bridge 包，业务代码无需修改。

### 设置 API 密钥

```bash
export OPENAI_API_KEY="sk-your-api-key-here"
```

> **🔒 安全建议：** 永远不要将 API 密钥硬编码在源代码中。在 Symfony 项目中，应使用 `.env.local` 文件或服务器环境变量来管理密钥；在 CI/CD 中使用 Secrets 管理。

---

## Step 1：创建 Platform 实例

Platform 是与 AI 服务通信的统一接口。不管你用 OpenAI、Anthropic 还是其他平台，代码结构都是一样的。

```php
<?php

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;

// 通过工厂方法创建 Platform 实例
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);
```

`PlatformFactory::create()` 会自动创建所需的 HTTP 客户端。你也可以传入自定义的 `HttpClientInterface` 实例：

```php
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create(['timeout' => 30]);
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], $httpClient);
```

> **🏭 生产建议：** 在 Symfony 项目中推荐通过 AI Bundle（`symfony/ai-bundle`）来自动配置 Platform，通过依赖注入获取实例，而不是手动调用工厂方法。

---

## Step 2：构建消息并发送

LLM 的交互基于消息列表。`MessageBag` 是消息容器，`Message` 提供静态工厂方法来创建不同角色的消息。

```php
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 构建消息：系统提示 + 用户问题
$messages = new MessageBag(
    Message::forSystem('你是一个专业的产品客服。你的任务是回答用户关于我们 SaaS 产品的问题。回答要简洁、专业、友好。如果不确定答案，请诚实地告诉用户。'),
    Message::ofUser('你们的产品支持哪些编程语言？'),
);

// 调用 AI 平台
$result = $platform->invoke('gpt-4o-mini', $messages);

// 获取文本回复
echo $result->asText();
// 输出示例："我们的产品目前支持 PHP、Python、JavaScript、Go 等主流编程语言..."
```

三种消息角色各有用途：

| 工厂方法 | 角色 | 用途 |
|---------|------|------|
| `Message::forSystem()` | 系统 | 定义 AI 的角色、行为规则和知识背景 |
| `Message::ofUser()` | 用户 | 用户发送的问题或指令 |
| `Message::ofAssistant()` | 助手 | AI 的回复（用于构建多轮对话上下文） |

> **💡 提示：** `Message::forSystem()` 还支持接收 `Template` 对象，用于构建可复用的系统提示词模板。详见 Platform 文档中的 Template 章节。

---

## Step 3：多轮对话——维护上下文

在真实场景中，用户会连续提问。每次新的提问需要把之前的对话上下文也传过去，LLM 才能理解上下文。

```php
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// === 第一轮对话 ===
$messages = new MessageBag(
    Message::forSystem('你是一个专业的 SaaS 产品客服。回答要简洁专业。'),
    Message::ofUser('你们的产品多少钱？'),
);

$result = $platform->invoke('gpt-4o-mini', $messages);
$answer1 = $result->asText();
echo "客服：{$answer1}\n";
// 输出示例："我们提供三个定价方案：基础版 ¥99/月，专业版 ¥299/月，企业版需要联系销售..."

// === 第二轮对话（带上下文）===
// 把上一轮的对话加入消息列表
$messages = new MessageBag(
    Message::forSystem('你是一个专业的 SaaS 产品客服。回答要简洁专业。'),
    Message::ofUser('你们的产品多少钱？'),
    Message::ofAssistant($answer1),
    Message::ofUser('专业版和企业版有什么区别？'),
);

$result = $platform->invoke('gpt-4o-mini', $messages);
$answer2 = $result->asText();
echo "客服：{$answer2}\n";

// === 第三轮对话（继续带上下文）===
$messages = new MessageBag(
    Message::forSystem('你是一个专业的 SaaS 产品客服。回答要简洁专业。'),
    Message::ofUser('你们的产品多少钱？'),
    Message::ofAssistant($answer1),
    Message::ofUser('专业版和企业版有什么区别？'),
    Message::ofAssistant($answer2),
    Message::ofUser('企业版可以试用吗？'),
);

$result = $platform->invoke('gpt-4o-mini', $messages);
echo "客服：" . $result->asText() . "\n";
```

> **⚠️ 注意：** 每轮对话都会将完整历史发送给 AI，这意味着 Token 消耗会随对话轮数增长。在生产环境中，务必关注对话长度并设置合理的上限（参见 Step 5 的 Token 用量追踪）。

---

## Step 4：使用流式输出提升用户体验

在网页聊天中，用户希望看到 AI 逐字打出回答，而不是等待几秒后一次性显示。这就是流式输出。

```php
$messages = new MessageBag(
    Message::forSystem('你是一个产品客服，请详细解释我们的退款政策。'),
    Message::ofUser('如果我购买后不满意，可以退款吗？'),
);

// 开启流式输出
$result = $platform->invoke('gpt-4o-mini', $messages, [
    'stream' => true,
]);

// 逐词输出（在 Web 场景中，这里可以通过 SSE 推送给前端）
foreach ($result->asStream() as $chunk) {
    echo $chunk;
    // 实际 Web 应用中：发送 Server-Sent Event 给前端
    // echo "data: {$chunk}\n\n";
    // flush();
}
echo "\n";
```

> **💡 提示：** 流式输出的底层通过 `DeferredResult` 实现延迟求值——在调用 `asStream()` 之前不会真正发起请求。流式传输过程中会触发 `StartEvent`、`ChunkEvent` 和 `CompleteEvent`，你可以通过 `EventDispatcher` 监听这些事件来实现日志记录、进度追踪等功能。

---

## Step 5：控制输出参数与追踪 Token 用量

### 模型选项

你可以通过选项数组控制 AI 的输出行为：

```php
$messages = new MessageBag(
    Message::forSystem('你是一个简洁的客服，回答不超过两句话。'),
    Message::ofUser('你们公司在哪？'),
);

$result = $platform->invoke('gpt-4o-mini', $messages, [
    'max_output_tokens' => 100,  // 限制最大输出 token 数
    'temperature' => 0.3,        // 降低随机性，让回答更稳定
]);

echo $result->asText();
```

> **💡 提示：** 参数也可以附加在模型名称的查询字符串上，适合快速测试：
> ```php
> $result = $platform->invoke('gpt-4o-mini?max_output_tokens=100&temperature=0.3', $messages);
> ```

### Token 用量追踪

每次调用都会消耗 Token，直接关系到 API 费用。你可以通过结果的元数据获取本次调用的 Token 消耗：

```php
$result = $platform->invoke('gpt-4o-mini', $messages);

echo $result->asText();

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

> **🏭 生产建议：** 建议在生产环境中记录每次请求的 Token 用量，用于成本监控和异常检测。流式输出同样支持获取 Token 用量——在 `asStream()` 迭代完成后，通过相同的 `getMetadata()` 方法获取。

---

## 完整示例：一个简单的命令行聊天机器人

把以上所有步骤整合，模拟一个真实的多轮命令行聊天机器人：

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);

$systemPrompt = '你是一个友好的 SaaS 产品客服。产品名称是 CloudFlow。'
    . '主要功能：项目管理、团队协作、自动化工作流。'
    . '定价：基础版 ¥99/月，专业版 ¥299/月，企业版联系销售。'
    . '回答要简洁、友好、准确。如果用户的问题超出产品范围，礼貌地引导回来。';

// 对话历史（手动维护）
$conversationMessages = [Message::forSystem($systemPrompt)];
$totalTokens = 0;

echo "=== CloudFlow 客服 ===\n";
echo "输入 'quit' 退出\n\n";

// 模拟用户的几轮提问
$userQuestions = [
    '你们的产品是做什么的？',
    '专业版有什么功能？',
    '可以先试用吗？',
    '好的，怎么注册试用？',
];

foreach ($userQuestions as $question) {
    echo "用户：{$question}\n";

    // 把用户消息加入历史
    $conversationMessages[] = Message::ofUser($question);

    // 发送完整对话历史给 AI
    $result = $platform->invoke('gpt-4o-mini', new MessageBag(...$conversationMessages), [
        'temperature' => 0.7,
    ]);

    $answer = $result->asText();
    echo "客服：{$answer}\n";

    // 输出 Token 用量
    $tokenUsage = $result->getMetadata()->get('token_usage');
    if (null !== $tokenUsage) {
        $totalTokens += $tokenUsage->getTotalTokens();
        echo sprintf("  [本轮 Token: %d | 累计: %d]\n", $tokenUsage->getTotalTokens(), $totalTokens);
    }

    echo "\n";

    // 把 AI 回复也加入历史，保持上下文
    $conversationMessages[] = Message::ofAssistant($answer);
}

echo sprintf("=== 对话结束，共消耗 %d Tokens ===\n", $totalTokens);
```

---

## 其他实现方案：使用 Anthropic (Claude)

Symfony AI 的 Bridge 架构让切换 AI 平台变得非常简单。下面演示如何使用 Anthropic Claude 实现同样的聊天机器人。

### 安装 Anthropic Bridge

```bash
composer require symfony/ai-anthropic-platform
```

```bash
export ANTHROPIC_API_KEY="sk-ant-your-api-key-here"
```

### 完整代码

```php
<?php

require 'vendor/autoload.php';

// 仅需更换 PlatformFactory 的引入、API 密钥和模型名称
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY']);

$messages = new MessageBag(
    Message::forSystem('你是一个友好的 SaaS 产品客服。产品名称是 CloudFlow。回答要简洁、友好、准确。'),
    Message::ofUser('你们的产品是做什么的？'),
);

// 使用 Claude 模型
$result = $platform->invoke('claude-sonnet-4-20250514', $messages);

echo $result->asText();

// Token 用量获取方式完全一致
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

> **💡 提示：** 注意对比两个方案——除了 `use` 语句、API 密钥和模型名称不同，其余代码（`MessageBag`、`invoke()`、`asText()`、`getMetadata()`）完全一致。这就是 Symfony AI Platform 抽象层的价值。

### 其他可选平台

Symfony AI 还提供以下 Bridge 包，使用方式完全相同：

| Bridge 包 | 平台 | 安装命令 |
|-----------|------|---------|
| `symfony/ai-gemini-platform` | Google Gemini | `composer require symfony/ai-gemini-platform` |
| `symfony/ai-mistral-platform` | Mistral AI | `composer require symfony/ai-mistral-platform` |
| `symfony/ai-deepseek-platform` | DeepSeek | `composer require symfony/ai-deepseek-platform` |
| `symfony/ai-ollama-platform` | Ollama（本地） | `composer require symfony/ai-ollama-platform` |

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `PlatformFactory::create()` | 创建平台连接，不同平台用不同 Bridge 的工厂类 |
| `Message::forSystem()` | 系统提示词，定义 AI 的角色和行为（支持 `Template`） |
| `Message::ofUser()` | 用户发送的消息 |
| `Message::ofAssistant()` | AI 的回复（用于构建多轮对话上下文） |
| `MessageBag` | 消息容器，包含整个对话历史 |
| `$platform->invoke()` | 调用 AI 平台，返回 `DeferredResult`（延迟求值） |
| `$result->asText()` | 获取纯文本回复 |
| `$result->asStream()` | 获取流式输出迭代器（需配合 `'stream' => true`） |
| `$result->getMetadata()` | 获取结果元数据，包含 Token 用量等信息 |
| 选项数组 | 通过 `temperature`、`max_output_tokens` 等参数控制 AI 输出行为 |

## 下一步

这个例子中，对话历史是在内存中手动维护的。如果用户刷新页面或关闭浏览器，对话就丢失了。要实现持久化对话，请看 [02-multi-turn-conversation.md](./02-multi-turn-conversation.md)。
