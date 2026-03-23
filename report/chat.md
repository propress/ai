# Chat 模块分析报告

> 基于 `src/chat/` 源码的深度技术分析，版本 symfony/ai-chat ^0.6

---

## 1. 模块概述

Chat 模块（`symfony/ai-chat`）是 Symfony AI 生态系统中负责**多轮对话状态管理**的核心组件。它并不直接与 AI 模型通信——这由 Platform 层负责——而是在每次用户与助手的交互之间，负责**持久化和恢复完整的对话上下文**（`MessageBag`）。

### 1.1 模块在整体架构中的位置

```
用户请求
   │
   ▼
ChatInterface (Chat.php)
   │  ┌─────────────────────────────┐
   │  │  MessageStoreInterface       │  ←── 本模块核心
   │  │  (Cache / Redis / Doctrine   │
   │  │   / Session / MongoDB / …)   │
   │  └─────────────────────────────┘
   │
   ▼
AgentInterface (来自 symfony/ai-agent)
   │
   ▼
PlatformInterface (来自 symfony/ai-platform)
   │
   ▼
LLM API（OpenAI / Anthropic / Gemini 等）
```

Chat 模块的核心职责：
- 将当前轮次的用户消息附加到历史消息序列
- 调用 Agent 获取 AI 响应
- 将助手回复也追加进消息序列
- 把更新后的完整消息序列序列化回存储后端

### 1.2 主要类与接口清单

| 类/接口 | 职责 |
|---|---|
| `ChatInterface` | 对话操作的顶层契约 |
| `Chat` | `ChatInterface` 的唯一内置实现 |
| `MessageStoreInterface` | 消息的存取契约 |
| `ManagedStoreInterface` | 存储基础设施的生命周期管理契约 |
| `MessageNormalizer` | 消息的序列化/反序列化（JSON） |
| `InMemory\Store` | 进程内临时存储实现 |
| `Bridge\Cache\MessageStore` | PSR-6 Cache 存储实现 |
| `Bridge\Redis\MessageStore` | Redis 原生存储实现 |
| `Bridge\Doctrine\DoctrineDbalMessageStore` | Doctrine DBAL 数据库存储实现 |
| `Bridge\Session\MessageStore` | Symfony Session 存储实现 |
| `Bridge\MongoDB\MessageStore` | MongoDB 存储实现 |
| `Bridge\Meilisearch\MessageStore` | Meilisearch 存储实现 |
| `Bridge\Cloudflare\MessageStore` | Cloudflare KV 存储实现 |
| `Bridge\Pogocache\MessageStore` | Pogocache 存储实现 |
| `Bridge\SurrealDb\MessageStore` | SurrealDB 存储实现 |
| `Command\SetupStoreCommand` | CLI：初始化存储基础设施 |
| `Command\DropStoreCommand` | CLI：销毁存储基础设施 |

### 1.3 依赖关系

```json
{
    "require": {
        "php": ">=8.2",
        "symfony/ai-agent": "^0.6",
        "symfony/ai-platform": "^0.6",
        "symfony/service-contracts": "^2.5|^3"
    }
}
```

桥接各后端的额外依赖（按需安装）：
- **Cache**: `psr/cache`
- **Redis**: `ext-redis`
- **Doctrine**: `doctrine/dbal`
- **Session**: `symfony/http-foundation`
- **MongoDB**: `mongodb/mongodb`
- **Meilisearch**: `meilisearch/meilisearch-php`
- **Cloudflare**: Cloudflare Workers KV API
- **Pogocache**: Pogocache PHP 客户端
- **SurrealDB**: SurrealDB PHP 客户端

---

## 2. 核心接口与输入输出

### 2.1 ChatInterface

```php
namespace Symfony\AI\Chat;

use Symfony\AI\Agent\Exception\ExceptionInterface;
use Symfony\AI\Platform\Message\AssistantMessage;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\Message\UserMessage;

interface ChatInterface
{
    /**
     * 初始化对话，传入系统提示等初始消息包。
     * 调用此方法会重置当前存储中的消息历史。
     */
    public function initiate(MessageBag $messages): void;

    /**
     * 提交用户消息，获取助手回复。
     *
     * @throws ExceptionInterface Agent 调用失败时抛出
     */
    public function submit(UserMessage $message): AssistantMessage;
}
```

**`initiate()` 方法详解：**

| 参数 | 类型 | 说明 |
|---|---|---|
| `$messages` | `MessageBag` | 初始消息集合，通常包含 `SystemMessage`（角色定义、规则约束） |

