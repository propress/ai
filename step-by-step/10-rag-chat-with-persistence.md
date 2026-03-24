# 完整 RAG 聊天系统（全模块整合）

## 业务场景

你在做一个企业级智能客服系统，需要具备以下全部能力：
1. **知识库问答**：基于公司文档回答问题（RAG）
2. **对话持久化**：用户关闭页面后重新打开，对话还在
3. **工具调用**：能查订单、查天气等实时信息
4. **用户记忆**：记住用户的偏好和历史信息
5. **处理器链**：通过 InputProcessor / OutputProcessor 灵活编排请求管线

这个场景整合了前面所有章节的模块，展示它们如何协同工作。

**典型应用：** 企业综合客服平台、智能助手产品、内部知识管理系统

## 涉及模块

| 模块 | Composer 包 | 用途 |
|------|------------|------|
| **Platform** | `symfony/ai-platform` + `symfony/ai-mistral-platform` | 连接 AI 平台（文本生成 + Embedding） |
| **Agent** | `symfony/ai-agent` | 智能体框架、工具调用、处理器链 |
| **Chat** | `symfony/ai-chat` + `symfony/ai-redis-chat` | 对话管理与持久化 |
| **Store** | `symfony/ai-store` + `symfony/ai-postgres-store` | 向量存储（知识库检索） |
| **Agent Bridge** | `symfony/ai-similarity-search` + `symfony/ai-clock` | 工具集（相似度搜索、时间等） |

> **💡 提示：** 本教程使用 **Mistral**（`mistral-large-latest`）作为主要平台，以展示平台多样性。
> 前面的教程分别使用了 OpenAI、Anthropic、Gemini 等平台。Symfony AI 的 Bridge 架构使切换平台只需更换一行工厂方法，
> 所有业务代码保持不变。如果你更熟悉 OpenAI，可参考 [01-basic-chatbot.md](./01-basic-chatbot.md) 了解对接方式。

---

## 整体架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Chat 层                                   │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ Chat 类：initiate() 初始化会话 / submit() 提交消息             │ │
│  │ 自动处理：加载历史 → 追加消息 → 调用 Agent → 保存回复          │ │
│  └────────────────────────┬───────────────────────────────────────┘ │
│                           │                                         │
│  ┌────────────────────────▼───────────────────────────────────────┐ │
│  │ MessageStore（对话持久化后端）                                  │ │
│  │ · InMemory\Store ─────── 开发/测试用                           │ │
│  │ · Bridge\Redis\MessageStore ── 生产推荐（高性能、TTL 过期）     │ │
│  │ · Bridge\Doctrine\DoctrineDbalMessageStore ── 关系数据库持久化  │ │
│  └────────────────────────────────────────────────────────────────┘ │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────────┐
│                          Agent 层                                    │
│                                                                      │
│  ┌─────────────── InputProcessor 链（按顺序执行）──────────────────┐ │
│  │ ① SystemPromptInputProcessor → 注入系统提示词                   │ │
│  │ ② MemoryInputProcessor ──────→ 检索并注入用户记忆               │ │
│  │ ③ AgentProcessor（输入阶段）──→ 注册可用工具列表                │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                            │                                         │
│                    ┌───────▼───────┐                                 │
│                    │  调用 Platform │                                 │
│                    └───────┬───────┘                                 │
│                            │                                         │
│  ┌─────────────── OutputProcessor 链（处理 AI 输出）───────────────┐ │
│  │ ① AgentProcessor（输出阶段）──→ 检测并执行工具调用              │ │
│  │    ├─ SimilaritySearch ──── 向量检索知识库文档                   │ │
│  │    ├─ Clock ────────────── 获取当前时间                          │ │
│  │    └─ 自定义工具 ────────── 订单查询、工单管理等                 │ │
│  │   执行结果回传 LLM → LLM 生成最终回答                           │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────────┐
│                        Platform 层                                   │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │ Mistral (mistral-large-latest) ── 主平台                     │    │
│  │ 也可切换为 OpenAI / Anthropic / Gemini 等任意 Bridge         │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                            │                                         │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │ Store 层（向量数据库）                                        │    │
│  │ · InMemory\Store ────── 开发/测试                             │    │
│  │ · Bridge\Postgres\Store ── 生产推荐（pgvector，支持混合检索） │    │
│  │ · Bridge\Qdrant\Store ─── 专用向量数据库                      │    │
│  └──────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 前置准备

