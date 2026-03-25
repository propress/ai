# 第 11 章：实战场景（高级篇）

## 🎯 本章学习目标

通过 5 个高级实战场景，掌握生产环境关键技术：高可用容灾架构、缓存策略、本地模型部署、多智能体团队协作、以及完整的端到端系统集成。

---

## 1. 回顾

在 [第 10 章](10-scenarios-intermediate.md) 中，我们掌握了：

- 工具调用、RAG 检索、多智能体路由、联网搜索、记忆系统

本章聚焦**生产环境的关键技术**，解决可用性、性能、成本和安全等实际挑战。

---

## 2. 场景一：高可用 AI 服务架构

### 2.1 业务场景

生产环境中，AI 服务需要高可用性保障。单一平台可能出现故障、限速或延迟。解决方案是使用多平台容灾和智能缓存。

**适用场景**：任何生产级 AI 应用。

### 2.2 FailoverPlatform：多平台容灾

```php
<?php

use Symfony\AI\Platform\Bridge\Failover\FailoverPlatform;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory as OpenAiFactory;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory as AnthropicFactory;

// 创建多个平台实例
$openai = OpenAiFactory::create($_ENV['OPENAI_API_KEY']);
$anthropic = AnthropicFactory::create($_ENV['ANTHROPIC_API_KEY']);

// 创建容灾平台——主平台失败时自动切换到备用平台
$platform = new FailoverPlatform([$openai, $anthropic]);

// 使用方式与单平台完全一致
$response = $platform->invoke($messages, $model);
```

### 2.3 CachePlatform：响应缓存

```php
use Symfony\AI\Platform\Bridge\Cache\CachePlatform;
use Symfony\Component\Cache\Adapter\RedisAdapter;

$cache = new RedisAdapter(/* Redis 连接 */);

$platform = new CachePlatform(
    $innerPlatform,
    $cache,
);

// 相同输入的重复请求将命中缓存
$response = $platform->invoke($messages, $model, [
    'prompt_cache_key' => 'product-faq',  // 缓存键（必须提供）
]);
```

> ⚠️ **注意**：`CachePlatform` 只在选项中包含非空 `prompt_cache_key` 时才启用缓存。对于 `MessageBag` 输入，它使用 `MessageBag::getId()->toString()` 作为缓存键组件。

### 2.4 组合使用

```php
// 缓存 → 容灾 → 实际平台
// 请求优先命中缓存，缓存未命中时使用容灾平台

$failover = new FailoverPlatform([$openai, $anthropic]);
$platform = new CachePlatform($failover, $cache);

// 完整的高可用调用链：
// 1. 检查缓存 → 命中则直接返回
// 2. 缓存未命中 → 调用 OpenAI
// 3. OpenAI 失败 → 自动切换到 Anthropic
// 4. 成功响应写入缓存
```

### 2.5 AI Bundle 配置

```yaml
# config/packages/ai.yaml
ai:
    platform:
        open_ai:
            api_key: '%env(OPENAI_API_KEY)%'
        anthropic:
            api_key: '%env(ANTHROPIC_API_KEY)%'
        # 容灾平台
        failover:
            type: failover
            platforms:
                - ai.platform.open_ai
                - ai.platform.anthropic
        # 缓存平台
        cached:
            type: cache
            platform: ai.platform.failover
            cache_pool: cache.ai
```

---

## 3. 场景二：本地模型私有化部署

### 3.1 业务场景

某些场景（数据安全、合规要求、成本控制）需要使用本地模型，避免数据外传。Ollama 让本地运行大模型变得简单。

**适用场景**：数据敏感行业（金融、医疗）、离线环境、成本优化。

### 3.2 安装 Ollama

```bash
# macOS / Linux
curl -fsSL https://ollama.com/install.sh | sh

# 下载模型
ollama pull llama3.1
ollama pull nomic-embed-text  # Embedding 模型
```

### 3.3 使用 Ollama Bridge

```php
<?php

use Symfony\AI\Platform\Bridge\Ollama\PlatformFactory;
use Symfony\AI\Platform\Bridge\Ollama\Llama;

// Ollama 默认运行在 http://localhost:11434
$platform = PlatformFactory::create();

$response = $platform->invoke(
    $messages,
    new Llama(Llama::LLAMA_3_1),
);

echo $response->asText();
```

### 3.4 本地 RAG

```php
use Symfony\AI\Platform\Bridge\Ollama\PlatformFactory;

// 使用本地 Embedding 模型
$platform = PlatformFactory::create();

// 向量化文档——数据完全不出本地网络
$vectorizer = new Vectorizer($platform, 'nomic-embed-text');
$vectorizer->vectorize($documents);

// 使用本地 LLM 生成回答
$agent = new Agent($platform, new Llama(), toolbox: $toolbox);
```