- 内部调用 `$this->store->drop()` 清空旧历史
- 然后调用 `$this->store->save($messages)` 保存初始上下文
- 适合"开始新对话"或"重置对话场景"

**`submit()` 方法详解：**

| 参数 | 类型 | 说明 |
|---|---|---|
| `$message` | `UserMessage` | 当前轮次用户输入，支持纯文本、图像、文件、音频等内容类型 |

返回值 `AssistantMessage` 包含：
- `getContent(): string` — AI 生成的文本回复
- `getMetadata()` — 元数据（token 使用量、模型信息等）
- `hasToolCalls(): bool` + `getToolCalls()` — 工具调用（如使用了 Function Calling）

**Chat 实现的完整流程：**

```php
// src/chat/src/Chat.php
public function submit(UserMessage $message): AssistantMessage
{
    // 1. 从存储恢复完整历史
    $messages = $this->store->load();

    // 2. 追加当前用户消息
    $messages->add($message);

    // 3. 将完整历史发给 AI Agent
    $result = $this->agent->call($messages);
    assert($result instanceof TextResult);

    // 4. 构建助手消息对象，合并元数据
    $assistantMessage = Message::ofAssistant($result->getContent());
    $assistantMessage->getMetadata()->merge($result->getMetadata());

    // 5. 将助手回复也追加进历史
    $messages->add($assistantMessage);

    // 6. 持久化更新后的完整历史
    $this->store->save($messages);

    return $assistantMessage;
}
```

### 2.2 MessageStoreInterface

```php
namespace Symfony\AI\Chat;

use Symfony\AI\Platform\Message\MessageBag;

interface MessageStoreInterface
{
    /**
     * 持久化完整的消息包（覆盖写入）
     */
    public function save(MessageBag $messages): void;

    /**
     * 加载完整的消息包，若不存在则返回空 MessageBag
     */
    public function load(): MessageBag;
}
```

**MessageBag 结构：**

`MessageBag` 是一个有序消息集合，内部维护一个 `MessageInterface[]` 数组。支持的消息类型：

| 消息类型 | 说明 |
|---|---|
| `SystemMessage` | 系统角色提示（对话全程有效） |
| `UserMessage` | 用户输入，支持多模态内容 |
| `AssistantMessage` | AI 助手回复，可含工具调用 |
| `ToolCallMessage` | 工具调用结果（Function Calling 场景） |

### 2.3 ManagedStoreInterface

```php
namespace Symfony\AI\Chat;

interface ManagedStoreInterface
{
    /**
     * 初始化存储基础设施（创建表、索引、缓存键等）
     *
     * @param array<mixed> $options 特定后端的扩展选项
     */
    public function setup(array $options = []): void;

    /**
     * 销毁/清空存储（表、集合、键等）
     */
    public function drop(): void;
}
```

所有内置 `MessageStore` 实现均同时实现 `MessageStoreInterface` 和 `ManagedStoreInterface`，而 `Chat` 类构造器要求注入的 store 也必须同时满足这两个接口：

```php
public function __construct(
    private readonly AgentInterface $agent,
    private readonly MessageStoreInterface&ManagedStoreInterface $store,
) {}
```

### 2.4 消息的序列化格式（MessageNormalizer）

`MessageNormalizer` 实现了 Symfony Serializer 的 `NormalizerInterface` 和 `DenormalizerInterface`，将消息序列化为 JSON 可存储的数组结构：

```json
{
    "id": "018e7b9c-3d2f-7000-a1b2-c3d4e5f67890",
    "type": "Symfony\\AI\\Platform\\Message\\UserMessage",
    "content": "",
    "contentAsBase64": [
        {
            "type": "Symfony\\AI\\Platform\\Message\\Content\\Text",
            "content": "你好，请帮我解释一下 PHP 8.2 的新特性。"
        }
    ],
    "toolsCalls": [],
    "metadata": {
        "addedAt": 1735027200
    },
    "addedAt": 1735027200
}
```

AssistantMessage 的序列化示例（含工具调用）：

```json
{
    "id": "018e7b9c-4a1e-7000-b2c3-d4e5f6789abc",
    "type": "Symfony\\AI\\Platform\\Message\\AssistantMessage",
    "content": "好的，我来为您解释...",
    "contentAsBase64": [],
    "toolsCalls": [
        {
            "id": "call_abc123",
            "function": {
                "name": "search_database",
                "arguments": "{\"query\": \"PHP 8.2\"}"
            }
        }
    ],
    "metadata": {
        "usage": {"prompt_tokens": 150, "completion_tokens": 200},
        "addedAt": 1735027201
    },
    "addedAt": 1735027201
}
```