### 环境要求

- PHP >= 8.2
- Composer
- PostgreSQL 15+（启用 pgvector 扩展）— 向量存储
- Redis 7+（可选）— 对话持久化
- Docker（可选，快速启动依赖服务）

### 启动基础设施

```bash
# PostgreSQL + pgvector（向量存储后端）
docker run -d --name pgvector \
    -e POSTGRES_PASSWORD=secret \
    -e POSTGRES_DB=knowledge_base \
    -p 5432:5432 \
    pgvector/pgvector:pg17

# Redis（对话持久化后端）
docker run -d --name redis \
    -p 6379:6379 \
    redis:7-alpine
```

### 安装依赖

```bash
composer require symfony/ai-platform symfony/ai-mistral-platform \
    symfony/ai-agent symfony/ai-similarity-search symfony/ai-clock \
    symfony/ai-chat symfony/ai-redis-chat \
    symfony/ai-store symfony/ai-postgres-store \
    symfony/uid symfony/http-client symfony/clock
```

### 设置 API 密钥

```bash
export MISTRAL_API_KEY="your-mistral-api-key"
```

> **💡 提示：** 如果使用 OpenAI 代替 Mistral，安装 `symfony/ai-open-ai-platform` 并设置 `OPENAI_API_KEY`。
> 后续代码只需修改 `PlatformFactory` 和模型名称，其余逻辑完全相同。

> **🔒 安全建议：** 永远不要将 API 密钥硬编码在源代码中。在 Symfony 项目中，应使用 `.env.local` 文件或
> 服务器环境变量来管理密钥；在 CI/CD 中使用 Secrets 管理。

---

## Step 1：初始化 Platform 与向量存储

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\Mistral\PlatformFactory;
use Symfony\AI\Store\Bridge\Postgres\Store as VectorStore;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentIndexer;
use Symfony\AI\Store\Indexer\DocumentProcessor;
use Symfony\Component\HttpClient\HttpClient;
use Symfony\Component\Uid\Uuid;

// 1. 创建 Platform 实例（连接 Mistral）
$platform = PlatformFactory::create(
    $_ENV['MISTRAL_API_KEY'],
    HttpClient::create(),
);

// 2. 初始化 PostgreSQL pgvector 向量存储
$pdo = new \PDO(
    'pgsql:host=localhost;port=5432;dbname=knowledge_base',
    'postgres',
    'secret',
);
$vectorStore = new VectorStore($pdo, 'kb_documents');
$vectorStore->setup(['vector_size' => 1024]); // Mistral Embed 输出 1024 维

// 3. 初始化向量化器（使用 Mistral 的 Embedding 模型）
$vectorizer = new Vectorizer($platform, 'mistral-embed');
```

> **💡 提示：** 不同 Embedding 模型的向量维度不同。Mistral Embed 为 1024 维，OpenAI `text-embedding-3-small`
> 为 1536 维。`setup()` 中的 `vector_size` 必须与模型输出维度匹配，否则会报错。

**使用 OpenAI 替代：**

```php
// 切换为 OpenAI 只需两行改动：
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());
$vectorizer = new Vectorizer($platform, 'text-embedding-3-small');
// 注意：setup() 的 vector_size 改为 1536
```

**开发/测试环境使用 InMemory 存储：**

```php
// 无需数据库，适合快速原型和单元测试
use Symfony\AI\Store\InMemory\Store as InMemoryVectorStore;

