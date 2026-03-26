# 第 14 章：实战 —— 实时流式对话

## 学习目标

通过本章实战，你将掌握：
- 使用 SSE（Server-Sent Events）构建实时流式对话
- 实现 Symfony StreamedResponse 控制器
- 编写前端 JavaScript 消费 SSE 流
- 结合 Chat 组件实现多轮流式对话
- 配置 Nginx 和 PHP-FPM 以支持流式传输

## 前置知识

<!-- 建议在正式学习前回顾以下章节 -->
- Platform 组件的 asStream() 用法
- Symfony StreamedResponse
- Server-Sent Events（SSE）协议基础
- Chat 组件（用于多轮流式对话）

## 业务场景描述

构建一个 ChatGPT 式的实时流式对话 Web 应用，使用 Server-Sent Events（SSE）将 AI 回复逐字推送到前端。

**典型应用**：ChatGPT 式交互界面、实时客服、在线教育、代码辅助。

## 架构概述

```text
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

## 环境准备

<!-- 安装所需依赖 -->

```bash
composer require symfony/ai-platform symfony/ai-open-ai-platform
# 多轮流式对话还需要：
# composer require symfony/ai-chat symfony/ai-redis-message-store
```

## 核心实现

### Symfony 控制器

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

                $response = $this->platform->invoke('gpt-4o', $messages, [
                    'stream' => true,
                ]);

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

### 前端 JavaScript（使用 Fetch API）

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

### 结合 Chat 组件实现多轮流式对话

> **注意**：`Chat::submit()` 返回的是已解析的 `AssistantMessage` 对象，不是 `DeferredResult`，因此不支持 `asStream()` 流式输出。
> 如果需要多轮对话 + 流式输出，需要自行管理消息历史（使用 `MessageStoreInterface`）并直接调用 `Platform::invoke()` 配合 `'stream' => true`：

```php
class MultiRoundStreamController
{
    public function __construct(
        private readonly PlatformInterface $platform,
        private readonly MessageStoreInterface&ManagedStoreInterface $store,
    ) {}

    #[Route('/api/chat/stream', methods: ['POST'])]
    public function stream(Request $request): StreamedResponse
    {
        $userMessage = $request->getPayload()->getString('message');

        return new StreamedResponse(function () use ($userMessage) {
            header('Content-Type: text/event-stream');
            header('Cache-Control: no-cache');

            // 1. 加载历史消息
            $messages = $this->store->load();
            $messages->add(Message::ofUser($userMessage));

            // 2. 直接用 Platform 调用，启用流式
            $response = $this->platform->invoke('gpt-4o', $messages, [
                'stream' => true,
            ]);

            // 3. 流式输出
            $fullContent = '';
            foreach ($response->asStream() as $chunk) {
                $fullContent .= $chunk;

                echo 'data: '.json_encode(
                    ['text' => $chunk],
                    JSON_UNESCAPED_UNICODE,
                )."\n\n";

                if (ob_get_level() > 0) {
                    ob_flush();
                }
                flush();
            }

            // 4. 流结束后，手动保存完整响应到历史
            $messages->add(Message::ofAssistant($fullContent));
            $this->store->save($messages);

            echo "data: [DONE]\n\n";
            flush();
        });
    }
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

## 扩展方向

<!-- 基于本场景的进一步扩展思路 -->

## 完整源代码

<!-- 完整可运行的源代码汇总 -->

## 下一步

恭喜你完成了全部 6 个基础实战场景！接下来可以探索更高级的 Agent 工具调用和 Store 向量检索场景。