---

## 3. 参数不同带来的结果差异

### 3.1 chatId / 存储键策略

Chat 模块本身没有内置"chatId"参数——不同对话的隔离通过**实例化不同存储对象**（指向不同存储键）来实现。以下是三种主要策略：

#### 策略 A：按用户 ID 分隔（每个用户独立对话历史）

```php
// Symfony 服务配置
$store = new \Symfony\AI\Chat\Bridge\Cache\MessageStore(
    cache: $cachePool,
    cacheKey: 'chat_user_' . $currentUserId,  // 每用户一个键
    ttl: 3600
);
$chat = new Chat($agent, $store);
```

适用场景：客服机器人（每个客服工单一个会话）、个人助手应用

#### 策略 B：按会话 ID 分隔（同一用户可并发多个独立对话）

```php
$sessionId = $request->getSession()->getId();
$store = new \Symfony\AI\Chat\Bridge\Redis\MessageStore(
    redis: $redis,
    indexName: 'chat:session:' . $sessionId
);
```

适用场景：多标签对话页面（如 ChatGPT 多会话窗口）

#### 策略 C：按任务/主题 ID 分隔（特定任务的上下文隔离）

```php
$taskId = $request->attributes->get('taskId');
$store = new \Symfony\AI\Chat\Bridge\Doctrine\DoctrineDbalMessageStore(
    tableName: 'chat_messages_' . $taskId,
    dbalConnection: $connection
);
```

适用场景：代码审查对话（每个 PR 独立上下文）、项目协作（每个任务独立历史）

### 3.2 TTL / 过期时间参数

不同存储后端对 TTL 的支持：

| 存储后端 | TTL 支持 | 默认值 | 配置方式 |
|---|---|---|---|
| Cache | ✅ | 86400 秒（1天） | 构造器参数 `$ttl` |
| Redis | ✅（需手动设置） | 无 | `$redis->expire($key, $ttl)` |
| Session | ✅（PHP Session 生命周期） | PHP 配置 | `session.gc_maxlifetime` |
| Doctrine | ❌（持久化） | 永久 | 业务逻辑清理 |
| MongoDB | ✅（TTL 索引） | 无 | 创建 TTL 索引 |
| InMemory | ❌（进程级） | 进程结束 | — |

Cache 存储的 TTL 效果示例：

```php
// TTL = 3600：对话历史 1 小时后自动过期
$shortTermStore = new Cache\MessageStore($cache, 'chat_quick', ttl: 3600);

// TTL = 86400 * 30：医疗问诊记录保留 30 天
$longTermStore = new Cache\MessageStore($cache, 'chat_medical', ttl: 86400 * 30);

// TTL = 0：禁用过期（注意：部分 PSR-6 实现可能有不同行为）
$permanentStore = new Cache\MessageStore($cache, 'chat_permanent', ttl: 0);
```

### 3.3 持久化 vs 临时存储的影响

```
重启应用后历史消失？
├── 是  →  InMemory\Store（仅用于测试/原型）
│          Session\MessageStore（用户关闭浏览器后消失）
└── 否  →  Cache\MessageStore（依赖缓存后端，若为 APCu 则重启消失）
           Redis\MessageStore（Redis 持久化配置决定）
           Doctrine\MessageStore（数据库持久，完整历史）
           MongoDB\MessageStore（文档持久，灵活扩展）
           Cloudflare\MessageStore（边缘持久，全球访问）
           Meilisearch\MessageStore（支持全文搜索历史）
           SurrealDb\MessageStore（多模型数据库）
           Pogocache\MessageStore（高性能缓存持久化）
```

---

## 4. 实际应用场景（含代码片段）

### 4.1 多用户聊天机器人

为每个用户维护独立的对话历史，支持跨请求的上下文连续性：