$vectorStore = new InMemoryVectorStore();
```

---

## Step 2：索引知识库文档

```php
<?php

// 准备知识库文档
$knowledgeBase = [
    ['title' => '退款政策', 'content' => '14天内全额退款，超过14天按比例退。退款3-5个工作日到账。'],
    ['title' => '定价', 'content' => '基础版¥99/月(5用户)，专业版¥299/月(20用户)，企业版联系销售。'],
    ['title' => '安全', 'content' => 'AES-256加密，SOC2+ISO27001认证，支持SSO和双因素认证。'],
    ['title' => 'API', 'content' => 'REST API，Bearer Token认证。基础版1000次/天，专业版10000次/天。'],
    ['title' => '集成', 'content' => '支持Slack、企业微信、钉钉集成。Webhook支持自定义通知。'],
    ['title' => '技术要求', 'content' => '支持Chrome/Firefox/Safari最新版，移动端支持iOS 15+和Android 12+。'],
];

$documents = [];
foreach ($knowledgeBase as $doc) {
    $documents[] = new TextDocument(
        id: Uuid::v4(),
        content: "{$doc['title']}：{$doc['content']}",
        metadata: new Metadata($doc),
    );
}

// DocumentProcessor 自动编排：向量化 → 存储
$indexer = new DocumentIndexer(new DocumentProcessor($vectorizer, $vectorStore));
$indexer->index($documents);

echo "✅ 知识库已索引（" . count($documents) . " 篇文档）\n";
```

> **💡 提示：** `DocumentProcessor` 自动完成「向量化 → 存储」管线。对于大规模知识库，建议将索引过程放在
> 后台任务（如 Symfony Messenger）中异步执行，避免阻塞 Web 请求。

---

## Step 3：创建带全部能力的 Agent

Agent 是整个系统的核心引擎。通过组合 **InputProcessor**（输入处理器）和 **OutputProcessor**（输出处理器），
你可以灵活编排请求管线中的每一个环节。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\Clock\Clock;
use Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;
use Symfony\AI\Agent\Memory\StaticMemoryProvider;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\Component\Clock\Clock as SymfonyClock;

// ── InputProcessor ①：系统提示 ──
$systemPrompt = new SystemPromptInputProcessor(
    '你是 CloudFlow 的智能客服。请遵循以下原则：'
    . '1) 优先使用 similarity_search 工具从知识库查找答案；'
    . '2) 如果知识库没有相关信息，诚实告知用户；'
    . '3) 不要编造产品功能或政策；'
    . '4) 回答简洁、友好、专业。'
);

// ── InputProcessor ②：用户记忆 ──
$userMemory = new StaticMemoryProvider(
    '用户：张先生，企业版客户',
    '公司：某科技有限公司，50人团队',
    '使用产品已 6 个月',
    '上次咨询过 API 限额问题',
);
$memoryProcessor = new MemoryInputProcessor([$userMemory]);

// ── InputProcessor ③ + OutputProcessor ①：工具调用 ──
$tools = [
    new SimilaritySearch($vectorizer, $vectorStore),  // RAG 知识库检索
    new Clock(new SymfonyClock()),                     // 时间查询
];

$toolbox = new Toolbox($tools);
$toolProcessor = new AgentProcessor($toolbox);

// ── 组装 Agent ──
$agent = new Agent(
    platform: $platform,
    model: 'mistral-large-latest',
    inputProcessors: [$systemPrompt, $memoryProcessor, $toolProcessor],
    outputProcessors: [$toolProcessor],
);
```

> **💡 提示：** `AgentProcessor` 同时实现了 `InputProcessorInterface` 和 `OutputProcessorInterface`。
> 在输入阶段，它将可用工具列表注册到请求中；在输出阶段，它检测 LLM 返回的工具调用请求并执行对应工具，
> 然后将执行结果回传 LLM 生成最终回答。这就是为什么它同时出现在 `inputProcessors` 和 `outputProcessors` 中。

### InputProcessor / OutputProcessor 处理链详解

