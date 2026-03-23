# 基础问答聊天机器人

## 业务场景

你正在开发一个产品官网，需要在页面上嵌入一个 AI 聊天窗口，用户可以询问关于产品的问题，AI 根据预设的系统提示词来回答。

**典型应用：** 官网客服机器人、FAQ 助手、产品介绍机器人

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台（OpenAI / Anthropic / Gemini 等），发送消息并获取回复 |

## 前置准备

```bash
composer require symfony/ai-platform
# 根据你选择的 AI 平台安装对应的 Bridge
composer require symfony/ai-platform-openai    # OpenAI
# 或
composer require symfony/ai-platform-anthropic  # Anthropic
# 或
composer require symfony/ai-platform-gemini     # Gemini
```

设置 API 密钥（以 OpenAI 为例）：

```bash
export OPENAI_API_KEY="sk-your-api-key-here"
```

---

## Step 1：创建 Platform 实例

Platform 是与 AI 服务通信的统一接口。不管你用 OpenAI、Anthropic 还是其他平台，代码结构都是一样的。

```php
<?php

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\Component\HttpClient\HttpClient;

// 创建 HTTP 客户端
$httpClient = HttpClient::create();

// 通过工厂方法创建 Platform 实例
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], $httpClient);
```

> **切换平台：** 只需换用不同的 `PlatformFactory`，后续代码完全不用改。
> ```php
> // Anthropic
> use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
> $platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY'], $httpClient);
>
> // Gemini
> use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
> $platform = PlatformFactory::create($_ENV['GEMINI_API_KEY'], $httpClient);
> ```

---

## Step 2：构建消息并发送

LLM 的交互基于消息列表。`MessageBag` 是消息容器，`Message` 用于创建不同角色的消息。

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

---

## Step 3：模拟真实用户交互流程

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

---

## Step 5：控制输出参数

你可以通过选项控制 AI 的输出行为。

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

> **参数也可以附加在模型名上：**
> ```php
> $result = $platform->invoke('gpt-4o-mini?max_output_tokens=100&temperature=0.3', $messages);
> ```

---

## 完整示例：一个简单的命令行聊天机器人

把以上所有步骤整合，模拟一个真实的多轮命令行聊天机器人：

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());

$systemPrompt = '你是一个友好的 SaaS 产品客服。产品名称是 CloudFlow。'
    . '主要功能：项目管理、团队协作、自动化工作流。'
    . '定价：基础版 ¥99/月，专业版 ¥299/月，企业版联系销售。'
    . '回答要简洁、友好、准确。如果用户的问题超出产品范围，礼貌地引导回来。';

// 对话历史（手动维护）
$conversationMessages = [Message::forSystem($systemPrompt)];

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
    echo "客服：{$answer}\n\n";

    // 把 AI 回复也加入历史，保持上下文
    $conversationMessages[] = Message::ofAssistant($answer);
}
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `PlatformFactory::create()` | 创建平台连接，不同平台用不同的工厂类 |
| `Message::forSystem()` | 系统提示词，定义 AI 的角色和行为 |
| `Message::ofUser()` | 用户发送的消息 |
| `Message::ofAssistant()` | AI 的回复（用于构建多轮对话上下文） |
| `MessageBag` | 消息容器，包含整个对话历史 |
| `$platform->invoke()` | 调用 AI 平台，返回结果 |
| `$result->asText()` | 获取纯文本回复 |
| `$result->asStream()` | 获取流式输出迭代器 |

## 下一步

这个例子中，对话历史是在内存中手动维护的。如果用户刷新页面或关闭浏览器，对话就丢失了。要实现持久化对话，请看 [02-multi-turn-conversation.md](./02-multi-turn-conversation.md)。