```php
class ChatbotController extends AbstractController
{
    public function chat(
        Request $request,
        AgentInterface $agent,
        CacheItemPoolInterface $cache,
    ): JsonResponse {
        $userId = $this->getUser()->getId();

        // 每用户独立的 Cache 存储键
        $store = new \Symfony\AI\Chat\Bridge\Cache\MessageStore(
            cache: $cache,
            cacheKey: sprintf('chatbot_user_%d', $userId),
            ttl: 7200 // 2小时会话超时
        );
        $chat = new \Symfony\AI\Chat\Chat($agent, $store);

        // 若是新用户，初始化系统提示
        if (0 === $store->load()->count()) {
            $chat->initiate(new MessageBag(
                Message::forSystem('你是一个友好的客服助手，专注于解决用户的技术问题。')
            ));
        }

        $userMessage = Message::ofUser($request->getPayload()->getString('message'));
        $response = $chat->submit($userMessage);

        return $this->json(['reply' => $response->getContent()]);
    }
}
```

### 4.2 客服对话历史持久化（Doctrine DBAL）

将对话历史存入关系型数据库，支持审计和追溯：

```php
// config/services.yaml
services:
    customer_service_store:
        class: Symfony\AI\Chat\Bridge\Doctrine\DoctrineDbalMessageStore
        arguments:
            $tableName: 'customer_service_messages'
            $dbalConnection: '@doctrine.dbal.default_connection'

// 使用示例
class CustomerServiceChatService
{
    public function __construct(
        private AgentInterface $agent,
        private DoctrineDbalMessageStore $store,
    ) {}

    public function handleTicket(string $ticketId, string $userInput): string
    {
        // 加载已有工单历史（首次为空）
        $chat = new Chat($this->agent, $this->store);

        // 系统提示包含工单上下文
        if (0 === $this->store->load()->count()) {
            $chat->initiate(new MessageBag(
                Message::forSystem(sprintf(
                    '你是专业的客服代表。当前处理工单 #%s。请保持专业和耐心。',
                    $ticketId
                ))
            ));
        }

        $reply = $chat->submit(Message::ofUser($userInput));
        return $reply->getContent();
    }
}
```

### 4.3 AI 导师学习记录（长期持久化）

学生的学习进度和历史问答记录需要跨课程保存：

```php
class TutorSessionService
{
    public function startLesson(
        Student $student,
        Course $course,
        AgentInterface $tutorAgent,
        \Redis $redis
    ): Chat {
        // Redis 键：学生 + 课程的唯一组合
        $store = new \Symfony\AI\Chat\Bridge\Redis\MessageStore(
            redis: $redis,
            indexName: sprintf('tutor:%d:course:%d', $student->getId(), $course->getId())
        );

        $chat = new Chat($tutorAgent, $store);

        // 仅在新课程开始时初始化
        $history = $store->load();
        if (0 === $history->count()) {
            $chat->initiate(new MessageBag(
                Message::forSystem(sprintf(
                    '你是 %s 课程的 AI 导师。学生姓名：%s，当前学习进度：%d%%。
                    请根据学生的历史问题调整讲解深度，对已掌握的概念简化解释，对薄弱点详细展开。',
                    $course->getName(),
                    $student->getName(),
                    $student->getProgressFor($course)
                ))
            ));
        }

        return $chat;
    }

    public function askQuestion(Chat $chat, string $question): string
    {
        $reply = $chat->submit(Message::ofUser($question));
        return $reply->getContent();
    }
}
```

### 4.4 代码审查对话（按 PR 隔离上下文）

每个 Pull Request 独立维护审查对话，审查者可随时暂停/恢复：

```php
class CodeReviewChatService
{
    public function getOrCreateReviewChat(
        PullRequest $pr,
        AgentInterface $reviewAgent,
        Connection $dbalConn
    ): Chat {
        $store = new \Symfony\AI\Chat\Bridge\Doctrine\DoctrineDbalMessageStore(
            tableName: 'pr_review_chat',
            dbalConnection: $dbalConn
        );

        $chat = new Chat($reviewAgent, $store);

        if (0 === $store->load()->count()) {
            // 将 PR 差异注入系统上下文
            $diff = $pr->getDiff();
            $chat->initiate(new MessageBag(
                Message::forSystem(sprintf(
                    '你是专业的代码审查员。以下是 PR #%d 的代码变更：
                    
                    ```diff
                    %s
                    ```
                    
                    请专注于：代码质量、安全性、性能、可维护性。',
                    $pr->getNumber(),
                    substr($diff, 0, 8000) // 防止超出 token 限制
                ))
            ));
        }

        return $chat;
    }
}
```

### 4.5 会议助手上下文（Session 存储）

浏览器 Session 绑定，无需数据库，用户关闭浏览器自动清理：