处理器按数组中的顺序依次执行，每个处理器修改共享的 `Input` 或 `Output` 对象：

```
InputProcessor 链（请求前处理）：
┌──────────────────────┐    ┌───────────────────────┐    ┌──────────────────┐
│ SystemPromptInput    │───▶│ MemoryInputProcessor  │───▶│ AgentProcessor   │
│ Processor            │    │                       │    │ （输入阶段）     │
│                      │    │ 从 MemoryProvider     │    │                  │
│ 向 MessageBag 添加   │    │ 获取记忆内容，         │    │ 将 Toolbox 中的  │
│ 系统提示消息          │    │ 注入系统提示上下文     │    │ 工具定义注册到   │
│                      │    │                       │    │ 请求的 options   │
└──────────────────────┘    └───────────────────────┘    └──────────────────┘

OutputProcessor 链（响应后处理）：
┌────────────────────────────────────────────────────────────────────┐
│ AgentProcessor（输出阶段）                                         │
│                                                                    │
│ 1. 检查 LLM 响应是否包含 tool_calls                               │
│ 2. 如果有 → 执行工具 → 将结果添加到 MessageBag → 再次调用 LLM     │
│ 3. 重复直到 LLM 返回纯文本（无工具调用）                           │
│ 4. 返回最终文本响应                                                │
└────────────────────────────────────────────────────────────────────┘
```

> **⚠️ 注意：** 输入处理器的顺序很重要。`SystemPromptInputProcessor` 应在最前面，确保系统提示最先添加；
> `AgentProcessor` 应在最后，因为它需要读取前面处理器已添加的完整上下文。

---

## Step 4：使用 Retriever 进行文档检索

除了通过 `SimilaritySearch` 工具让 Agent 自动检索，你也可以使用 `Retriever` 类直接执行检索。
`Retriever` 是一个高层封装，自动组合向量存储和向量化器，支持向量查询、文本查询和混合查询。

```php
<?php

use Symfony\AI\Store\Retriever;

// Retriever 封装了 Store + Vectorizer 的查询逻辑
$retriever = new Retriever(
    store: $vectorStore,
    vectorizer: $vectorizer,
);

// 直接执行语义检索
$results = $retriever->retrieve('退款政策是什么？');

foreach ($results as $document) {
    echo "相关文档：{$document->id}\n";
    // $document 包含原始内容和元数据
}
```

> **💡 提示：** `Retriever` 会自动选择最佳查询策略：如果 Store 支持混合检索（如 pgvector），
> 它会使用 `HybridQuery`（语义 + 全文搜索）；否则退回到 `VectorQuery` 或 `TextQuery`。
> 你也可以通过 `$options` 参数显式控制查询行为。

---

## Step 5：添加对话持久化

`Chat` 类将 Agent 与 MessageStore 组合在一起，自动管理对话历史的加载、追加和保存。

### 生产环境：使用 Redis 持久化对话

```php
<?php

use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\Bridge\Redis\MessageStore;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 连接 Redis
$redis = new \Redis();
$redis->connect('127.0.0.1', 6379);

// 每个会话使用唯一的 indexName 来隔离对话
$sessionId = 'user-zhang-session-001';
$chatStore = new MessageStore($redis, $sessionId);
$chatStore->setup();

// 创建 Chat 实例
$chat = new Chat($agent, $chatStore);
```

> **🔒 安全建议：** `MessageStore` 的 `$indexName` 参数是对话隔离的关键。在多用户系统中，
> 务必将用户 ID 嵌入 key，例如 `"chat:{$userId}:{$sessionId}"`，防止用户间数据泄露。
> 切勿使用客户端提交的 ID 直接作为 key，需先验证该用户是否有权访问该会话。

**开发/测试环境使用 InMemory：**

```php
use Symfony\AI\Chat\InMemory\Store as InMemoryChatStore;

$chatStore = new InMemoryChatStore();
$chat = new Chat($agent, $chatStore);
```

