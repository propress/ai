# 实时流式对话应用

## 业务场景

你在做一个面向用户的 AI 聊天产品（类似 ChatGPT）。用户提问后，不想干等 10 秒才看到完整回答，而是希望看到"打字机效果"——AI 的回答逐字流式输出。同时在流式过程中，如果 AI 需要调用工具（搜索、计算等），也要无缝处理。你还需要在流式传输过程中监控 token 用量、记录日志、持久化对话历史。

**典型应用：** AI 聊天产品、在线客服、编程助手、写作辅助、实时翻译、教学问答

## 涉及模块

| 模块 | 包名 | 用途 |
|------|------|------|
| **Platform** | `symfony/ai-platform` | 统一 AI 平台接口 |
| **Platform Bridge** | `symfony/ai-open-ai-platform` | OpenAI 流式支持 |
| **Platform Bridge** | `symfony/ai-anthropic-platform` | Anthropic 流式支持 |
| **Agent** | `symfony/ai-agent` | 工具调用 + 流式处理 |
| **Chat** | `symfony/ai-chat` | 对话持久化 |
| **Chat Bridge** | `symfony/ai-chat-redis` | Redis 消息存储 |

> 💡 **提示：** 流式响应由 Platform 层的 `StreamResult` 提供，几乎所有 Bridge（OpenAI、Anthropic、Gemini、Mistral、DeepSeek、Ollama、Cerebras、Perplexity 等 14+ 平台）都支持 `stream: true` 选项。

## 架构概述

流式对话系统需要从 AI 平台获取逐 token 的响应流，通过事件监听器处理中间状态（工具调用、token 统计），再以 SSE 格式推送给前端：

```
用户输入（前端）
    │
    ▼
┌────────────────────────────────────────────────┐
│              Symfony 控制器                      │
│  StreamedResponse + SSE                        │
│                                                │
│  ┌──────────────────────────────────────────┐  │
│  │            Agent / Platform               │  │
│  │  invoke(model, messages, stream: true)    │  │
│  │                                           │  │
│  │  ┌─────────────┐  ┌──────────────────┐   │  │
│  │  │ StreamResult │  │  StreamListener  │   │  │
│  │  │  (Generator) │  │  - StartEvent    │   │  │
│  │  │              │──│  - ChunkEvent    │   │  │
│  │  │  yield chunk │  │  - CompleteEvent │   │  │
│  │  └─────────────┘  └──────────────────┘   │  │
│  └──────────────────────────────────────────┘  │
│                                                │
│  ┌──────────────────────────────────────────┐  │
│  │         MessageStore (持久化)             │  │
│  │  Redis / Doctrine DBAL / MongoDB         │  │
│  └──────────────────────────────────────────┘  │
└────────────────────────────────────────────────┘
    │
    ▼ SSE: data: {"text": "..."}\n\n
前端实时渲染（打字机效果）
```

📝 **知识扩展：** `StreamResult` 内部使用 PHP Generator（`yield`）逐块产出响应内容。在每个 chunk 到达时，注册的 `ListenerInterface` 监听器会依次收到 `StartEvent` → 多个 `ChunkEvent` → `CompleteEvent`，可以在不阻塞流的情况下做日志记录、token 统计、工具调用拦截等。

## 前置准备

### 环境要求

- PHP 8.2+
- Composer
- 至少一个 AI 平台 API 密钥

### 安装依赖

```bash
# 核心 + OpenAI（主平台）
composer require symfony/ai-platform symfony/ai-open-ai-platform

# Agent（工具调用 + 流式处理）
composer require symfony/ai-agent

# 对话持久化（可选，按需选择一种）
composer require symfony/ai-chat symfony/ai-chat-redis

# 替代平台（按需安装）
composer require symfony/ai-anthropic-platform   # Anthropic Claude
composer require symfony/ai-gemini-platform       # Google Gemini
composer require symfony/ai-mistral-platform      # Mistral AI
composer require symfony/ai-ollama-platform       # 本地 Ollama
```