### 3.5 混合部署：本地 + 云端

```php
// 敏感数据处理用本地模型
$localPlatform = OllamaFactory::create();

// 非敏感的通用任务用云端模型（质量更好）
$cloudPlatform = OpenAiFactory::create($_ENV['OPENAI_API_KEY']);

// 根据数据敏感性选择平台
$platform = $isSensitive ? $localPlatform : $cloudPlatform;
```

---

## 4. 场景三：内容审核流水线

### 4.1 业务场景

构建一个生产级的内容审核系统，支持文本和图片审核，使用多平台交叉验证和缓存优化。

**适用场景**：UGC 平台、社交应用、电商评论系统。

### 4.2 审核结果 DTO

```php
<?php

namespace App\Dto;

use Symfony\AI\Platform\Contract\JsonSchema\Attribute\With;

class ModerationResult
{
    public function __construct(
        #[With(description: '是否安全通过')]
        public readonly bool $safe,
        #[With(description: '置信度 0-100')]
        public readonly int $confidence,
        #[With(description: '违规类别列表')]
        public readonly array $categories,
        #[With(description: '审核说明')]
        public readonly string $reason,
        #[With(description: '建议操作：approve/review/reject')]
        public readonly string $action,
    ) {}
}
```

### 4.3 审核 Agent

```php
<?php

use Symfony\AI\Agent\Agent;

// 审核系统 Agent
$moderator = new Agent($platform, $model, systemPrompt: <<<PROMPT
你是一个专业的内容审核系统。
审核内容是否包含以下违规类别：
- 暴力/血腥内容
- 色情/不雅内容
- 仇恨言论/歧视
- 虚假信息/诈骗
- 个人隐私泄露

对每条内容给出结构化审核结果。
当置信度低于 70% 时，建议人工复核。
PROMPT);

$response = $moderator->call(new MessageBag(
    Message::ofUser('请审核以下内容：'.$userContent),
), options: ['output' => ModerationResult::class]);

$result = $response->unwrap();

match ($result->action) {
    'approve' => $this->publishContent($content),
    'review' => $this->sendToManualReview($content, $result),
    'reject' => $this->rejectContent($content, $result->reason),
};
```

### 4.4 多模态审核（图片）

```php
use Symfony\AI\Platform\Message\Content\Image;

$response = $moderator->call(new MessageBag(
    Message::ofUser(
        Image::fromFile($uploadedImage->getPathname()),
        '请审核这张用户上传的图片。',
    ),
), options: ['output' => ModerationResult::class]);
```

### 4.5 缓存优化

对于相同或相似内容的重复审核，使用缓存避免重复调用 AI：

```php
$cachedPlatform = new CachePlatform($platform, $cache);

// 相同内容的审核请求会命中缓存
$response = $cachedPlatform->invoke($messages, $model, [
    'prompt_cache_key' => 'moderation-'.md5($content),
]);
```

---

## 5. 场景四：CrewAI 风格多智能体团队

### 5.1 业务场景

构建一个多智能体协作团队——研究员搜集信息、写手撰写内容、编辑审核修改，模拟真实团队的分工协作。

**适用场景**：内容生产流水线、研究报告自动生成、自动化写作。

### 5.2 定义角色

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\MultiAgent\Orchestrator;
use Symfony\AI\Agent\MultiAgent\Handoff;

// 1. 研究员——负责收集信息
$researcher = new Agent($platform, $model,
    toolbox: new Toolbox(new ReflectionToolAnalyzer(), [
        new TavilySearchTool($_ENV['TAVILY_API_KEY']),
    ]),
    systemPrompt: '你是一个高级研究员。使用搜索工具收集全面的信息，'
        .'整理成结构化的研究报告。注重数据和事实。'
);

// 2. 写手——负责撰写内容
$writer = new Agent($platform, $model, systemPrompt:
    '你是一个专业的技术写手。基于研究员提供的信息，'
    .'撰写清晰、有深度的技术文章。使用 Markdown 格式。'
);

// 3. 编辑——负责审核和优化
$editor = new Agent($platform, $model, systemPrompt:
    '你是一个资深编辑。审核文章的准确性、逻辑性和可读性。'
    .'提出具体的修改建议或直接修改。'
);
```

### 5.3 顺序流水线

```php
// 任务：撰写一篇关于 Symfony AI 的技术文章

// 步骤 1：研究
$research = $researcher->call(new MessageBag(
    Message::ofUser('请研究 Symfony AI 框架的最新动态和核心特性'),
));

// 步骤 2：写作（基于研究结果）
$draft = $writer->call(new MessageBag(
    Message::forSystem('以下是研究报告，请据此撰写技术文章：'),
    Message::ofUser($research->asText()),
));