**使用 Doctrine DBAL 持久化到关系数据库：**

```php
use Symfony\AI\Chat\Bridge\Doctrine\DoctrineDbalMessageStore;

$chatStore = new DoctrineDbalMessageStore('chat_messages', $dbalConnection);
$chatStore->setup(); // 自动创建数据库表
$chat = new Chat($agent, $chatStore);
```

### 对话生命周期管理

```php
<?php

// 1. initiate() — 初始化新会话（清除旧消息并保存初始 MessageBag）
$chat->initiate(
    new MessageBag(Message::forSystem('按系统提示工作。')),
);

// 2. submit() — 提交用户消息（自动：加载历史 → 追加消息 → 调用 Agent → 保存回复）
$response = $chat->submit(
    Message::ofUser('你们的退款政策是什么？'),
);
echo $response->getContent();

// 3. 后续消息自动保持上下文
$response = $chat->submit(
    Message::ofUser('如果超过 14 天了呢？'),
);
echo $response->getContent();
```

> **⚠️ 注意：** `Chat` 类的 `initiate()` 和 `submit()` 方法**没有** `conversationId` 参数。
> 对话隔离在 MessageStore 层实现——每个 Chat 实例绑定一个 MessageStore，
> 而 MessageStore 的 `$indexName`（Redis）或 `$identifier`（InMemory）就是会话标识。
> 多用户系统中，为每个用户会话创建独立的 `Chat` 实例。

---

## Step 6：模拟完整的客服对话

```php
<?php

echo "=== CloudFlow 智能客服 ===\n\n";

// 初始化对话
$chat->initiate(
    new MessageBag(Message::forSystem('按系统提示工作。')),
);

// 对话 1：普通知识库问题（触发 SimilaritySearch 工具）
echo "张先生：你们的退款政策是什么？\n";
$response = $chat->submit(
    Message::ofUser('你们的退款政策是什么？'),
);
echo "客服：" . $response->getContent() . "\n\n";

// 对话 2：跟进问题（依赖上下文，Chat 自动加载历史）
echo "张先生：如果超过 14 天了呢？\n";
$response = $chat->submit(
    Message::ofUser('如果超过 14 天了呢？'),
);
echo "客服：" . $response->getContent() . "\n\n";

// 对话 3：涉及工具调用的问题（触发 Clock 工具）
echo "张先生：现在几点了？你们客服的工作时间是？\n";
$response = $chat->submit(
    Message::ofUser('现在几点了？你们客服的工作时间是？'),
);
echo "客服：" . $response->getContent() . "\n\n";

// 对话 4：利用用户记忆的问题（MemoryInputProcessor 注入的上下文）
echo "张先生：我之前问过 API 限额的问题，企业版有限制吗？\n";
$response = $chat->submit(
    Message::ofUser('我之前问过 API 限额的问题，企业版有限制吗？'),
);
echo "客服：" . $response->getContent() . "\n\n";

// 对话 5：知识库没有的问题（Agent 应诚实回答）
echo "张先生：你们支持 GraphQL 吗？\n";
$response = $chat->submit(
    Message::ofUser('你们支持 GraphQL 吗？'),
);
echo "客服：" . $response->getContent() . "\n\n";
```

> **💡 提示：** 每次 `submit()` 调用时，Chat 会从 MessageStore 加载完整的对话历史，
> 包括之前的用户消息、AI 回复和工具调用结果。这就是为什么 AI 能理解「如果超过 14 天了呢？」
> ——它看到了上一轮关于退款政策的完整对话。

> **⚠️ 注意：** 随着对话轮次增多，MessageBag 中的消息会不断累积。大多数 LLM 有上下文窗口限制
> （如 Mistral Large 为 128K Token）。对于长对话，建议实现消息截断策略——保留系统提示
> 和最近 N 轮对话，丢弃较早的消息。可以通过自定义 `InputProcessor` 实现这一逻辑。

---