### 设置 API 密钥

```bash
# .env
OPENAI_API_KEY=sk-your-openai-key
ANTHROPIC_API_KEY=sk-ant-your-anthropic-key
GEMINI_API_KEY=your-gemini-key
```

🔒 **安全建议：** 在生产环境中使用 Symfony Secrets Vault 管理 API 密钥，不要将密钥提交到版本控制。

---

## Step 1：基础流式输出（Anthropic Claude）

使用 Anthropic Claude 演示最基本的流式调用：传入 `stream: true` 选项，通过 `asStream()` 迭代接收 token。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY']);

$messages = new MessageBag(
    Message::forSystem('你是一位资深 PHP 架构师，擅长用清晰的方式讲解设计模式。'),
    Message::ofUser('用 PHP 写一个发布-订阅模式的实现，附带详细注释。'),
);

// 流式调用：stream: true 让平台以 Generator 形式逐 token 返回
$result = $platform->invoke('claude-sonnet-4-5-20250929', $messages, [
    'stream' => true,
]);

// 实时输出每个 token
echo "🤖 AI：";
foreach ($result->asStream() as $chunk) {
    echo $chunk;
}
echo "\n";
```

### 流式 vs 非流式对比

| 特性 | 非流式（默认） | 流式（`stream: true`） |
|------|---------------|----------------------|
| 响应方式 | 等待完整结果后一次返回 | 逐 token 实时返回 |
| 首字节延迟 | 高（等待全部生成） | 低（首个 token 即返回） |
| 获取结果 | `$result->asText()` | `$result->asStream()` 返回 `Generator` |
| 工具调用 | 自动处理 | 由 `StreamListener` 拦截处理 |
| 适用场景 | 后端批处理 | 面向用户的实时交互 |

💡 **提示：** `asStream()` 返回的 `Generator` 只能迭代一次。如果需要同时显示和保存完整内容，在迭代时拼接字符串即可。

---

## Step 2：自定义流事件监听器

`StreamResult` 在整个流的生命周期中触发三种事件：`StartEvent`、`ChunkEvent`、`CompleteEvent`。你可以实现 `ListenerInterface` 或继承 `AbstractStreamListener` 来监控流式过程——记录日志、统计 token、过滤敏感内容等。

```php
<?php

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\Result\Stream\AbstractStreamListener;
use Symfony\AI\Platform\Result\Stream\ChunkEvent;
use Symfony\AI\Platform\Result\Stream\CompleteEvent;
use Symfony\AI\Platform\Result\Stream\StartEvent;

/**
 * 自定义监听器：记录流的开始/结束时间和 chunk 数量。
 */
final class StreamMonitorListener extends AbstractStreamListener
{
    private float $startTime = 0.0;
    private int $chunkCount = 0;
    private int $totalChars = 0;

    public function onStart(StartEvent $event): void
    {
        $this->startTime = microtime(true);
        $this->chunkCount = 0;
        $this->totalChars = 0;
    }

    public function onChunk(ChunkEvent $event): void
    {
        ++$this->chunkCount;
        $chunk = $event->getChunk();

        if (is_string($chunk)) {
            $this->totalChars += mb_strlen($chunk);
        }

        // 可以选择跳过某些 chunk（如包含敏感词）
        // if (str_contains($chunk, '敏感词')) {
        //     $event->skipChunk();
        // }
    }

    public function onComplete(CompleteEvent $event): void
    {
        $elapsed = microtime(true) - $this->startTime;
        echo \sprintf(
            "\n📊 流统计：%d 个 chunk，%d 字符，%.2f 秒\n",
            $this->chunkCount,
            $this->totalChars,
            $elapsed,
        );
    }
}
```

在 Platform 层直接使用自定义监听器：

```php
<?php

use Symfony\AI\Platform\Result\StreamResult;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);

$messages = new MessageBag(
    Message::ofUser('简要介绍 PHP 8.4 的新特性。'),
);