// 步骤 3：编辑审核
$final = $editor->call(new MessageBag(
    Message::forSystem('请审核并优化以下文章草稿：'),
    Message::ofUser($draft->asText()),
));

echo $final->asText();
```

### 5.4 层级编排（Orchestrator）

对于更复杂的场景，使用 Orchestrator 让 AI 自动决定工作流：

```php
$orchestrator = new Orchestrator($platform, $model, [
    new Handoff($researcher, 'researcher', '需要搜集信息和数据'),
    new Handoff($writer, 'writer', '需要撰写文章或内容'),
    new Handoff($editor, 'editor', '需要审核或修改内容'),
]);

$response = $orchestrator->call(new MessageBag(
    Message::ofUser('请帮我完成一篇关于 PHP 性能优化的技术博客'),
));
```

---

## 6. 场景五：端到端企业知识库系统

### 6.1 业务场景

构建一个完整的企业知识库系统，整合几乎所有组件：文档索引、向量检索、多轮对话、工具调用、缓存和持久化。

**适用场景**：企业知识管理、内部 wiki 智能问答。

### 6.2 架构概览

```
┌─────────────────────────────────────────────────────────┐
│                    企业知识库系统                          │
│                                                         │
│  ┌─────────────┐   ┌──────────────┐   ┌──────────────┐  │
│  │  文档管线     │   │  Chat 持久化  │   │  Agent       │  │
│  │  Loader →    │   │  Redis Store │   │  + Toolbox   │  │
│  │  Splitter →  │   │              │   │  + Retriever │  │
│  │  Vectorizer →│   │  initiate()  │   │              │  │
│  │  Store       │   │  submit()    │   │  call()      │  │
│  └──────┬──────┘   └──────┬───────┘   └──────┬───────┘  │
│         │                 │                   │          │
│         └─────────────────┼───────────────────┘          │
│                           │                              │
│                  ┌────────▼────────┐                     │
│                  │   CachePlatform │                     │
│                  │ + FailoverPlatform                    │
│                  │   (OpenAI | Anthropic)                │
│                  └─────────────────┘                     │
└─────────────────────────────────────────────────────────┘
```

### 6.3 系统配置（AI Bundle）

```yaml
# config/packages/ai.yaml
ai:
    platform:
        open_ai:
            api_key: '%env(OPENAI_API_KEY)%'
        anthropic:
            api_key: '%env(ANTHROPIC_API_KEY)%'
        failover:
            type: failover
            platforms: [ai.platform.open_ai, ai.platform.anthropic]

    store:
        knowledge_base:
            platform: ai.platform.failover
            store: ai.store.postgres

    agent:
        knowledge_assistant:
            platform: ai.platform.failover
            model: gpt-4o
            system: |
                你是企业知识库助手。
                使用 similarity_search 工具检索文档后回答问题。
                如果找不到相关信息，明确告知用户。
                每个回答都引用来源文档。
```

### 6.4 对话控制器

```php
<?php

namespace App\Controller;

use Symfony\AI\Agent\AgentInterface;
use Symfony\AI\Chat\ChatInterface;
use Symfony\Component\DependencyInjection\Attribute\Target;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\Routing\Attribute\Route;

class KnowledgeBaseController
{
    public function __construct(
        #[Target('knowledge_assistant')]
        private readonly AgentInterface $agent,
        private readonly ChatInterface $chat,
    ) {}

    #[Route('/api/kb/new', methods: ['POST'])]
    public function newConversation(): JsonResponse
    {
        $conversation = $this->chat->initiate(
            Message::forSystem('你是企业知识库助手。'),
        );

        return new JsonResponse(['conversation_id' => $conversation->getId()->toString()]);
    }

    #[Route('/api/kb/ask', methods: ['POST'])]
    public function ask(Request $request): JsonResponse
    {
        $conversationId = $request->getPayload()->getString('conversation_id');
        $question = $request->getPayload()->getString('question');

        $response = $this->chat->submit($conversationId, $question);

        return new JsonResponse([
            'answer' => $response->asText(),
            'sources' => $response->getSources(),
        ]);
    }
}
```

---

## 7. 本章小结

通过五个高级场景，我们掌握了以下生产级模式：

| 模式 | 场景 | 关键组件 |
|------|------|---------|
| **高可用** | 容灾架构 | FailoverPlatform + CachePlatform |
| **私有化** | 本地模型 | Ollama Bridge |
| **安全** | 内容审核 | StructuredOutput + Agent |
| **协作** | 多智能体团队 | Orchestrator + 顺序流水线 |
| **端到端** | 企业知识库 | 全组件集成 |

---

## 8. 下一步

在 [第 12 章](12-architecture.md) 中，我们将深入探讨高级架构设计与最佳实践——包括设计模式、错误处理、安全性、可观测性等跨场景的通用原则。