## Step 7：在 Symfony 控制器中使用

在 Symfony 框架中，使用 AI Bundle 可以通过依赖注入获取所有服务。

```php
<?php

namespace App\Controller;

use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\Bridge\Redis\MessageStore;
use Symfony\AI\Agent\AgentInterface;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Routing\Attribute\Route;
use Symfony\Component\Security\Http\Attribute\IsGranted;

#[IsGranted('ROLE_USER')]
class ChatController extends AbstractController
{
    public function __construct(
        private readonly AgentInterface $agent,
        private readonly \Redis $redis,
    ) {
    }

    #[Route('/api/chat', methods: ['POST'])]
    public function chat(Request $request): JsonResponse
    {
        $data = json_decode($request->getContent(), true);
        $userMessage = $data['message'];
        $sessionId = $data['session_id'];

        // 使用已认证的用户 ID 构建隔离的 Store key
        $userId = $this->getUser()->getUserIdentifier();
        $storeKey = "chat:{$userId}:{$sessionId}";

        $chatStore = new MessageStore($this->redis, $storeKey);
        $chatStore->setup();
        $chat = new Chat($this->agent, $chatStore);

        // Chat 自动处理：加载历史 → 调用 Agent（含 RAG + 工具 + 记忆）→ 保存消息
        $response = $chat->submit(
            Message::ofUser($userMessage),
        );

        return $this->json([
            'reply' => $response->getContent(),
            'session_id' => $sessionId,
        ]);
    }

    #[Route('/api/chat/new', methods: ['POST'])]
    public function newChat(Request $request): JsonResponse
    {
        $sessionId = bin2hex(random_bytes(16));
        $userId = $this->getUser()->getUserIdentifier();
        $storeKey = "chat:{$userId}:{$sessionId}";

        $chatStore = new MessageStore($this->redis, $storeKey);
        $chatStore->setup();
        $chat = new Chat($this->agent, $chatStore);

        // 初始化会话
        $chat->initiate(
            new MessageBag(Message::forSystem('按系统提示工作。')),
        );

        return $this->json(['session_id' => $sessionId]);
    }
}
```

> **🔒 安全建议：** Store key 中的用户 ID 必须来自服务端认证（如 `$this->getUser()`），
> 而非客户端提交。始终使用 `{userId}:{sessionId}` 格式构建 key，确保用户只能访问自己的会话。

> **🏭 生产建议：** 在 Symfony 项目中推荐通过 AI Bundle（`symfony/ai-bundle`）来配置 Agent、Chat Store 等服务，
> 通过依赖注入获取实例，而不是手动创建。参见 AI Bundle 文档了解 YAML 配置方式。

---

## 各模块的协作流程

完整一次请求的数据流：

```
1. 用户发送消息
   ↓
2. Chat 层：从 MessageStore 加载对话历史（MessageBag）
   ↓
3. Chat 层：将用户消息追加到 MessageBag
   ↓
4. Agent 层 - InputProcessor 链：
   a. SystemPromptInputProcessor → 注入系统提示词
   b. MemoryInputProcessor → 从 MemoryProvider 检索用户记忆，注入上下文
   c. AgentProcessor（输入）→ 从 Toolbox 读取工具定义，注册到请求 options
   ↓
5. Platform 层：将完整 MessageBag + options 发送给 Mistral LLM
   ↓
6. LLM 返回响应，可能包含 tool_calls
   ↓
7. Agent 层 - OutputProcessor 链：
   a. AgentProcessor（输出）检查响应：
      ├─ 包含 tool_calls → 执行对应工具：
      │   ├─ SimilaritySearch → 向量化搜索词 → pgvector 检索 → 返回相关文档
      │   ├─ Clock → 获取系统时间
      │   └─ 自定义工具 → 查询业务数据
      ├─ 将工具执行结果追加到 MessageBag
      ├─ 再次调用 Platform（LLM 基于工具结果生成回答）
      └─ 重复直到 LLM 返回纯文本
   ↓
8. Chat 层：将用户消息 + AI 回复保存到 MessageStore
   ↓
9. 返回 AssistantMessage 给用户
```

