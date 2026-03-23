# 完整 RAG 聊天系统（全模块整合）

## 业务场景

你在做一个企业级智能客服系统，需要具备以下全部能力：
1. **知识库问答**：基于公司文档回答问题（RAG）
2. **对话持久化**：用户关闭页面后重新打开，对话还在
3. **工具调用**：能查订单、查天气等实时信息
4. **用户记忆**：记住用户的偏好和历史信息
5. **流式输出**：逐字输出提升体验

这个场景整合了前面所有章节的模块，展示它们如何协同工作。

**典型应用：** 企业综合客服平台、智能助手产品、内部知识管理系统

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台 |
| **Agent** | 智能体框架 + 工具调用 |
| **Chat** | 对话持久化 |
| **Store** | 向量存储（知识库 + 记忆） |
| **Agent Memory** | 用户记忆注入 |
| **Agent Bridge** | 各种工具（搜索、时间等） |

## 整体架构

```
┌─────────────────────────────────────────────────┐
│                    Chat 层                       │
│  管理对话历史，持久化到 Redis/数据库               │
└───────────────────────┬─────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────┐
│                   Agent 层                       │
│  ┌───────────────┐ ┌──────────┐ ┌────────────┐  │
│  │MemoryProcessor│ │ToolboxPro│ │SystemPrompt│  │
│  │(用户记忆注入)  │ │(工具调用) │ │(角色定义)  │  │
│  └───────┬───────┘ └────┬─────┘ └────────────┘  │
│          │              │                        │
│  ┌───────▼───────┐ ┌────▼──────────────────┐    │
│  │MemoryProvider │ │  Toolbox              │    │
│  │·Static(用户画像)│ │·SimilaritySearch(RAG)│    │
│  │·Embedding(历史)│ │·Clock(时间)          │    │
│  └───────────────┘ │·Custom(订单查询等)    │    │
│                    └───────────────────────┘    │
└───────────────────────┬─────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────┐
│                 Platform 层                      │
│          连接 OpenAI / Anthropic 等              │
└─────────────────────────────────────────────────┘
```

---

## Step 1：准备知识库

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Store\Bridge\InMemory\Store as VectorStore;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentIndexer;
use Symfony\AI\Store\Indexer\DocumentProcessor;
use Symfony\Component\HttpClient\HttpClient;
use Symfony\Component\Uid\Uuid;

$httpClient = HttpClient::create();
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], $httpClient);

// 1. 初始化向量存储和向量化器
$vectorStore = new VectorStore();
$vectorizer = new Vectorizer($platform, 'text-embedding-3-small');

// 2. 索引知识库文档
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

$indexer = new DocumentIndexer(new DocumentProcessor($vectorizer, $vectorStore));
$indexer->index($documents);
echo "✅ 知识库已索引（" . count($documents) . " 篇文档）\n";
```

---

## Step 2：创建带全部能力的 Agent

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

// 1. 系统提示
$systemPrompt = new SystemPromptInputProcessor(
    '你是 CloudFlow 的智能客服。请遵循以下原则：'
    . '1) 优先使用 SimilaritySearch 工具从知识库查找答案；'
    . '2) 如果知识库没有相关信息，诚实告知用户；'
    . '3) 不要编造产品功能或政策；'
    . '4) 回答简洁、友好、专业。'
);

// 2. 用户记忆（模拟从数据库加载）
$userMemory = new StaticMemoryProvider(
    '用户：张先生，企业版客户',
    '公司：某科技有限公司，50人团队',
    '使用产品已 6 个月',
    '上次咨询过 API 限额问题',
);
$memoryProcessor = new MemoryInputProcessor([$userMemory]);

// 3. 工具集
$tools = [
    new SimilaritySearch($vectorizer, $vectorStore),  // RAG 知识库搜索
    new Clock(new SymfonyClock()),                     // 时间工具
    // 实际项目中还会有：
    // new OrderStatusTool(),   // 订单查询
    // new TicketTool(),        // 工单管理
];

$toolbox = new Toolbox($tools);
$toolProcessor = new AgentProcessor($toolbox);

// 4. 创建 Agent（组合所有处理器）
$agent = new Agent(
    $platform,
    'gpt-4o-mini',
    [$systemPrompt, $memoryProcessor, $toolProcessor],  // 输入处理器
    [$toolProcessor],                                     // 输出处理器
);
```

---

## Step 3：添加对话持久化

```php
<?php

use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\InMemory\Store as ChatStore;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 使用 Chat 组件管理对话（生产中用 Redis/DBAL Store）
$chatStore = new ChatStore();
$chat = new Chat($agent, $chatStore);
```

---

## Step 4：模拟完整的客服对话

