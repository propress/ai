# 实时流式对话应用

## 业务场景

你在做一个面向用户的 AI 聊天产品（类似 ChatGPT）。用户提问后，不想干等 10 秒钟才看到完整回答，而是希望看到"打字机效果"——AI 的回答一个字一个字地流式输出。同时在流式过程中，如果 AI 需要调用工具（搜索、计算），也要无缝处理。

**典型应用：** AI 聊天产品、在线客服、编程助手、写作辅助、实时翻译

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台（流式模式） |
| **Chat** | 对话持久化 |
| **Agent** | 工具调用 + 流式处理 |
| **Agent StreamListener** | 流式响应中处理工具调用 |

## 前置准备

```bash
composer require symfony/ai-platform symfony/ai-platform-openai
composer require symfony/ai-agent
composer require symfony/ai-chat
```

---

## Step 1：基础流式输出

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());

$messages = new MessageBag(
    Message::ofUser('用 PHP 写一个简单的发布-订阅模式的实现，带注释。'),
);

// 流式调用
$result = $platform->invoke('gpt-4o-mini', $messages, [
    'stream' => true,
]);

// 实时输出每个 token
echo "🤖 AI：";
foreach ($result->asStream() as $chunk) {
    echo $chunk;
    // 在 Web 应用中，这里可以用 SSE/WebSocket 推送给前端
}
echo "\n";
```

---

## Step 2：SSE（Server-Sent Events）Web 集成

在 Symfony 控制器中实现 SSE 流式推送。

```php
<?php

namespace App\Controller;

use Symfony\AI\Platform\PlatformInterface;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\StreamedResponse;
use Symfony\Component\Routing\Attribute\Route;

final class ChatStreamController
{
    public function __construct(
        private PlatformInterface $platform,
    ) {
    }

    #[Route('/api/chat/stream', methods: ['POST'])]
    public function stream(Request $request): StreamedResponse
    {
        $userMessage = $request->getPayload()->get('message', '');

        return new StreamedResponse(function () use ($userMessage) {
            $messages = new MessageBag(
                Message::ofUser($userMessage),
            );

            $result = $this->platform->invoke('gpt-4o-mini', $messages, [
                'stream' => true,
            ]);

            // SSE 格式输出
            foreach ($result->asStream() as $chunk) {
                echo "data: " . json_encode(['content' => $chunk]) . "\n\n";
                ob_flush();
                flush();
            }

            // 结束信号
            echo "data: [DONE]\n\n";
            ob_flush();
            flush();
        }, 200, [
            'Content-Type' => 'text/event-stream',
            'Cache-Control' => 'no-cache',
            'Connection' => 'keep-alive',
            'X-Accel-Buffering' => 'no',  // Nginx 禁用缓冲
        ]);
    }
}
```

### 前端 JavaScript 消费 SSE：

```javascript
// 前端代码
const eventSource = new EventSource('/api/chat/stream', {
    method: 'POST',
    body: JSON.stringify({ message: '什么是依赖注入？' }),
});

const outputDiv = document.getElementById('ai-response');

// 使用 fetch + ReadableStream 更灵活
async function streamChat(message) {
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
        const lines = text.split('\n');

        for (const line of lines) {
            if (line.startsWith('data: ') && line !== 'data: [DONE]') {
                const data = JSON.parse(line.slice(6));
                outputDiv.textContent += data.content;  // 打字机效果
            }
        }
    }
}
```

---

## Step 3：流式 + 工具调用

Agent 在流式响应过程中，如果遇到工具调用，StreamListener 会自动处理。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Attribute\AsTool;
use Symfony\AI\Agent\Toolbox\Toolbox;

#[AsTool('calculate', description: '计算数学表达式', method: '__invoke')]
final class CalculatorTool
{
    public function __invoke(string $expression): string
    {
        // 安全的数学计算
        $result = eval("return {$expression};");

        return "计算结果：{$expression} = {$result}";
    }
}

#[AsTool('get_current_time', description: '获取当前时间', method: '__invoke')]
final class TimeTool
{
    public function __invoke(): string
    {
        return '当前时间：' . date('Y-m-d H:i:s');
    }
}

$toolbox = new Toolbox([new CalculatorTool(), new TimeTool()]);
$processor = new AgentProcessor($toolbox);

$agent = new Agent(
    $platform, 'gpt-4o-mini',
    [
        new SystemPromptInputProcessor('你是有用的助手。可以计算数学表达式和查看当前时间。'),
        $processor,
    ],
    [$processor],
);

// 流式 + 工具调用
$result = $agent->call(new MessageBag(
    Message::ofUser('现在几点了？另外帮我算一下 1234 * 5678 等于多少。'),
), [
    'stream' => true,
]);

// StreamListener 自动在流中处理工具调用
echo "🤖 AI：";
foreach ($result->asStream() as $chunk) {
    echo $chunk;
}
echo "\n";
```

