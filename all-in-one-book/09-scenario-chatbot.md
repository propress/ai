# 第 9 章：实战 —— 基础问答聊天机器人

## 学习目标

通过本章实战，你将掌握：
- 使用 Platform 组件构建基础问答聊天机器人
- 理解 MessageBag、DeferredResult 等核心概念的实际用法
- 实现流式响应提升用户体验
- 掌握模型选项调优和平台切换技巧
- 建立完善的错误处理和事件监控机制

## 前置知识

<!-- 建议在正式学习前回顾以下章节 -->
- Platform 组件基础用法
- MessageBag 消息构建
- Bridge 架构与平台切换

## 业务场景描述

在产品官网嵌入一个 AI 聊天窗口，用户提问，AI 根据预设的系统提示词给出回答。

**典型应用**：官网客服机器人、FAQ 助手、产品介绍机器人、内部知识库问答。

## 架构概述

```php
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

> **知识扩展**：流程中的 `Contract` 是一个常被忽略但至关重要的组件。它基于 Symfony Serializer，负责将 `Message`、`ToolCall` 等对象序列化为各平台特定的 JSON 格式。每个 Bridge 都有自己的 Contract 实现（如 `AnthropicContract`、`OpenAiContract`），你无需直接操作它——`PlatformFactory` 会自动创建并注入。

## 环境准备

```bash
composer require symfony/ai-platform symfony/ai-anthropic-platform
# 或使用 OpenAI：
# composer require symfony/ai-platform symfony/ai-open-ai-platform
```

## 核心实现

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

### 流式响应

对于 Web 应用，流式输出能大幅提升用户体验——首字节通常在 200ms 内返回，而非等待完整回复的 3-10 秒：

```php
$response = $platform->invoke($model, $messages);

// 流式输出——逐块获取文本
foreach ($response->asStream() as $chunk) {
    echo $chunk;
    flush();
}
```

> **知识扩展：流式响应的内部机制**
>
> `asStream()` 返回一个 PHP `Generator`，内部通过 HTTP 分块传输（chunked transfer）逐步接收 AI 平台的响应。每收到一个 token（或几个 token），Generator 就 `yield` 一次。这意味着：
> - Generator 只能迭代**一次**——如果需要重复使用，先 `asText()` 保存完整内容
> - 流式模式下 `getMetadata()` 可能不完整（Token 计数在流结束后才确定）
> - 流式模式与结构化输出（`output`）不兼容——结构化输出需要完整 JSON 后才能解析

### 模型选项

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

### 切换平台

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

## 运行与验证

<!-- 运行示例并验证输出的步骤 -->

## 错误处理

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
    $retryAfter = $e->getRetryAfter();  // 可选：建议等待秒数
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

> **知识扩展：Platform 异常体系**
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

### 使用事件监控

```php
use Symfony\AI\Platform\Event\InvocationEvent;
use Symfony\AI\Platform\Event\ResultEvent;

// 在调用前拦截——可以修改模型、输入或选项
$dispatcher->addListener(InvocationEvent::class, function (InvocationEvent $event) {
    $this->logger->info('AI 调用开始', [
        'model' => $event->getModel()->getName(),
    ]);
});

// 在结果返回后——记录 Token 使用量和耗时
$dispatcher->addListener(ResultEvent::class, function (ResultEvent $event) {
    $metadata = $event->getDeferredResult()->getMetadata();
    $tokenUsage = $metadata->get('token_usage');
    $this->logger->info('AI 调用完成', [
        'model' => $event->getModel()->getName(),
        'prompt_tokens' => $tokenUsage?->getPromptTokens(),
        'completion_tokens' => $tokenUsage?->getCompletionTokens(),
    ]);
});
```

## 生产环境注意事项

<!-- 部署到生产环境时的配置和优化建议 -->

## 扩展方向

<!-- 基于本场景的进一步扩展思路 -->

## 完整源代码

<!-- 完整可运行的源代码汇总 -->

## 下一步

下一章我们将学习 [第 10 章：实战 —— 多轮对话与会话持久化](10-scenario-multi-turn-chat.md)，使用 Chat 组件实现带上下文记忆的客服聊天系统。