```php
class MeetingAssistantController extends AbstractController
{
    public function __construct(
        private AgentInterface $meetingAgent,
        private RequestStack $requestStack,
    ) {}

    #[Route('/meeting/assist', methods: ['POST'])]
    public function assist(Request $request): JsonResponse
    {
        // Session 存储：与浏览器生命周期绑定
        $store = new \Symfony\AI\Chat\Bridge\Session\MessageStore(
            requestStack: $this->requestStack,
            sessionKey: 'meeting_assistant_messages'
        );

        $chat = new Chat($this->meetingAgent, $store);

        // 会议开始时初始化
        if ($request->request->has('meeting_context')) {
            $chat->initiate(new MessageBag(
                Message::forSystem(sprintf(
                    '你是会议助手。会议议题：%s。参与者：%s。
                    帮助记录决策、跟踪行动项、总结讨论要点。',
                    $request->request->get('topic'),
                    $request->request->get('participants')
                ))
            ));
        }

        $reply = $chat->submit(
            Message::ofUser($request->request->getString('input'))
        );

        return $this->json(['response' => $reply->getContent()]);
    }
}
```

### 4.6 电商购物助手（带产品上下文）

结合商品信息的智能购物对话，利用 Redis 的高性能：

```php
class ShoppingAssistantService
{
    public function createShoppingChat(
        Cart $cart,
        CustomerProfile $profile,
        AgentInterface $shoppingAgent,
        \Redis $redis
    ): Chat {
        // 购物会话：购物车 ID 作为隔离键
        $store = new \Symfony\AI\Chat\Bridge\Redis\MessageStore(
            redis: $redis,
            indexName: 'shopping:cart:' . $cart->getId()
        );

        $chat = new Chat($shoppingAgent, $store);

        if (0 === $store->load()->count()) {
            $productList = implode("\n", array_map(
                fn($p) => "- {$p->getName()}（库存：{$p->getStock()}，价格：¥{$p->getPrice()}）",
                $cart->getAvailableProducts()
            ));

            $chat->initiate(new MessageBag(
                Message::forSystem(sprintf(
                    '你是智能购物助手。客户偏好：%s。当前可用商品：
                    %s
                    
                    请根据客户需求提供个性化推荐，并回答商品相关问题。',
                    $profile->getPreferences(),
                    $productList
                ))
            ));
        }

        return $chat;
    }
}
```

### 4.7 医疗问诊记录（MongoDB 长期存储）

医疗场景要求历史永久保存，MongoDB 的灵活文档结构适合存储多模态内容：

```php
class MedicalConsultationService
{
    public function getPatientChat(
        Patient $patient,
        AgentInterface $medicalAgent,
        \MongoDB\Client $mongoClient
    ): Chat {
        // MongoDB 存储：按患者 ID 分隔，永久保存
        $store = new \Symfony\AI\Chat\Bridge\MongoDB\MessageStore(
            client: $mongoClient,
            database: 'medical_records',
            collection: 'consultations_' . $patient->getId()
        );

        $chat = new Chat($medicalAgent, $store);

        if (0 === $store->load()->count()) {
            $medicalHistory = $patient->getMedicalHistory();
            $chat->initiate(new MessageBag(
                Message::forSystem(sprintf(
                    '你是辅助医疗顾问（非替代专业医生）。患者信息：
                    - 姓名：%s，年龄：%d
                    - 过敏史：%s
                    - 慢性病史：%s
                    - 当前用药：%s
                    
                    重要：始终建议患者在做医疗决定前咨询专业医生。',
                    $patient->getName(),
                    $patient->getAge(),
                    $medicalHistory->getAllergies(),
                    $medicalHistory->getChronicConditions(),
                    $medicalHistory->getCurrentMedications()
                ))
            ));
        }

        return $chat;
    }
}
```

### 4.8 协作写作助手（Cloudflare 边缘存储）

全球分布式写作协作，利用 Cloudflare KV 的边缘低延迟优势：

```php
class CollaborativeWritingService
{
    public function getDocumentChat(
        Document $document,
        AgentInterface $writingAgent,
        CloudflareClient $cfClient
    ): Chat {
        // Cloudflare KV：全球边缘访问，低延迟
        $store = new \Symfony\AI\Chat\Bridge\Cloudflare\MessageStore(
            client: $cfClient,
            namespace: 'writing_chats',
            key: 'doc_' . $document->getId()
        );

        $chat = new Chat($writingAgent, $store);

        if (0 === $store->load()->count()) {
            $chat->initiate(new MessageBag(
                Message::forSystem(sprintf(
                    '你是协作写作助手。当前文档：《%s》
                    风格指南：%s
                    目标读者：%s
                    
                    请帮助完善内容、改进表达、保持风格一致性。
                    当前草稿摘要：%s',
                    $document->getTitle(),
                    $document->getStyleGuide(),
                    $document->getTargetAudience(),
                    substr($document->getContent(), 0, 2000)
                ))
            ));
        }

        return $chat;
    }
}
```

