# 第 24 章：实战 —— 端到端企业知识库系统

## 学习目标

- 理解企业知识库系统的整体架构设计
- 掌握文档索引管线（Loader → Transformer → Vectorizer → Store）
- 学会使用 Chat 组件实现多轮对话
- 了解流式输出在知识库问答中的实现方式
- 掌握生产部署的完整检查清单

## 前置知识

- 熟悉 Symfony AI 的 Platform、Agent、Store、Chat 组件
- 了解 CachePlatform 和 FailoverPlatform 的组合使用（参见 [第 20 章](20-scenario-high-availability.md)）
- 了解 PostgreSQL + pgvector 扩展
- 了解 Redis 的基本使用（缓存 + 对话历史存储）

## 业务场景描述

构建一个完整的企业知识库系统，整合几乎所有 Symfony AI 组件：文档索引管线、向量检索、多轮对话、工具调用、缓存和持久化。这是一个综合性的架构设计场景。

**典型应用**：企业知识管理、内部 wiki 智能问答、客户自助服务。

## 架构概述

```text
端到端企业知识库系统架构
══════════════════════

  ┌─────────────────────────────────────────────────────────────┐
  │                     企业知识库系统                             │
  │                                                             │
  │  ┌──────────────────┐    ┌──────────────┐    ┌───────────┐ │
  │  │  文档索引管线       │    │  多轮对话引擎  │    │  AI Agent  │ │
  │  │  ──────────────   │    │  ──────────── │    │  ────────  │ │
  │  │  MarkdownLoader   │    │  Chat 组件     │    │  知识检索   │ │
  │  │  CsvLoader        │    │  Redis Store  │    │  工具      │ │
  │  │  TextSplitTransf. │    │              │    │            │ │
  │  │  TextTrimTransf.  │    │  initiate()  │    │  Toolbox:  │ │
  │  │  Vectorizer       │    │  submit()    │    │  - search  │ │
  │  └────────┬─────────┘    └──────┬───────┘    │  - clock   │ │
  │           │                     │            └─────┬─────┘ │
  │           │                     │                  │       │
  │           └─────────────────────┼──────────────────┘       │
  │                                 │                          │
  │                        ┌────────▼────────┐                 │
  │                        │  平台层            │                 │
  │                        │  ────────────     │                 │
  │                        │  CachePlatform   │                 │
  │                        │  └─ FailoverPlatform              │
  │                        │     ├─ OpenAI    │                 │
  │                        │     └─ Anthropic │                 │
  │                        └─────────────────┘                 │
  │                                                             │
  │  ┌──────────────────┐    ┌──────────────┐                  │
  │  │  PostgreSQL       │    │  Redis        │                  │
  │  │  (pgvector)       │    │  ──────────   │                  │
  │  │  ──────────────   │    │  对话历史     │                  │
  │  │  文档向量存储      │    │  响应缓存     │                  │
  │  └──────────────────┘    └──────────────┘                  │
  └─────────────────────────────────────────────────────────────┘
```

## 环境准备

### 完整 AI Bundle 配置

```yaml
# config/packages/ai.yaml
ai:
    platform:
        openai:
            api_key: '%env(OPENAI_API_KEY)%'
        anthropic:
            api_key: '%env(ANTHROPIC_API_KEY)%'
        failover:
            main:
                platforms: [ai.platform.openai, ai.platform.anthropic]
                rate_limiter: limiter.failover
        cache:
            cached:
                platform: ai.platform.failover.main
                service: cache.ai

    agent:
        knowledge_assistant:
            platform: ai.platform.cache.cached
            model: gpt-4o
            prompt: |
                你是企业知识库助手。

                行为准则：
                1. 使用 similarity_search 工具检索相关文档后回答问题
                2. 每个回答都引用来源文档
                3. 如果找不到相关信息，明确告知用户
                4. 不要编造信息——只基于检索到的文档回答
                5. 用中文回答，格式清晰
```

需要的基础设施：

```bash
# PostgreSQL with pgvector
CREATE EXTENSION vector;

# Redis（用于对话历史和响应缓存）
redis-server
```

## 核心实现

### 文档索引服务

```php
<?php

namespace App\Service;

use Symfony\AI\Store\Document\Loader\MarkdownLoader;
use Symfony\AI\Store\Document\Loader\CsvLoader;
use Symfony\AI\Store\Document\Transformer\ChainTransformer;
use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;
use Symfony\AI\Store\Document\Transformer\TextTrimTransformer;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\StoreInterface;
use Psr\Log\LoggerInterface;

class DocumentIndexService
{
    public function __construct(
        private readonly StoreInterface $store,
        private readonly Vectorizer $vectorizer,
        private readonly LoggerInterface $logger,
    ) {}

    /**
     * 索引指定目录下的所有文档
     */
    public function indexDirectory(string $directory): int
    {
        $this->logger->info('开始索引文档', ['directory' => $directory]);

        // 加载文档
        $loader = new MarkdownLoader();
        $documents = $loader->load($directory);

        // 转换管线
        $transformer = new ChainTransformer([
            new TextTrimTransformer(),
            new TextSplitTransformer(
                chunkSize: 500,
                overlap: 50,
            ),
        ]);
        $chunks = $transformer->transform($documents);

        // 向量化
        $this->vectorizer->vectorize($chunks);

        // 存储
        $this->store->add($chunks);

        $this->logger->info('文档索引完成', [
            'documents' => count($documents),
            'chunks' => count($chunks),
        ]);

        return count($chunks);
    }
}
```