---

## 从开发到生产：存储后端切换

| 组件 | 开发/测试 | 生产推荐 | 切换方式 |
|------|----------|---------|---------|
| **向量存储** | `InMemory\Store` | `Bridge\Postgres\Store`（pgvector） | 替换 `$vectorStore` 实例 |
| **对话存储** | `InMemory\Store` | `Bridge\Redis\MessageStore` | 替换 `$chatStore` 实例 |
| **AI 平台** | 任意 Bridge | 同左 | 替换 `PlatformFactory` |

> **🏭 生产建议：** pgvector 选择 `Cosine` 距离度量适合大多数文本语义搜索场景。
> 对于大规模数据集（百万级文档），建议在 `setup()` 时配置 HNSW 索引以加速查询：
> `$vectorStore->setup(['vector_size' => 1024, 'index_method' => 'hnsw'])`。

---

## 关键知识点总结

| 模块组合 | 用途 |
|---------|------|
| Platform 单独 | 简单的一问一答 |
| Platform + Chat | 多轮持久化对话 |
| Platform + Agent + Toolbox | 带工具的智能助手 |
| Platform + Store + Agent | RAG 知识库问答 |
| Platform + Agent + Memory | 带记忆的个性化助手 |
| **全部组合** | **企业级智能客服（本教程）** |

| 概念 | 说明 |
|------|------|
| InputProcessor 链 | 多个 `InputProcessorInterface` 按数组顺序执行，依次向 `Input` 添加上下文 |
| OutputProcessor 链 | 多个 `OutputProcessorInterface` 处理 AI 输出（如检测并执行工具调用） |
| AgentProcessor | 同时实现 Input/Output 两个接口，负责工具注册和工具执行的完整循环 |
| MessageStore 隔离 | 每个 Chat 实例绑定一个 MessageStore，通过 `$indexName` 隔离不同用户/会话 |
| Retriever | 高层检索封装，自动选择最佳查询策略（向量 / 文本 / 混合） |
| SimilaritySearch | Agent 工具，让 LLM 在对话中自动触发语义检索 |
| Token 上下文窗口 | 对话历史会累积，需注意 LLM 的上下文长度限制，长对话需截断策略 |

## 系列文档总结

| 文档 | 场景 | 核心模块 | 复杂度 |
|------|------|---------|--------|
| [01-basic-chatbot](./01-basic-chatbot.md) | 基础问答 | Platform | ⭐ |
| [02-multi-turn-conversation](./02-multi-turn-conversation.md) | 持久化对话 | Platform + Chat | ⭐⭐ |
| [03-tool-augmented-assistant](./03-tool-augmented-assistant.md) | 工具调用 | Platform + Agent | ⭐⭐ |
| [04-rag-knowledge-base](./04-rag-knowledge-base.md) | 知识库 RAG | Platform + Agent + Store | ⭐⭐⭐ |
| [05-customer-service-multi-agent](./05-customer-service-multi-agent.md) | 多智能体路由 | Platform + Agent MultiAgent | ⭐⭐⭐ |
| [06-structured-data-extraction](./06-structured-data-extraction.md) | 结构化提取 | Platform + StructuredOutput | ⭐⭐ |
| [07-multimodal-content-understanding](./07-multimodal-content-understanding.md) | 多模态处理 | Platform + Content 类 | ⭐⭐ |
| [08-agent-with-memory](./08-agent-with-memory.md) | 长期记忆 | Platform + Agent + Store | ⭐⭐⭐ |
| [09-web-research-assistant](./09-web-research-assistant.md) | 联网研究 | Platform + Agent + 工具桥接 | ⭐⭐⭐ |
| [10-rag-chat-with-persistence](./10-rag-chat-with-persistence.md) | 全模块整合 | 全部 | ⭐⭐⭐⭐ |