$result = $platform->invoke('gpt-4o-mini', $messages, [
    'stream' => true,
]);

// 注册自定义监听器
$streamResult = $result->as(StreamResult::class);
$streamResult->addListener(new StreamMonitorListener());

// 迭代流内容——监听器会在每个事件点自动被调用
echo "🤖 ";
foreach ($streamResult->getContent() as $chunk) {
    echo $chunk;
}
```

📝 **知识扩展：** `ChunkEvent::skipChunk()` 可以在监听器中跳过特定 chunk，这对实时内容过滤（如屏蔽敏感词）非常有用。被跳过的 chunk 不会出现在 `getContent()` 的迭代结果中。

### 流事件 API 参考

| 事件类 | 触发时机 | 关键方法 |
|--------|---------|---------|
| `StartEvent` | 流开始时（第一个 chunk 之前） | 继承自 `Event`：可访问 `StreamResult` |
| `ChunkEvent` | 每个 chunk 到达时 | `getChunk()`、`setChunk()`、`skipChunk()`、`isChunkSkipped()` |
| `CompleteEvent` | 流结束时（所有 chunk 之后） | 继承自 `Event`：可访问完整元数据 |

---

## Step 3：SSE（Server-Sent Events）Web 集成

在 Symfony 控制器中使用 `StreamedResponse` 实现 SSE 流式推送。前端通过 `fetch` + `ReadableStream` 消费 SSE 事件。

```php
<?php

namespace App\Controller;

use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\PlatformInterface;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\StreamedResponse;
use Symfony\Component\Routing\Attribute\Route;

final class ChatStreamController
{
    public function __construct(
        private readonly PlatformInterface $platform,
    ) {
    }

    #[Route('/api/chat/stream', methods: ['POST'])]
    public function stream(Request $request): StreamedResponse
    {
        $userMessage = $request->getPayload()->getString('message');

        return new StreamedResponse(function () use ($userMessage): void {
            $messages = new MessageBag(
                Message::forSystem('你是一位友好的 AI 助手。'),
                Message::ofUser($userMessage),
            );

            $result = $this->platform->invoke('gpt-4o-mini', $messages, [
                'stream' => true,
            ]);

            // SSE 格式逐 chunk 推送
            foreach ($result->asStream() as $chunk) {
                echo 'data: '.json_encode(['content' => $chunk], \JSON_THROW_ON_ERROR)."\n\n";
                ob_flush();
                flush();
            }

            // 发送结束信号
            echo "data: [DONE]\n\n";
            ob_flush();
            flush();
        }, 200, [
            'Content-Type' => 'text/event-stream',
            'Cache-Control' => 'no-cache',
            'Connection' => 'keep-alive',
            'X-Accel-Buffering' => 'no',
        ]);
    }
}
```

⚠️ **注意：** `X-Accel-Buffering: no` 是 Nginx 反向代理下的必须设置，否则 Nginx 会缓冲整个响应而不是实时转发。如果使用 Apache，需要确保 `mod_proxy` 没有启用 `flushpackets` 缓冲。

### 前端 JavaScript 消费 SSE

使用 `fetch` + `ReadableStream` 方式比 `EventSource` 更灵活，支持 POST 请求和自定义请求头：

```javascript
async function streamChat(message) {
    const outputDiv = document.getElementById('ai-response');
    outputDiv.textContent = '';

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

        const text = decoder.decode(value, { stream: true });
        const lines = text.split('\n');

        for (const line of lines) {
            if (line.startsWith('data: ') && line !== 'data: [DONE]') {
                const data = JSON.parse(line.slice(6));
                outputDiv.textContent += data.content;
            }
        }
    }
}
```

💡 **提示：** `EventSource` API 仅支持 GET 请求。对于需要 POST 请求体的聊天场景，使用 `fetch` + `ReadableStream` 是更好的选择。

---

## Step 4：流式 + 工具调用

Agent 的 `AgentProcessor` 会在流式模式下自动注册 `StreamListener`，拦截工具调用 chunk、执行工具、将结果回注到对话中。对调用方来说，只需像普通流式一样迭代 `getContent()` 即可。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Attribute\AsTool;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// ---- 定义工具 ----

#[AsTool('get_weather', description: '查询城市天气', method: '__invoke')]
final class WeatherTool
{
    /**
     * @param string $city 城市名称
     */
    public function __invoke(string $city): string
    {
        // 实际项目中调用天气 API（如 OpenMeteo）
        return \sprintf('%s 当前天气：晴，25°C，湿度 60%%', $city);
    }
}

#[AsTool('search_web', description: '搜索网页获取信息', method: '__invoke')]
final class WebSearchTool
{
    /**
     * @param string $query 搜索关键词
     */
    public function __invoke(string $query): string
    {
        // 实际项目中使用 Tavily、SerpApi 等
        return \sprintf('搜索"%s"的结果：找到 3 条相关信息……', $query);
    }
}

// ---- 组装 Agent ----

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);
$toolbox = new Toolbox([new WeatherTool(), new WebSearchTool()]);
$processor = new AgentProcessor($toolbox);

$agent = new Agent(
    $platform,
    'gpt-4o-mini',
    [
        new SystemPromptInputProcessor('你是智能助手，可以查天气和搜索网页。'),
        $processor,  // 输入阶段：注册工具定义
    ],
    [$processor],    // 输出阶段：处理工具调用结果
);

// ---- 流式调用 ----

$messages = new MessageBag(
    Message::ofUser('北京今天天气如何？另外搜索一下 PHP 8.4 的新特性。'),
);

$result = $agent->call($messages, ['stream' => true]);

echo "🤖 AI：";
foreach ($result->getContent() as $chunk) {
    echo $chunk;
}
echo "\n";
```