---

## 5. 各存储后端对比

### 5.1 完整对比表

| 存储后端 | 适用场景 | 性能 | 持久化 | 水平扩展 | 搜索能力 | 全局分发 | 依赖复杂度 |
|---|---|---|---|---|---|---|---|
| **InMemory** | 测试、原型、单元测试 | ⭐⭐⭐⭐⭐ | ❌ | ❌ | ❌ | ❌ | 极低 |
| **Session** | 浏览器绑定对话、无登录场景 | ⭐⭐⭐⭐ | 部分 | ❌ | ❌ | ❌ | 低 |
| **Cache** | 短期对话、高并发场景 | ⭐⭐⭐⭐ | 依赖后端 | ✅ | ❌ | ❌ | 低 |
| **Redis** | 高性能、需要 TTL 精控 | ⭐⭐⭐⭐⭐ | ✅ | ✅ | ❌ | ❌ | 中 |
| **Doctrine** | 企业应用、需要 SQL 查询历史 | ⭐⭐⭐ | ✅ | ✅ | ❌ | ❌ | 中 |
| **MongoDB** | 多模态内容、灵活结构 | ⭐⭐⭐⭐ | ✅ | ✅ | 部分 | ❌ | 中 |
| **Meilisearch** | 需要全文搜索对话历史 | ⭐⭐⭐ | ✅ | ✅ | ⭐⭐⭐⭐⭐ | ❌ | 高 |
| **Cloudflare** | 全球分布式、边缘低延迟 | ⭐⭐⭐⭐⭐ | ✅ | ✅ | ❌ | ✅ | 高 |
| **Pogocache** | 高性能分布式缓存 | ⭐⭐⭐⭐⭐ | ✅ | ✅ | ❌ | ❌ | 高 |
| **SurrealDB** | 多模型（图/文档/关系）查询 | ⭐⭐⭐⭐ | ✅ | ✅ | ✅ | ❌ | 高 |

### 5.2 各后端详细说明

#### InMemory Store

```php
use Symfony\AI\Chat\InMemory\Store;

// 默认实例：所有请求共享同一键（适合单测）
$store = new Store();

// 自定义标识符（同进程中隔离不同 chat 实例）
$store = new Store('user_42_session');

// 实现了 ResetInterface，可配合 Symfony 的 kernel.reset 事件自动清理
```

**特点：**
- 无需任何外部依赖
- 实现了 `Symfony\Contracts\Service\ResetInterface`
- `reset()` 方法清空所有存储键（`$this->messages = []`）
- 适合 PHPUnit 测试中模拟对话状态

#### Cache Store

```php
use Symfony\AI\Chat\Bridge\Cache\MessageStore;
use Symfony\Component\Cache\Adapter\RedisAdapter;

$cache = new RedisAdapter($redis);
$store = new MessageStore(
    cache: $cache,
    cacheKey: 'chat_' . $userId,
    ttl: 3600  // 1小时 TTL
);

// setup() 会预创建缓存键，设置初始空 MessageBag
$store->setup();

// drop() 调用 $cache->deleteItem($cacheKey)
$store->drop();
```

**关键实现细节：**
- `save()` 和 `setup()` 都调用 `$item->expiresAfter($this->ttl)`，每次写入刷新 TTL
- `load()` 通过 `$item->isHit()` 判断键是否存在，不存在时返回空 `MessageBag`
- 支持 PSR-6 的任意缓存后端（Redis、Memcached、APCu、Filesystem 等）

#### Redis Store

```php
use Symfony\AI\Chat\Bridge\Redis\MessageStore;
use Symfony\AI\Chat\MessageNormalizer;
use Symfony\Component\Serializer\Encoder\JsonEncoder;
use Symfony\Component\Serializer\Normalizer\ArrayDenormalizer;
use Symfony\Component\Serializer\Serializer;

$redis = new \Redis();
$redis->connect('127.0.0.1', 6379);

$store = new MessageStore(
    redis: $redis,
    indexName: 'chat:session:abc123',
    // 可注入自定义 Serializer
    serializer: new Serializer(
        [new ArrayDenormalizer(), new MessageNormalizer()],
        [new JsonEncoder()]
    )
);
```