### 对话控制器

```php
<?php

namespace App\Controller;

use Symfony\AI\Agent\AgentInterface;
use Symfony\AI\Chat\ChatInterface;
use Symfony\AI\Platform\Message\Message;
use Symfony\Component\DependencyInjection\Attribute\Target;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\StreamedResponse;
use Symfony\Component\Routing\Attribute\Route;

#[Route('/api/kb')]
class KnowledgeBaseController
{
    public function __construct(
        #[Target('knowledge_assistant')]
        private readonly AgentInterface $agent,
        private readonly ChatInterface $chat,
        private readonly PlatformInterface $platform,
        private readonly MessageStoreInterface&ManagedStoreInterface $store,
    ) {}

    /**
     * 创建新对话
     */
    #[Route('/conversations', methods: ['POST'])]
    public function newConversation(): JsonResponse
    {
        // initiate() 清空历史并保存初始消息，返回 void
        $this->chat->initiate(new MessageBag(
            Message::forSystem('你是企业知识库助手。'),
        ));

        return new JsonResponse(['status' => 'ok']);
    }

    /**
     * 提交问题（普通模式）
     */
    #[Route('/conversations/messages', methods: ['POST'])]
    public function ask(Request $request): JsonResponse
    {
        $question = $request->getPayload()->getString('question');

        try {
            // submit() 接收 UserMessage，返回 AssistantMessage
            $response = $this->chat->submit(Message::ofUser($question));

            return new JsonResponse([
                'answer' => $response->getContent(),
                'metadata' => $response->getMetadata()->all(),
            ]);
        } catch (\Throwable $e) {
            return new JsonResponse([
                'error' => '回答生成失败，请稍后重试。',
            ], 500);
        }
    }

    /**
     * 流式问答——需要绕过 Chat 组件，直接管理消息历史
     * 因为 Chat::submit() 返回 AssistantMessage，不支持流式输出
     */
    #[Route('/conversations/stream', methods: ['POST'])]
    public function stream(Request $request): StreamedResponse
    {
        $question = $request->getPayload()->getString('question');

        return new StreamedResponse(function () use ($question) {
            header('Content-Type: text/event-stream');
            header('Cache-Control: no-cache');

            // 1. 加载历史并追加用户消息
            $messages = $this->store->load();
            $messages->add(Message::ofUser($question));

            // 2. 直接调用 Platform，启用流式
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

            // 4. 保存完整回复到历史
            $messages->add(Message::ofAssistant($fullContent));
            $this->store->save($messages);

            echo "data: [DONE]\n\n";
            flush();
        });
    }
}
```

## 运行与验证

1. 启动 PostgreSQL（带 pgvector 扩展）和 Redis
2. 运行文档索引：调用 `DocumentIndexService::indexDirectory()` 索引文档目录
3. 创建新对话：`POST /api/kb/conversations`
4. 提交问题：`POST /api/kb/conversations/messages`，验证返回基于文档的回答
5. 测试流式输出：`POST /api/kb/conversations/stream`，验证 SSE 事件流
6. 验证多轮对话：连续提问，确认上下文保持

## 错误处理

- **文档索引失败**：捕获加载/转换/向量化各阶段异常，记录已完成的进度以便断点续传
- **向量检索无结果**：Agent 应按 system prompt 要求明确告知用户未找到相关信息
- **流式输出中断**：客户端应实现重连机制（EventSource 自动重连）
- **对话历史丢失**：Redis 持久化配置（AOF/RDB），或使用数据库存储关键对话

## 生产环境注意事项

### 生产部署检查清单

| 检查项 | 状态 | 说明 |
|--------|:----:|------|
| FailoverPlatform 配置 | [ ] | 至少配置 2 个 AI 平台 |
| CachePlatform 配置 | [ ] | 使用 Redis 缓存池 |
| PostgreSQL pgvector 扩展 | [ ] | `CREATE EXTENSION vector;` |
| Redis 连接 | [ ] | 对话历史 + 响应缓存 |
| Nginx 流式配置 | [ ] | `proxy_buffering off;` |
| API Key 环境变量 | [ ] | 不要硬编码 |
| 日志和监控 | [ ] | Platform 事件监听 |
| 错误处理 | [ ] | try/catch + 优雅降级 |
| 文档定期重新索引 | [ ] | Cron / Symfony Messenger |

## 扩展方向

- 支持更多文档格式：PDF、Word、HTML 等 Loader
- 实现文档版本管理和增量索引
- 添加权限控制：不同用户只能查询授权范围内的文档
- 集成 Symfony Messenger 实现异步文档索引
- 添加对话评分和反馈机制，持续优化回答质量

## 完整源代码

本章涉及的完整代码片段已在各小节中给出。核心组件：

1. `DocumentIndexService`：文档索引管线（Loader → Transformer → Vectorizer → Store）
2. `KnowledgeBaseController`：对话控制器（新建对话、普通问答、流式问答）
3. AI Bundle YAML 配置：平台层（CachePlatform + FailoverPlatform）+ Agent 定义

## 下一步

恭喜你完成了所有高级实战场景！你已经掌握了 Symfony AI 在生产环境中的核心技术。接下来请继续阅读 [第 25 章：高级架构与最佳实践](25-architecture.md)，或开始构建你自己的项目。