📝 **知识扩展：** 在流式 + 工具调用场景中，`AgentProcessor` 作为 `OutputProcessorInterface` 会自动创建 `Symfony\AI\Agent\Toolbox\StreamListener` 并注册到 `StreamResult`。当 chunk 中出现 `ToolCallResult` 时，StreamListener 会暂停文本流、执行工具、将工具结果注入对话、然后重新发起一轮请求。整个过程对迭代 `getContent()` 的调用方完全透明。

### 流式工具调用流程

```
Agent.call(messages, stream: true)
    │
    ├──► Platform.invoke() → StreamResult
    │         │
    │         ├── ChunkEvent("北京今天…")  → 直接 yield 文本
    │         ├── ChunkEvent("北京今天…")  → 直接 yield 文本
    │         ├── ChunkEvent(ToolCall)     → StreamListener 拦截
    │         │       │
    │         │       ▼
    │         │   执行工具 → 获取结果
    │         │       │
    │         │       ▼
    │         │   重新调用 Platform（带工具结果）
    │         │       │
    │         │       ▼
    │         │   新的 StreamResult → 继续 yield 文本
    │         │
    │         └── CompleteEvent → 流结束
    │
    └──► 返回最终 ResultInterface
```

---

## Step 5：Token 用量追踪

在流式模式下，Platform 会自动注册 `TokenUsage\StreamListener`，将 token 用量信息添加到结果元数据中。流式迭代完成后即可读取。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\TokenUsage\TokenUsage;

$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY']);
$agent = new Agent($platform, 'claude-sonnet-4-5-20250929');

$messages = new MessageBag(
    Message::ofUser('比较 Redis 和 Memcached 的优劣。'),
);

$result = $agent->call($messages, ['stream' => true]);

// 1. 先消费完整个流
echo "🤖 AI：";
foreach ($result->getContent() as $chunk) {
    echo $chunk;
}
echo "\n\n";