**关键实现细节：**
- 使用 `$redis->set()` 和 `$redis->get()` 操作
- 整个 MessageBag 序列化为 JSON 字符串存储
- `drop()` 将键值设为空数组 `[]` 而非删除键（保留键的存在）
- `setup()` 仅在键不存在时创建（`$redis->exists()` 检查）

#### Doctrine DBAL Store

```php
use Symfony\AI\Chat\Bridge\Doctrine\DoctrineDbalMessageStore;

$store = new DoctrineDbalMessageStore(
    tableName: 'chat_messages',
    dbalConnection: $connection
);

// setup() 调用 introspectSchema() 检查表是否存在，不存在则创建
// 表结构：id (BIGINT AUTO_INCREMENT), messages (TEXT), added_at (INTEGER)
$store->setup();
```

**数据库表结构：**

```sql
CREATE TABLE chat_messages (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    messages TEXT NOT NULL,      -- JSON 序列化的消息数组
    added_at INTEGER NOT NULL    -- Unix 时间戳
);
```

**关键实现细节：**
- 每次 `save()` 都执行 `INSERT`（追加模式，不更新）
- `load()` 按 `added_at ASC` 排序，合并所有历史记录
- `drop()` 执行 `DELETE` 清空所有行（保留表结构）
- 支持 Oracle 数据库的序列化自增 ID 特殊处理
- `setup()` 不接受任何 `$options`（传入会抛出 `InvalidArgumentException`）

#### Session Store

```php
use Symfony\AI\Chat\Bridge\Session\MessageStore;
use Symfony\Component\HttpFoundation\RequestStack;

$store = new MessageStore(
    requestStack: $requestStack,
    sessionKey: 'ai_chat_messages'  // 默认 'messages'
);
```

**关键实现细节：**
- 直接存储 `MessageBag` 对象到 PHP Session（需 Session 序列化支持）
- `drop()` 调用 `$this->session->remove($this->sessionKey)`
- 与用户浏览器会话绑定，天然隔离不同用户
- 不适合长时间保存（Session 过期时历史丢失）

---

## 6. 与 Agent 集成模式

### 6.1 基础集成

Chat 模块通过 `AgentInterface` 连接到 AI Platform：

```php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Platform\Bridge\OpenAI\Platform;
use Symfony\AI\Platform\Bridge\OpenAI\GPT;

// 构建 Platform（与 OpenAI API 通信）
$platform = new Platform($httpClient, $apiKey);

// 构建 Agent（可添加工具、预处理器、后处理器）
$agent = new Agent($platform, new GPT(GPT::GPT_4O_MINI));

// 构建 MessageStore
$store = new \Symfony\AI\Chat\InMemory\Store();

// 构建 Chat（连接 Agent 和 Store）
$chat = new Chat($agent, $store);
```

### 6.2 带 RAG 的集成（检索增强生成）

```php
use Symfony\AI\Agent\Toolbox\Tool;
use Symfony\AI\Store\Bridge\Pinecone\VectorStore;

// 配置带向量搜索工具的 Agent
$agent = new Agent(
    platform: $platform,
    model: new GPT(GPT::GPT_4O),
    tools: [new KnowledgeBaseTool($vectorStore)]
);

$store = new \Symfony\AI\Chat\Bridge\Redis\MessageStore($redis, 'rag_chat:user:42');
$chat = new Chat($agent, $store);

// 初始化时注入 RAG 系统提示
$chat->initiate(new MessageBag(
    Message::forSystem(
        '你是知识库助手。当需要查找具体信息时，使用 search_knowledge_base 工具。'
    )
));
```

### 6.3 Symfony 服务容器集成（AI Bundle）

在使用 `symfony/ai-bundle` 时，Chat 可通过 DI 配置：

```yaml
# config/packages/ai.yaml
ai:
    chat:
        default:
            agent: default
            store: cache
    message_store:
        cache:
            type: cache
            cache_pool: cache.app
            ttl: 3600
        redis:
            type: redis
            redis: snc_redis.default
            key_prefix: 'chat_'
        doctrine:
            type: doctrine
            table_name: 'ai_chat_messages'
```

```php
// Controller 注入
class ChatController extends AbstractController
{
    public function __construct(
        #[AutowireDecorated]
        private ChatInterface $chat,
    ) {}
}
```

---

## 7. 消息序列化格式