---

## Step 4：流式 + 对话持久化

```php
<?php

use Symfony\AI\Chat\ChatInterface;

/** @var ChatInterface $chat */

$conversationId = 'user-session-123';

// 加载历史对话
$messages = $chat->loadOrCreate($conversationId, new MessageBag());
$messages->add(Message::ofUser('用 PHP 实现一个简单的 LRU 缓存'));

// 流式生成
$result = $agent->call($messages, ['stream' => true]);

// 流式输出
echo "🤖 ";
$fullResponse = '';
foreach ($result->asStream() as $chunk) {
    echo $chunk;
    $fullResponse .= $chunk;
}
echo "\n";

// 保存完整对话（流式结束后）
$chat->save($conversationId, $result->getMessages());
echo "💾 对话已保存\n";
```

---

## Step 5：带打字指示器的完整 SSE 控制器

```php
<?php

namespace App\Controller;

use Symfony\AI\Agent\AgentInterface;
use Symfony\AI\Chat\ChatInterface;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\StreamedResponse;
use Symfony\Component\Routing\Attribute\Route;

final class FullStreamController
{
    public function __construct(
        private AgentInterface $agent,
        private ChatInterface $chat,
    ) {
    }

    #[Route('/api/chat/{conversationId}/stream', methods: ['POST'])]
    public function stream(Request $request, string $conversationId): StreamedResponse
    {
        $userMessage = $request->getPayload()->getString('message');

        return new StreamedResponse(function () use ($conversationId, $userMessage) {
            // 加载历史
            $messages = $this->chat->loadOrCreate($conversationId, new MessageBag());
            $messages->add(Message::ofUser($userMessage));

            // 发送"正在思考"状态
            $this->sendEvent('status', ['state' => 'thinking']);

            // 流式生成
            $result = $this->agent->call($messages, ['stream' => true]);

            // 发送"正在输入"状态
            $this->sendEvent('status', ['state' => 'typing']);

            // 流式输出内容
            foreach ($result->asStream() as $chunk) {
                $this->sendEvent('content', ['text' => $chunk]);
            }

            // 保存对话
            $this->chat->save($conversationId, $result->getMessages());

            // 发送完成信号
            $this->sendEvent('done', ['conversationId' => $conversationId]);
        }, 200, [
            'Content-Type' => 'text/event-stream',
            'Cache-Control' => 'no-cache',
            'Connection' => 'keep-alive',
            'X-Accel-Buffering' => 'no',
        ]);
    }

    /**
     * @param array<string, mixed> $data
     */
    private function sendEvent(string $event, array $data): void
    {
        echo "event: {$event}\n";
        echo 'data: ' . json_encode($data) . "\n\n";
        ob_flush();
        flush();
    }
}
```

### 对应前端：

```javascript
async function streamChat(conversationId, message) {
    const response = await fetch(`/api/chat/${conversationId}/stream`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ message }),
    });

    const reader = response.body.getReader();
    const decoder = new TextDecoder();
    let buffer = '';

    while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        buffer += decoder.decode(value, { stream: true });
        const lines = buffer.split('\n');
        buffer = lines.pop(); // 保留不完整的行

        for (const line of lines) {
            if (line.startsWith('event: ')) {
                currentEvent = line.slice(7);
            } else if (line.startsWith('data: ')) {
                const data = JSON.parse(line.slice(6));
                handleEvent(currentEvent, data);
            }
        }
    }
}

function handleEvent(event, data) {
    switch (event) {
        case 'status':
            showIndicator(data.state); // "思考中..." / "输入中..."
            break;
        case 'content':
            appendText(data.text);     // 打字机效果
            break;
        case 'done':
            hideIndicator();           // 隐藏指示器
            break;
    }
}
```

---

## 完整流程

```
用户输入
    │
    ▼
[加载对话历史] → Chat
    │
    ▼
[Agent 处理] → stream: true
    │
    ├──► 工具调用 → StreamListener 自动处理
    │
    ├──► 逐 token 输出 → SSE 推送
    │         │
    │         ▼
    │    前端实时渲染（打字机效果）
    │
    └──► 完成后保存对话 → Chat
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `stream: true` | 开启流式响应 |
| `$result->asStream()` | 获取 token 流迭代器 |
| `StreamedResponse` | Symfony 流式 HTTP 响应 |
| SSE 格式 | `data: {...}\n\n` 事件流格式 |
| `StreamListener` | 流式中自动处理工具调用 |
| 对话持久化 | 流式结束后保存完整消息 |
| `X-Accel-Buffering: no` | Nginx 禁用缓冲（SSE 必须） |

## 下一步

如果你想让 AI 帮你做会议纪要（语音转文字 + 结构化提取），请看 [34-meeting-minutes-assistant.md](./34-meeting-minutes-assistant.md)。