// 2. 流结束后读取 token 用量
$tokenUsage = $result->getMetadata()->get('token_usage');
if ($tokenUsage instanceof TokenUsage) {
    echo \sprintf(
        "📊 Token 用量：输入 %d，输出 %d，总计 %d\n",
        $tokenUsage->inputTokens,
        $tokenUsage->outputTokens,
        $tokenUsage->totalTokens,
    );
}
```

⚠️ **注意：** 必须先完整迭代 `getContent()` 或 `asStream()` 后，元数据才会被填充。在迭代完成之前调用 `getMetadata()` 可能得到不完整的结果。

### 结合自定义监听器做实时 token 估算

```php
<?php

use Symfony\AI\Platform\Result\Stream\AbstractStreamListener;
use Symfony\AI\Platform\Result\Stream\ChunkEvent;
use Symfony\AI\Platform\Result\Stream\CompleteEvent;

/**
 * 实时估算 token 数（简易版：按字符数 / 4 估算英文 token）。
 * 实际生产中建议使用 tiktoken 等专用库。
 */
final class TokenEstimateListener extends AbstractStreamListener
{
    private int $estimatedTokens = 0;

    public function onChunk(ChunkEvent $event): void
    {
        $chunk = $event->getChunk();
        if (is_string($chunk)) {
            // 粗略估算：英文约 4 字符 = 1 token，中文约 1.5 字符 = 1 token
            $this->estimatedTokens += max(1, (int) (mb_strlen($chunk) / 2));
        }
    }

    public function onComplete(CompleteEvent $event): void
    {
        echo \sprintf("\n📏 估算输出约 %d tokens\n", $this->estimatedTokens);
    }
}
```

---

## Step 6：对话持久化（MessageStore）

`Chat` 组件提供 `MessageStoreInterface` 用于持久化对话历史。Chat 持有一个 Agent 和一个 MessageStore，通过 `initiate()` 初始化新对话、`submit()` 提交用户消息并获取 AI 回复。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\Bridge\Redis\MessageStore;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\Message\UserMessage;

// ---- 创建 MessageStore ----

$redis = new \Redis();
$redis->connect('127.0.0.1', 6379);

$messageStore = new MessageStore(
    redis: $redis,
    indexName: 'chat:conversation:user-session-001',
);
$messageStore->setup();

// ---- 创建 Chat ----

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);
$agent = new Agent($platform, 'gpt-4o-mini');

$chat = new Chat($agent, $messageStore);

// ---- 开始对话 ----

// 初始化带系统提示的对话
$chat->initiate(new MessageBag(
    Message::forSystem('你是一位 PHP 专家。简洁回答问题。'),
));

// 提交用户消息，获取 AI 回复
$reply = $chat->submit(new UserMessage('PHP 8.4 有哪些新特性？'));
echo "🤖 AI：{$reply->content}\n";

// 继续多轮对话（历史会自动保存和加载）
$reply = $chat->submit(new UserMessage('其中哪个特性对性能影响最大？'));
echo "🤖 AI：{$reply->content}\n";
```

### MessageStore 实现比较

| 实现 | 包名 | 适用场景 | 持久性 |
|------|------|---------|--------|
| `InMemory\Store` | `symfony/ai-chat` | 开发测试 | 进程内 |
| `Bridge\Redis\MessageStore` | `symfony/ai-chat-redis` | 高并发、短期对话 | Redis TTL |
| `Bridge\Doctrine\DoctrineDbalMessageStore` | `symfony/ai-chat-doctrine` | 需要永久存储 | 数据库 |
| `Bridge\MongoDb\MessageStore` | `symfony/ai-chat-mongodb` | 文档型存储 | MongoDB |
| `Bridge\Session\MessageStore` | `symfony/ai-chat-session` | Web 会话级对话 | Session |
| `Bridge\Cache\MessageStore` | `symfony/ai-chat-cache` | 通用缓存后端 | 可配 TTL |
| `Bridge\SurrealDb\MessageStore` | `symfony/ai-chat-surrealdb` | SurrealDB 后端 | SurrealDB |
| `Bridge\Meilisearch\MessageStore` | `symfony/ai-chat-meilisearch` | 可搜索的对话记录 | Meilisearch |