### 7.1 完整消息类型的序列化结构

**SystemMessage：**

```json
{
    "id": "018e7b9c-1234-7000-aaaa-bbbbccccdddd",
    "type": "Symfony\\AI\\Platform\\Message\\SystemMessage",
    "content": "你是一个专业的技术助手。",
    "contentAsBase64": [],
    "toolsCalls": [],
    "metadata": {},
    "addedAt": 1735027200
}
```

**UserMessage（多模态：文本 + 图片）：**

```json
{
    "id": "018e7b9c-5678-7000-bbbb-ccccddddeeee",
    "type": "Symfony\\AI\\Platform\\Message\\UserMessage",
    "content": "",
    "contentAsBase64": [
        {
            "type": "Symfony\\AI\\Platform\\Message\\Content\\Text",
            "content": "请描述这张图片中的内容"
        },
        {
            "type": "Symfony\\AI\\Platform\\Message\\Content\\Image",
            "content": "data:image/jpeg;base64,/9j/4AAQSkZJRgAB..."
        }
    ],
    "toolsCalls": [],
    "metadata": {},
    "addedAt": 1735027201
}
```

**ToolCallMessage（工具调用结果）：**

```json
{
    "id": "018e7b9c-9abc-7000-cccc-ddddeeeefffff",
    "type": "Symfony\\AI\\Platform\\Message\\ToolCallMessage",
    "content": "{\"results\": [{\"title\": \"PHP 8.2 新特性\", \"url\": \"...\"}]}",
    "contentAsBase64": [],
    "toolsCalls": {
        "id": "call_abc123",
        "function": {
            "name": "web_search",
            "arguments": "{\"query\": \"PHP 8.2 新特性\"}"
        }
    },
    "metadata": {},
    "addedAt": 1735027202
}
```

### 7.2 MessageNormalizer 支持的内容类型

| PHP 类 | JSON `type` 字段 | 内容存储方式 |
|---|---|---|
| `Text` | `...Content\Text` | `content` 字段（纯文本） |
| `Image` | `...Content\Image` | `content` 字段（Base64 Data URL） |
| `Audio` | `...Content\Audio` | `content` 字段（Base64 Data URL） |
| `File` | `...Content\File` | `content` 字段（Base64 Data URL） |
| `Document` | `...Content\Document` | `content` 字段（Base64 Data URL） |
| `ImageUrl` | `...Content\ImageUrl` | `content` 字段（URL 字符串） |
| `DocumentUrl` | `...Content\DocumentUrl` | `content` 字段（URL 字符串） |

### 7.3 CLI 命令使用

```bash
# 初始化 Doctrine 存储的数据库表
php bin/console ai:message-store:setup my_store

# 清空（删除所有数据）指定存储
php bin/console ai:message-store:drop my_store --force

# Tab 补全：自动列出所有已注册的 store 名称
php bin/console ai:message-store:setup <TAB>
```

---

## 8. 异常体系

Chat 模块定义了独立的异常层级，避免与全局 PHP 异常混用：

```
ExceptionInterface (Throwable)
├── InvalidArgumentException  → 无效参数（如 Doctrine 传入不支持的 options）
├── LogicException            → 逻辑错误（如 MessageNormalizer 遇到未知消息类型）
└── RuntimeException          → 运行时错误（如 CLI 命令执行失败）
```

```php
// 使用项目特定异常而非全局异常
try {
    $chat->submit($message);
} catch (\Symfony\AI\Agent\Exception\ExceptionInterface $e) {
    // Agent 层错误（API 调用失败、超时等）
    $logger->error('AI agent error', ['exception' => $e]);
} catch (\Symfony\AI\Chat\Exception\RuntimeException $e) {
    // Chat/Store 层错误
    $logger->error('Chat store error', ['exception' => $e]);
}
```

---

## 9. 总结

Chat 模块以简洁的接口设计（仅 `initiate()` + `submit()`）封装了多轮对话的完整状态管理逻辑：

1. **存储抽象层**：通过 `MessageStoreInterface` + `ManagedStoreInterface` 双接口，实现与具体存储技术的解耦
2. **10 种存储后端**：覆盖从内存到数据库、从本地到云端的全部场景
3. **JSON 序列化**：`MessageNormalizer` 支持所有消息类型（含多模态内容）的完整序列化
4. **对话隔离**：通过不同存储键实现多用户/多会话/多任务的上下文隔离
5. **生命周期管理**：CLI 命令支持基础设施的初始化和清理
