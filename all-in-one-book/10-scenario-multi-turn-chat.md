# 第 10 章：实战 —— 多轮对话与会话持久化

## 学习目标

通过本章实战，你将掌握：
- 使用 Chat 组件构建多轮对话系统
- 理解会话持久化的架构和存储后端选择
- 在 Symfony 控制器中集成 Chat 组件
- 管理上下文长度，避免超出模型窗口限制

## 前置知识

<!-- 建议在正式学习前回顾以下章节 -->
- Platform 组件基础用法
- Chat 组件的 initiate/submit 生命周期
- MessageStoreInterface 与 ManagedStoreInterface

## 业务场景描述

构建一个能记住上下文的客服聊天系统。用户可以进行多轮对话，系统记住之前的交流内容。传统聊天机器人每次请求都是独立的——Chat 组件解决了这个问题。

**典型应用**：在线客服、技术支持、咨询服务、智能导购。

## 架构概述

```php
                                 Chat 组件的内部流程
                                 ═══════════════════

  initiate(MessageBag)                      submit(UserMessage)
  ┌──────────────────┐                      ┌──────────────────────────────┐
  │ 1. 清空现有消息历史                       │ 1. 从 Store 加载消息历史       │
  │    (store->drop())                      │ 2. 追加 UserMessage           │
  │ 2. 保存初始消息                           │ 3. 调用 Agent::call()         │
  │    到 Store                              │ 4. 追加 AssistantMessage      │
  │ 3. 返回 void                             │ 5. 保存到 Store               │
  └──────────────────┘                      │ 6. 返回 AssistantMessage      │
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

## 环境准备

```bash
composer require symfony/ai-platform symfony/ai-anthropic-platform \
    symfony/ai-chat symfony/ai-redis-message-store
```

## 核心实现

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
$store = new MessageStore(new \Redis(/* 连接配置 */), 'chat_session');
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

> **知识扩展：Chat 的交叉类型约束**
>
> `Chat::__construct()` 对 store 参数使用了 PHP 交叉类型（intersection type）：
> ```php
> public function __construct(
> private readonly AgentInterface $agent,
> private readonly MessageStoreInterface&ManagedStoreInterface $store,
> )
> ```
> 这意味着 store 必须**同时**实现两个接口：`MessageStoreInterface`（读写消息）和 `ManagedStoreInterface`（管理会话生命周期）。所有内置的 Chat Bridge（Redis、Doctrine、MongoDB 等）都满足此要求。
> 同时注意 Chat 接收的是 `AgentInterface`，而非直接的 `PlatformInterface`——这使得 Chat 可以利用 Agent 的完整能力（工具调用、处理器管线等）。

### 消息存储后端完整对比

| 后端 | Composer 包 | 持久化 | 性能 | 搜索 | 适用场景 |
|------|------------|--------|------|------|---------|
| **InMemory** | 内置 | ❌ | 极快 | ❌ | 单元测试 |
| **Redis** | `symfony/ai-redis-message-store` | ✅ | 快 | ❌ | 高并发生产环境 |
| **Doctrine DBAL** | `symfony/ai-doctrine-message-store` | ✅ | 中 | SQL | 已有关系数据库的项目 |
| **MongoDB** | `symfony/ai-mongo-db-message-store` | ✅ | 快 | ✅ | 文档型数据项目 |
| **Session** | 内置 | 会话 | 快 | ❌ | 简单 Web 应用 |
| **Cache** | 内置 | TTL | 快 | ❌ | 短期对话 |
| **Cloudflare** | `symfony/ai-cloudflare-message-store` | ✅ | CDN | ❌ | 边缘部署 |
| **Meilisearch** | `symfony/ai-meilisearch-message-store` | ✅ | 快 | ✅ | 需要搜索对话历史 |
| **SurrealDB** | `symfony/ai-surreal-db-message-store` | ✅ | 快 | ✅ | SurrealDB 项目 |

### 在 Symfony 控制器中使用

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
        // initiate() 清空历史并保存初始消息，返回 void
        $this->chat->initiate(new MessageBag(
            Message::forSystem('你是专业客服，回答简洁有礼貌。'),
        ));

        return new JsonResponse(['status' => 'ok']);
    }

    #[Route('/api/chat/message', methods: ['POST'])]
    public function sendMessage(Request $request): JsonResponse
    {
        $message = $request->getPayload()->getString('message');

        try {
            // submit() 接收 UserMessage，返回 AssistantMessage
            $response = $this->chat->submit(Message::ofUser($message));

            return new JsonResponse([
                'reply' => $response->getContent(),
                'metadata' => $response->getMetadata()->all(),
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

### 上下文长度管理

多轮对话的消息历史会持续增长，最终超过模型的上下文窗口。你需要制定截断策略：

```php
// 策略 1：限制保留的消息轮数
// 在自定义 Chat 子类中或通过事件处理器实现
$maxRounds = 20;  // 保留最近 20 轮对话

// 策略 2：使用摘要压缩历史
// 当历史超过阈值时，让 AI 生成摘要替换旧消息
$summary = $platform->invoke($model, new MessageBag(
    Message::forSystem('请用一段话总结以下对话的关键信息。'),
    Message::ofUser($oldHistoryText),
))->asText();

// 然后用摘要替换原始历史
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

下一章我们将学习 [第 11 章：实战 —— 结构化数据提取](11-scenario-structured-extraction.md)，使用 StructuredOutput 从非结构化文本中提取符合预定义 Schema 的 PHP 对象。