💡 **提示：** 在 AI Bundle（`symfony/ai-bundle`）中，MessageStore 可以通过服务配置自动注入，无需手动创建。Bundle 会根据安装的 Bridge 包自动注册对应的 Store 服务。

---

## Step 7：完整生产级 SSE 控制器

将上述所有能力——流式输出、工具调用、对话持久化、状态通知——组合成一个生产级控制器：

```php
<?php

namespace App\Controller;

use Symfony\AI\Agent\AgentInterface;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\Bridge\Redis\MessageStore;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\Message\UserMessage;
use Symfony\AI\Platform\TokenUsage\TokenUsage;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\StreamedResponse;
use Symfony\Component\Routing\Attribute\Route;

final class ProductionStreamController
{
    public function __construct(
        private readonly AgentInterface $agent,
        private readonly MessageStore $messageStore,
    ) {
    }

    #[Route('/api/chat/{conversationId}/stream', methods: ['POST'])]
    public function stream(Request $request, string $conversationId): StreamedResponse
    {
        $userMessage = $request->getPayload()->getString('message');

        return new StreamedResponse(function () use ($conversationId, $userMessage): void {
            // 1. 构建带有历史上下文的 Chat
            $store = new MessageStore(
                redis: $this->getRedis(),
                indexName: 'chat:'.$conversationId,
            );
            $chat = new Chat($this->agent, $store);

            // 2. 发送"正在思考"状态
            $this->sendEvent('status', ['state' => 'thinking']);

            try {
                // 3. 提交用户消息并获取回复
                $reply = $chat->submit(new UserMessage($userMessage));

                // 4. 发送完整回复
                $this->sendEvent('content', ['text' => $reply->content]);

                // 5. 发送完成信号
                $this->sendEvent('done', [
                    'conversationId' => $conversationId,
                ]);
            } catch (\Throwable $e) {
                $this->sendEvent('error', [
                    'message' => '生成回复时出错，请稍后重试。',
                ]);
            }
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
        echo 'data: '.json_encode($data, \JSON_THROW_ON_ERROR)."\n\n";

        if (ob_get_level() > 0) {
            ob_flush();
        }
        flush();
    }

    private function getRedis(): \Redis
    {
        $redis = new \Redis();
        $redis->connect($_ENV['REDIS_HOST'] ?? '127.0.0.1', (int) ($_ENV['REDIS_PORT'] ?? 6379));

        return $redis;
    }
}
```

### 对应前端实现

```javascript
async function streamChat(conversationId, message) {
    const outputDiv = document.getElementById('ai-response');
    const statusDiv = document.getElementById('status-indicator');

    const response = await fetch(`/api/chat/${conversationId}/stream`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ message }),
    });

    const reader = response.body.getReader();
    const decoder = new TextDecoder();
    let buffer = '';
    let currentEvent = '';

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

    function handleEvent(event, data) {
        switch (event) {
            case 'status':
                statusDiv.textContent =
                    data.state === 'thinking' ? '🤔 思考中...' : '✍️ 输入中...';
                statusDiv.style.display = 'block';
                break;
            case 'content':
                statusDiv.style.display = 'none';
                outputDiv.textContent += data.text;
                break;
            case 'error':
                statusDiv.style.display = 'none';
                outputDiv.textContent = `❌ ${data.message}`;
                break;
            case 'done':
                statusDiv.style.display = 'none';
                break;
        }
    }
}
```

🏭 **生产建议：**
- 为 SSE 连接设置合理的超时时间（如 60 秒），避免长连接占用服务器资源。
- 使用 `ob_get_level()` 检查输出缓冲区状态，避免在无缓冲区时调用 `ob_flush()` 报错。
- 在 Nginx 配置中，除了 `X-Accel-Buffering: no`，还应设置 `proxy_read_timeout` 为足够长的时间。
- 考虑使用 Mercure 替代原生 SSE，获得更好的认证和重连支持。