```php
<?php

// 模拟一个完整的客服对话场景
$conversationId = 'user-zhang-session-001';

// 初始化对话
$chat->initiate(
    new MessageBag(Message::forSystem('按系统提示工作。')),
    $conversationId,
);

echo "=== CloudFlow 智能客服 ===\n\n";

// 对话 1：普通知识库问题
echo "张先生：你们的退款政策是什么？\n";
$response = $chat->submit(
    Message::ofUser('你们的退款政策是什么？'),
    $conversationId,
);
echo "客服：" . $response->getContent() . "\n\n";

// 对话 2：跟进问题（依赖上下文）
echo "张先生：如果超过 14 天了呢？\n";
$response = $chat->submit(
    Message::ofUser('如果超过 14 天了呢？'),
    $conversationId,
);
echo "客服：" . $response->getContent() . "\n\n";

// 对话 3：涉及工具调用的问题
echo "张先生：现在几点了？你们客服的工作时间是？\n";
$response = $chat->submit(
    Message::ofUser('现在几点了？你们客服的工作时间是？'),
    $conversationId,
);
echo "客服：" . $response->getContent() . "\n\n";

// 对话 4：利用用户记忆的问题
echo "张先生：我之前问过 API 限额的问题，企业版有限制吗？\n";
$response = $chat->submit(
    Message::ofUser('我之前问过 API 限额的问题，企业版有限制吗？'),
    $conversationId,
);
echo "客服：" . $response->getContent() . "\n\n";

// 对话 5：知识库没有的问题
echo "张先生：你们支持 GraphQL 吗？\n";
$response = $chat->submit(
    Message::ofUser('你们支持 GraphQL 吗？'),
    $conversationId,
);
echo "客服：" . $response->getContent() . "\n\n";
```

---

## Step 5：在 Symfony 控制器中使用

在 Symfony 框架中，使用 AI Bundle 可以通过依赖注入获取所有服务。

```php
<?php

namespace App\Controller;

use Symfony\AI\Chat\Chat;
use Symfony\AI\Platform\Message\Message;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Routing\Attribute\Route;

class ChatController extends AbstractController
{
    public function __construct(
        private readonly Chat $chat,
    ) {
    }

    #[Route('/api/chat', methods: ['POST'])]
    public function chat(Request $request): JsonResponse
    {
        $data = json_decode($request->getContent(), true);
        $userMessage = $data['message'];
        $conversationId = $data['conversation_id'];

        // Chat 自动处理：加载历史 → 调用 Agent（含 RAG + 工具 + 记忆）→ 保存消息
        $response = $this->chat->submit(
            Message::ofUser($userMessage),
            $conversationId,
        );

        return $this->json([
            'reply' => $response->getContent(),
            'conversation_id' => $conversationId,
        ]);
    }
}
```

### AI Bundle 配置示例（`config/packages/ai.yaml`）

```yaml
# AI Bundle 会自动注册所有服务
# 具体配置参见 AI Bundle 文档
symfony_ai:
    platform:
        openai:
            api_key: '%env(OPENAI_API_KEY)%'
    agent:
        model: 'gpt-4o-mini'
    store:
        # 配置向量存储
```

---

## 各模块的协作流程

完整一次请求的数据流：

```
1. 用户发送消息
   ↓
2. Chat 层：从 Redis 加载对话历史
   ↓
3. Agent 层 - 输入处理：
   a. SystemPromptInputProcessor → 注入系统提示
   b. MemoryInputProcessor → 检索用户记忆，注入上下文
   c. AgentProcessor → 告知 AI 可用工具列表
   ↓
4. Platform 层：发送给 LLM
   ↓
5. LLM 决定是否需要工具：
   - 需要 → 返回工具调用请求
   - 不需要 → 直接返回文本
   ↓
6. Agent 层 - 输出处理：
   a. AgentProcessor → 执行工具调用
      - SimilaritySearch → 从向量库检索相关文档
      - Clock → 获取当前时间
      - 自定义工具 → 查询业务数据
   b. 将工具结果发回 LLM
   c. LLM 生成最终回答
   ↓
7. Chat 层：保存用户消息和 AI 回复到 Redis
   ↓
8. 返回回复给用户
```

---

## 关键知识点总结

| 模块组合 | 用途 |
|---------|------|
| Platform 单独 | 简单的一问一答 |
| Platform + Chat | 多轮持久化对话 |
| Platform + Agent + Toolbox | 带工具的智能助手 |
| Platform + Store + Agent | RAG 知识库问答 |
| Platform + Agent + Memory | 带记忆的个性化助手 |
| **全部组合** | **企业级智能客服** |

| 概念 | 说明 |
|------|------|
| 输入处理器链 | 多个 InputProcessor 按顺序处理，依次添加上下文 |
| 输出处理器链 | 多个 OutputProcessor 处理 AI 输出（如执行工具调用） |
| 对话 ID | Chat 用它隔离不同用户的对话 |
| 向量存储 | 同时服务于 RAG 和 Embedding 记忆 |

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