---

## 替代实现方案

### 使用 Gemini 流式

```bash
composer require symfony/ai-gemini-platform
```

```php
<?php

use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['GEMINI_API_KEY']);

$result = $platform->invoke('gemini-2.0-flash', new MessageBag(
    Message::ofUser('用 PHP 实现一个简单的依赖注入容器。'),
), ['stream' => true]);

foreach ($result->asStream() as $chunk) {
    echo $chunk;
}
```

💡 **提示：** Gemini 的流式响应粒度可能与 OpenAI 不同——有时一个 chunk 包含多个词。这不影响功能，但前端显示速度会有细微差异。

### 使用 Mistral 流式

```bash
composer require symfony/ai-mistral-platform
```

```php
<?php

use Symfony\AI\Platform\Bridge\Mistral\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['MISTRAL_API_KEY']);

$result = $platform->invoke('mistral-small-latest', new MessageBag(
    Message::ofUser('解释 PHP Fiber 的工作原理。'),
), ['stream' => true]);

foreach ($result->asStream() as $chunk) {
    echo $chunk;
}
```

### 使用 Ollama 本地流式（无需 API 密钥）

```bash
composer require symfony/ai-ollama-platform
```

```php
<?php

use Symfony\AI\Platform\Bridge\Ollama\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// Ollama 是本地部署，不需要 API 密钥
$platform = PlatformFactory::create('http://localhost:11434');

$result = $platform->invoke('llama3.1', new MessageBag(
    Message::ofUser('什么是 SOLID 原则？'),
), ['stream' => true]);

foreach ($result->asStream() as $chunk) {
    echo $chunk;
}
```

💡 **提示：** Ollama 适合开发环境和对数据隐私要求高的场景。本地部署延迟通常比云端更低，流式体验更流畅。

### 使用 DeepSeek 流式

```bash
composer require symfony/ai-deep-seek-platform
```

```php
<?php

use Symfony\AI\Platform\Bridge\DeepSeek\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['DEEPSEEK_API_KEY']);

$result = $platform->invoke('deepseek-chat', new MessageBag(
    Message::ofUser('比较 PHP 和 Go 在 Web 开发中的优劣。'),
), ['stream' => true]);

foreach ($result->asStream() as $chunk) {
    echo $chunk;
}
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `stream: true` | 传入 `invoke()` 的选项，启用逐 token 流式响应 |
| `$result->asStream()` | 获取 `Generator`，迭代每个文本 chunk |
| `$result->getContent()` | Agent 返回的可迭代内容，流式时等价于 `asStream()` |
| `StreamResult` | 流式结果包装类，持有 Generator 和 Listener 列表 |
| `ListenerInterface` | 流事件监听接口：`onStart`、`onChunk`、`onComplete` |
| `AbstractStreamListener` | 监听器基类，所有方法有空默认实现 |
| `ChunkEvent` | chunk 事件，支持 `skipChunk()` 过滤和 `setChunk()` 修改 |
| `AgentProcessor` | Agent 工具处理器，流式模式下自动注册 `StreamListener` |
| `StreamedResponse` | Symfony HTTP 流式响应，用于 SSE 推送 |
| SSE 格式 | `event: name\ndata: {...}\n\n`，标准浏览器事件流协议 |
| `X-Accel-Buffering: no` | Nginx 必须禁用缓冲，否则 SSE 不工作 |
| `TokenUsage` | 流完成后通过 `getMetadata()->get('token_usage')` 获取 |
| `Chat` | 对话管理器：`initiate()` 新建、`submit()` 提交消息 |
| `MessageStore` | 对话存储接口，10+ 后端实现（Redis、DBAL、MongoDB 等） |

## 下一步

如果你想让 AI 帮你做会议纪要（语音转文字 + 结构化提取），请看 [34-meeting-minutes-assistant.md](./34-meeting-minutes-assistant.md)。
