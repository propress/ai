# 多轮对话与会话持久化

## 业务场景

你正在开发一个在线客服系统。用户通过网页与 AI 客服对话，关闭页面后再打开，仍然能看到之前的对话并继续聊天。不同用户之间的对话互相隔离。

**典型应用：** 在线客服系统、咨询助手、教育辅导对话、医疗问诊预分诊、技术支持工单

## 涉及模块

| 模块 | 包名 | 用途 |
|------|------|------|
| **Platform** | `symfony/ai-platform` | 核心抽象层——定义 `PlatformInterface`、消息系统和结果系统 |
| **Bridge（OpenAI）** | `symfony/ai-open-ai-platform` | 本教程的主实现，提供 OpenAI 的 `PlatformFactory` |
| **Agent** | `symfony/ai-agent` | 封装 AI 调用逻辑，作为 `Chat` 的后端引擎。支持 `InputProcessor`/`OutputProcessor` 管线 |
| **Chat** | `symfony/ai-chat` | 对话管理器，自动处理消息历史的加载、追加和持久化 |
| **Chat Store Bridge** | 如 `symfony/ai-redis-message-store` | 消息持久化后端。内置 InMemory，可选 Redis / DBAL / MongoDB / Session / Cache 等 |

## 架构概述

在 [01-basic-chatbot.md](./01-basic-chatbot.md) 中，我们手动维护对话历史数组。`Chat` 模块自动化了这个过程——它在每次用户提交消息时自动加载历史、调用 Agent、保存结果。

Chat 依赖三个核心组件：

- **`ChatInterface`** 定义了 `initiate()` 和 `submit()` 两个方法——分别用于初始化对话和提交新消息
- **`AgentInterface`** 是 AI 调用引擎——它接收 `MessageBag` 并返回 `ResultInterface`（通常是 `TextResult`）
- **`MessageStoreInterface` & `ManagedStoreInterface`** 是持久化接口——前者负责 `save()` / `load()`，后者负责 `setup()` / `drop()`

`Chat` 构造函数要求 Store 同时实现两个接口（交叉类型 `MessageStoreInterface & ManagedStoreInterface`），所有内置 Store 实现都满足这一要求。

## 项目流程图

```
┌──────────┐     ┌──────────────┐     ┌─────────────────────┐     ┌─────────────┐
│  用户输入  │ ──▶│   Chat       │ ──▶│  Agent              │ ──▶│  Platform    │
│ (文本消息) │     │  submit()   │     │  call(MessageBag)   │     │  invoke()   │
└──────────┘     └──────┬───────┘     └──────────┬──────────┘     └──────┬──────┘
                        │                         │                       │
                 ┌──────▼───────┐          ┌──────▼──────────┐     ┌─────▼──────┐
                 │ MessageStore │          │ InputProcessors │     │  OpenAI    │
                 │ load() 加载   │          │ OutputProcessors│     │  API       │
                 │ save() 保存   │          │ （可选管线）      │     │            │
                 └──────────────┘          └─────────────────┘     └─────┬──────┘
                                                                         │
┌──────────┐     ┌──────────────┐     ┌─────────────────────┐           │
│  显示回复  │ ◀──│ Assistant    │ ◀──│  TextResult         │ ◀────────┘
│ (前端渲染) │     │ Message     │     │  (结果 + Metadata)   │
└──────────┘     └──────────────┘     └─────────────────────┘
```

**核心类的关系：**

```
Chat（对话管理器）
 ├── Agent（AI 调用引擎）
 │    ├── Platform（AI 平台连接，如 OpenAI / Anthropic / Gemini）
 │    ├── InputProcessor[]（输入处理管线，可选）
 │    └── OutputProcessor[]（输出处理管线，可选）
 └── MessageStore（消息持久化，实现 MessageStoreInterface & ManagedStoreInterface）
      └── 内置实现：InMemory / Redis / DBAL / MongoDB / Session / Cache / Cloudflare / SurrealDB / Meilisearch 等
```

---

## 前置准备

### 环境要求

- PHP >= 8.2
- Composer

### 安装依赖

```bash
# 核心依赖
composer require symfony/ai-platform symfony/ai-open-ai-platform
composer require symfony/ai-agent
composer require symfony/ai-chat
```

`symfony/ai-chat` 已内置 `InMemory\Store`，无需安装额外 Bridge 即可开始开发。根据生产需要，再安装对应的存储 Bridge：

```bash
# 存储后端 Bridge（按需安装）
composer require symfony/ai-redis-message-store         # Redis
composer require symfony/ai-doctrine-message-store      # MySQL / PostgreSQL（Doctrine DBAL）
composer require symfony/ai-mongo-db-message-store      # MongoDB
composer require symfony/ai-session-message-store       # Symfony Session
composer require symfony/ai-cache-message-store         # PSR-6 Cache
composer require symfony/ai-cloudflare-message-store    # Cloudflare Workers KV
composer require symfony/ai-surrealdb-message-store     # SurrealDB
composer require symfony/ai-meilisearch-message-store   # Meilisearch
```

### 设置 API 密钥

```bash
export OPENAI_API_KEY="sk-your-api-key-here"
```

> **🔒 安全建议：** 不要将 API 密钥硬编码在代码中。在 Symfony 项目中使用 `.env.local` 文件，在生产环境中使用 Symfony Secrets 或环境变量管理服务（如 Vault、AWS Secrets Manager）来管理敏感配置。

---

## Step 1：Chat 的对话生命周期

Chat 管理的对话有两个阶段：

1. **初始化阶段** — 调用 `initiate(MessageBag)`，清空存储并保存初始消息（通常是系统提示）
2. **对话阶段** — 反复调用 `submit(UserMessage)`，每次自动加载历史、追加新消息、调用 AI、保存结果

```
initiate(系统提示)          // 创建新对话，清空旧数据
    │
    ▼
submit(用户消息) ──▶ AI 回复  // 第 1 轮
    │
    ▼
submit(用户消息) ──▶ AI 回复  // 第 2 轮（自动携带第 1 轮上下文）
    │
    ▼
submit(用户消息) ──▶ AI 回复  // 第 3 轮（自动携带所有历史）
    ...
```

### Chat 的内部实现

`Chat::submit()` 每次调用时执行以下步骤：

```php
// 简化后的 submit() 内部逻辑
public function submit(UserMessage $message): AssistantMessage
{
    $messages = $this->store->load();          // 1. 加载历史消息
    $messages->add($message);                  // 2. 追加用户消息

    $result = $this->agent->call($messages);   // 3. 调用 Agent（AI 推理）

    $assistantMessage = Message::ofAssistant($result->getContent());
    $assistantMessage->getMetadata()->merge($result->getMetadata());

    $messages->add($assistantMessage);         // 4. 追加 AI 回复
    $this->store->save($messages);             // 5. 保存完整历史

    return $assistantMessage;                  // 6. 返回 AssistantMessage
}
```

> **📝 知识扩展：** `submit()` 返回的 `AssistantMessage` 会自动从 `TextResult` 继承 `Metadata`（含 Token 用量等信息）。你可以通过 `$response->getMetadata()->get('token_usage')` 获取本轮的 Token 消耗。

### 核心接口

| 接口 | 方法 | 说明 |
|------|------|------|
| `MessageStoreInterface` | `save(MessageBag)` | 将完整消息列表保存到存储 |
| | `load(): MessageBag` | 从存储加载完整消息列表 |
| `ManagedStoreInterface` | `setup(array $options)` | 初始化存储（建表、建索引等） |
| | `drop()` | 清空存储中的所有消息 |

---

## Step 2：使用内存存储的基础对话

先用内存存储来理解 Chat 的工作方式。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\InMemory\Store as InMemoryStore;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 1. 创建 Platform 和 Agent
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);
$agent = new Agent($platform, 'gpt-4o-mini');

// 2. 创建 Chat（使用内存存储）
$chat = new Chat($agent, new InMemoryStore());

// 3. 初始化对话：设置系统提示
$chat->initiate(new MessageBag(
    Message::forSystem('你是一个在线英语老师。用简单的例句帮助用户学英语。每次只教一个知识点。'),
));

// 4. 第一轮对话
$response = $chat->submit(Message::ofUser('我想学习如何用英语自我介绍'));
echo "老师：" . $response->getContent() . "\n\n";

// 5. 第二轮对话 —— Chat 自动携带上一轮的上下文
$response = $chat->submit(Message::ofUser('能给我一个更正式的版本吗？'));
echo "老师：" . $response->getContent() . "\n\n";

// 6. 第三轮对话 —— AI 知道之前的所有内容
$response = $chat->submit(Message::ofUser('如果是在面试中自我介绍呢？'));
echo "老师：" . $response->getContent() . "\n\n";
```

> **⚠️ 注意：** `InMemoryStore` 的数据仅存在于当前进程的内存中。进程结束后数据丢失，不适合跨请求的 Web 场景。适合脚本、测试和开发环境。`InMemoryStore` 还实现了 `ResetInterface`，在 Symfony 的 `kernel.reset` 事件中会自动清空所有对话。

### Agent 的构造函数

`Agent` 是 Chat 的后端引擎，它不仅仅是一个简单的 Platform 调用包装：

```php
$agent = new Agent(
    platform: $platform,           // 必需：AI 平台
    model: 'gpt-4o-mini',          // 必需：模型名称
    inputProcessors: [],           // 可选：输入处理管线
    outputProcessors: [],          // 可选：输出处理管线
    name: 'customer-service',      // 可选：Agent 名称标识
);
```

`inputProcessors` 和 `outputProcessors` 在工具调用（第 03 篇）和记忆注入（第 08 篇）场景中会用到。对于纯对话场景，保持默认空数组即可。

---

## Step 3：对话隔离——支持多用户

> **⚠️ 注意：** `Chat` 类的 `initiate()` 和 `submit()` 方法**没有** `conversationId` 参数。每个 `Chat` 实例绑定一个 `MessageStore` 实例，一个 Store 实例管理一个对话。

要支持多个对话（如不同用户），需要为每个对话创建独立的 Store 实例：

```php
// 用户 A 的对话 —— 使用独立的 Store 实例
$storeA = new InMemoryStore('conversation:user-a');
$chatA  = new Chat($agent, $storeA);

// 用户 B 的对话 —— 使用不同标识的 Store 实例
$storeB = new InMemoryStore('conversation:user-b');
$chatB  = new Chat($agent, $storeB);
```

各存储后端的隔离参数：

| Store | 构造参数 | 隔离方式 | 示例 |
|-------|---------|---------|------|
| `InMemory\Store` | `$identifier` | 内存 key | `new Store('conv-123')` |
| `Redis\MessageStore` | `$indexName` | Redis key | `new MessageStore($redis, 'conv-123')` |
| `MongoDb\MessageStore` | `$collectionName` | 集合名 | `new MessageStore($client, 'mydb', 'conv-123')` |
| `Doctrine\DoctrineDbalMessageStore` | `$tableName` | 表名 | `new DoctrineDbalMessageStore('conv_123', $conn)` |
| `Cache\MessageStore` | `$cacheKey` | 缓存 key | `new MessageStore($cache, 'conv-123')` |
| `Session\MessageStore` | `$sessionKey` | Session key | `new MessageStore($stack, 'conv-123')` |
| `Cloudflare\MessageStore` | `$namespace` | KV Namespace | `new MessageStore($http, 'conv-123', $acct, $key)` |
| `SurrealDb\MessageStore` | `$table` | 表名 | `new MessageStore($http, $database, 'conv-123', $ns, $user, $pass)` |

> **🔒 安全建议：** `conversationId` 不要使用可预测的值（如自增 ID）。建议使用 UUID 或加密 token，防止用户篡改 ID 访问他人对话（IDOR 漏洞）。

---

## Step 4：使用 Redis 实现跨请求持久化

在 Web 应用中，每个 HTTP 请求是独立的进程。我们需要将对话存入 Redis，这样跨请求也能保持对话上下文。

```bash
composer require symfony/ai-redis-message-store
```

### Redis Store 的构造

```php
use Symfony\AI\Chat\Bridge\Redis\MessageStore as RedisMessageStore;

$redis = new \Redis();
$redis->connect('127.0.0.1', 6379);

// Redis Store 使用 Symfony Serializer 自动序列化/反序列化消息
$store = new RedisMessageStore(
    redis: $redis,                  // 必需：Redis 连接
    indexName: $conversationId,     // 必需：Redis key，用于隔离对话
    // serializer: ...              // 可选：自定义 Serializer
);
```

> **📝 知识扩展：** 所有需要序列化的 Store（Redis、MongoDB、Doctrine DBAL、Cloudflare、SurrealDB）内部使用 `MessageNormalizer`——一个 Symfony Serializer normalizer，负责将 `SystemMessage`、`UserMessage`、`AssistantMessage`、`ToolCallMessage` 转换为 JSON。每条消息存储时会自动保留其 UUID v7 标识符（`getId()`）、内容、工具调用、元数据和时间戳。反序列化时通过 `withId()` 恢复原始 UUID，确保消息跨请求的一致性。

### 模拟 Web 场景：跨请求的对话

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\Bridge\Redis\MessageStore as RedisMessageStore;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);
$agent = new Agent($platform, 'gpt-4o-mini');

$redis = new \Redis();
$redis->connect('127.0.0.1', 6379);

// ============================================
// 模拟第一个 HTTP 请求：用户开始新对话
// ============================================
$conversationId = 'user-123-conv-456';

$store = new RedisMessageStore($redis, $conversationId);
$chat = new Chat($agent, $store);

// 初始化对话：设置系统提示
$chat->initiate(new MessageBag(
    Message::forSystem('你是一个旅行顾问。帮用户规划旅行，给出具体的行程建议。'),
));

$response = $chat->submit(Message::ofUser('我想去日本玩 5 天，预算 1 万元'));
echo "顾问：" . $response->getContent() . "\n";
// 请求结束，对话已保存到 Redis

// ============================================
// 模拟第二个 HTTP 请求：用户回来继续对话
// ============================================
// 用相同的 conversationId 重建 Store，Chat 自动从 Redis 加载历史
$store = new RedisMessageStore($redis, $conversationId);
$chat = new Chat($agent, $store);

// 不需要再次 initiate —— 历史消息已在 Redis 中
$response = $chat->submit(
    Message::ofUser('我对京都特别感兴趣，可以多安排一些时间吗？'),
);
echo "顾问：" . $response->getContent() . "\n";

// ============================================
// 模拟第三个 HTTP 请求
// ============================================
$store = new RedisMessageStore($redis, $conversationId);
$chat = new Chat($agent, $store);

$response = $chat->submit(Message::ofUser('京都有什么特别推荐的美食吗？'));
echo "顾问：" . $response->getContent() . "\n";
```

> **🏭 生产建议：** Redis 适合对话量大、响应速度要求高的场景。建议通过 Redis 的 `EXPIRE` 命令或在应用层设置 TTL 来自动清理过期对话数据，避免内存无限增长。

---

## Step 5：使用 Doctrine DBAL 存入关系型数据库

如果你的应用使用 MySQL 或 PostgreSQL，可以将对话存入数据库表。Doctrine DBAL Store 自动处理建表和 SQL 查询。

```bash
composer require symfony/ai-doctrine-message-store
```

```php
<?php

use Doctrine\DBAL\DriverManager;
use Symfony\AI\Agent\Agent;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\Bridge\Doctrine\DoctrineDbalMessageStore;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);
$agent = new Agent($platform, 'gpt-4o-mini');

// 创建数据库连接
$connection = DriverManager::getConnection([
    'url' => 'mysql://user:password@127.0.0.1:3306/myapp',
]);

$conversationId = 'user-123-conv-456';

// Doctrine DBAL Store 使用表名来隔离对话
$store = new DoctrineDbalMessageStore(
    tableName: $conversationId,      // 必需：表名（也用于对话隔离）
    dbalConnection: $connection,     // 必需：数据库连接
    // serializer: ...               // 可选：自定义 Serializer
    // clock: ...                    // 可选：ClockInterface（用于时间戳）
);

// 初始化存储（首次使用时自动建表）
$store->setup();

$chat = new Chat($agent, $store);

$chat->initiate(new MessageBag(
    Message::forSystem('你是一个专业的法律咨询助手。提供一般性法律信息，但始终建议咨询正式律师。'),
));

$response = $chat->submit(Message::ofUser('租房合同到期，房东不退押金怎么办？'));
echo "律师助手：" . $response->getContent() . "\n";
```

> **💡 提示：** `DoctrineDbalMessageStore` 的 `setup()` 方法自动创建包含 `id`（自增主键）、`messages`（TEXT）和 `added_at`（INTEGER 时间戳）字段的表。它还内置了 Oracle 数据库的序列支持。

> **⚠️ 注意：** Doctrine DBAL Store 使用表名来隔离对话，因此每个对话对应一张表。对于海量对话场景，请考虑使用 MongoDB 或 Redis 等更适合的方案。

---

## Step 6：使用 MongoDB 存储

MongoDB 的文档模型天然适合存储对话数据。每条消息作为一个文档存入集合中。

```bash
composer require symfony/ai-mongo-db-message-store
```

```php
<?php

require 'vendor/autoload.php';

use MongoDB\Client;
use Symfony\AI\Agent\Agent;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\Bridge\MongoDb\MessageStore as MongoMessageStore;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);
$agent = new Agent($platform, 'gpt-4o-mini');

$mongoClient = new Client('mongodb://127.0.0.1:27017');

$conversationId = 'user-123-conv-456';

// MongoDB Store 使用 collectionName 来隔离对话
$store = new MongoMessageStore(
    client: $mongoClient,               // 必需：MongoDB 客户端
    databaseName: 'myapp_chat',          // 必需：数据库名
    collectionName: $conversationId,     // 必需：集合名（用于隔离）
);

$store->setup();

$chat = new Chat($agent, $store);

$chat->initiate(new MessageBag(
    Message::forSystem('你是一个专业的健康咨询师。提供一般性健康建议，但始终建议用户就医。'),
));

$response = $chat->submit(Message::ofUser('最近总是感觉很累，是什么原因？'));
echo "咨询师：" . $response->getContent() . "\n";

// 后续请求中，用相同的参数重建 Store 即可继续对话
$store = new MongoMessageStore($mongoClient, 'myapp_chat', $conversationId);
$chat = new Chat($agent, $store);

$response = $chat->submit(Message::ofUser('我的作息时间比较不规律，这有关系吗？'));
echo "咨询师：" . $response->getContent() . "\n";
```

> **📝 知识扩展：** MongoDB Store 的序列化使用 `_id` 作为消息标识字段（而非默认的 `id`），这是通过 normalizer 的 context 参数 `['identifier' => '_id']` 实现的——符合 MongoDB 的文档标识符约定。

> **🏭 生产建议：** MongoDB 适合消息结构可能变化、需要灵活查询的场景。它的文档模型天然适配对话数据，且支持水平扩展（分片），适合大规模应用。

---

## Step 7：使用 Symfony Session 和 PSR-6 Cache 存储

### Session 存储

在 Symfony 框架中，可以直接利用 Session 组件存储对话。适合简单场景，无需额外数据库。

```bash
composer require symfony/ai-session-message-store
```

```php
<?php

use Symfony\AI\Chat\Bridge\Session\MessageStore as SessionMessageStore;
use Symfony\Component\HttpFoundation\RequestStack;

// 在 Symfony Controller 中
public function chatAction(RequestStack $requestStack): Response
{
    $store = new SessionMessageStore(
        requestStack: $requestStack,     // 必需：Symfony RequestStack
        sessionKey: 'my_chat',           // 可选：Session key（默认 'messages'）
    );
    $chat = new Chat($agent, $store);

    // 不同用户的 Session 天然隔离，无需手动管理对话 ID
    $response = $chat->submit(Message::ofUser('你好'));

    return new JsonResponse(['reply' => $response->getContent()]);
}
```

> **⚠️ 注意：** Session 存储受限于 Session 的大小和生命周期。对话过长时可能导致 Session 数据膨胀。适合原型开发和轻量级应用，不建议用于高并发生产环境。

### PSR-6 Cache 存储

如果你的应用已有 PSR-6 缓存基础设施（如 Redis、Filesystem、APCu 等），可以直接复用：

```bash
composer require symfony/ai-cache-message-store
```

```php
use Symfony\AI\Chat\Bridge\Cache\MessageStore as CacheMessageStore;
use Symfony\Component\Cache\Adapter\FilesystemAdapter;

$cache = new FilesystemAdapter();
$store = new CacheMessageStore(
    cache: $cache,                // 必需：PSR-6 CacheItemPoolInterface
    cacheKey: 'conv-123',         // 可选：缓存 key（默认 '_message_store_cache'）
    ttl: 86400,                   // 可选：过期时间，秒（默认 86400 = 24 小时）
);
$chat = new Chat($agent, $store);
```

> **💡 提示：** Cache Store 的 TTL 参数让对话自动过期——非常适合临时对话场景（如表单助手、一次性咨询）。过期后用户需要重新开始对话。

---

## Step 8：追踪 Token 用量

每轮对话都会将完整历史发送给 AI，Token 消耗随对话轮数线性增长。`submit()` 返回的 `AssistantMessage` 会继承 AI 调用的 `Metadata`，你可以从中获取 Token 用量：

```php
$response = $chat->submit(Message::ofUser('请推荐一些适合初学者的景点'));

echo "回复：" . $response->getContent() . "\n";

// 获取本轮 Token 用量
$tokenUsage = $response->getMetadata()->get('token_usage');

if (null !== $tokenUsage) {
    echo sprintf(
        "[Token] 输入: %d, 输出: %d, 合计: %d\n",
        $tokenUsage->getPromptTokens(),
        $tokenUsage->getCompletionTokens(),
        $tokenUsage->getTotalTokens(),
    );
}
```

> **⚠️ 注意：** 10 轮深度对话的输入 Token 可能达到数千甚至上万。在生产环境中，务必设置对话轮数上限或实现消息截断策略来控制成本。

---

## 完整示例：多轮教育辅导对话

以下是一个整合所有概念的完整示例——使用 Anthropic Claude 实现，展示平台切换的便捷性：

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\InMemory\Store as InMemoryStore;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 使用 Anthropic Claude（展示平台切换——只需改这两行）
$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY']);
$agent = new Agent($platform, 'claude-sonnet-4-20250514', name: 'math-tutor');

$chat = new Chat($agent, new InMemoryStore());

// 设置系统提示：定义 AI 的教学风格
$chat->initiate(new MessageBag(
    Message::forSystem(
        '你是一个耐心的数学辅导老师，面向初中学生。'
        . '教学原则：1) 先问学生哪里不懂；2) 用简单的例子解释；3) 每次只讲一个概念；'
        . '4) 讲完后给学生一道练习题检验理解；5) 如果学生做错了，鼓励并引导纠正。'
    ),
));

// 模拟完整的教学对话流程
$dialogue = [
    '老师你好，我不太懂一元二次方程怎么解',
    '因式分解是什么意思？',
    '哦，我懂了。那 x² + 5x + 6 = 0 怎么用因式分解？',
    '答案是 x = -2 和 x = -3 对吗？',
    '太好了！能再给我一道题练习吗？',
];

echo "=== 数学辅导对话 ===\n\n";

$totalTokens = 0;

foreach ($dialogue as $question) {
    echo "学生：{$question}\n\n";

    $response = $chat->submit(Message::ofUser($question));
    echo "老师：" . $response->getContent() . "\n";

    // 追踪 Token 用量
    $tokenUsage = $response->getMetadata()->get('token_usage');
    if (null !== $tokenUsage) {
        $totalTokens += $tokenUsage->getTotalTokens();
        echo sprintf(
            "  [Token: %d | 累计: %d]\n",
            $tokenUsage->getTotalTokens(),
            $totalTokens,
        );
    }

    echo "\n" . str_repeat('-', 50) . "\n\n";
}

echo sprintf("=== 对话结束，共消耗 %d Tokens ===\n", $totalTokens);
```

> **💡 提示：** 注意对比完整示例与前面的步骤——Platform 从 OpenAI 切换到 Anthropic，只改了 `use` 语句、API 密钥和模型名称。`Chat`、`Agent`、`MessageBag` 等代码完全不变。这就是 Symfony AI 抽象层的价值。

---

## 替代实现方案

### Gemini——Google AI 平台

```bash
composer require symfony/ai-gemini-platform
```

```php
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;

$platform = PlatformFactory::create($_ENV['GEMINI_API_KEY']);
$agent = new Agent($platform, 'gemini-2.5-flash');

// 后续使用方式与 OpenAI / Anthropic 完全一致
$chat = new Chat($agent, new InMemoryStore());
```

### Ollama——本地私有化部署

如果你对数据隐私有严格要求，或者想避免 API 费用，Ollama 让你在本地运行开源模型：

```bash
composer require symfony/ai-ollama-platform
```

```php
use Symfony\AI\Platform\Bridge\Ollama\PlatformFactory;

// Ollama 的工厂参数不同：第一个参数是服务端点
$platform = PlatformFactory::create('http://localhost:11434');
$agent = new Agent($platform, 'llama3.2');

$chat = new Chat($agent, new InMemoryStore());
```

> **💡 提示：** Ollama 需要先在本地安装并运行（`ollama serve`），然后通过 `ollama pull llama3.2` 下载模型。中文场景推荐使用 `qwen2.5` 系列模型。

### 存储后端选型指南

| 存储后端 | 适用场景 | 持久化 | 跨请求 | 性能 | 可查询 | 生产推荐 |
|---------|---------|--------|--------|------|--------|---------|
| **InMemory** | 开发、测试、CLI 脚本 | ❌ | ❌ | ⚡ 极快 | ❌ | 仅开发 |
| **Session** | Symfony 轻量应用 | ⚠️ 随 Session | ✅ | ⚡ 快 | ❌ | 原型/内部工具 |
| **Cache (PSR-6)** | 短期对话、有现成缓存 | ⚠️ 有 TTL | ✅ | ⚡ 快 | ❌ | 中小规模 |
| **Redis** | 高并发、实时应用 | ✅ | ✅ | ⚡ 极快 | ⚠️ 有限 | ✅ 推荐 |
| **MongoDB** | 灵活查询、大规模对话 | ✅ | ✅ | 🔶 快 | ✅ 强 | ✅ 推荐 |
| **Doctrine DBAL** | 关系型数据库为主的应用 | ✅ | ✅ | 🔶 中等 | ✅ 强 | ✅ 推荐 |
| **Cloudflare KV** | 边缘计算、全球分布 | ✅ | ✅ | ⚡ 快 | ⚠️ 有限 | 特定场景 |
| **SurrealDB** | 多模型数据库、图关系 | ✅ | ✅ | 🔶 中等 | ✅ 强 | 特定场景 |
| **Meilisearch** | 需要全文搜索对话内容 | ✅ | ✅ | 🔶 快 | ✅ 全文 | 特定场景 |

> **🏭 生产建议：** 根据你的实际需求选择存储后端：
> - **追求低延迟**（如实时聊天）→ Redis
> - **需要灵活查询和分析**（如对话审计）→ MongoDB 或 Doctrine DBAL
> - **已有关系型数据库**，不想引入新组件 → Doctrine DBAL
> - **需要水平扩展**（百万级对话）→ MongoDB（支持分片）
> - **临时/短期对话**（如表单助手）→ Cache (PSR-6)
> - **全球边缘部署** → Cloudflare KV
> - **需要全文搜索对话历史** → Meilisearch

> **🔒 安全建议：** 无论选择哪种存储后端，都要注意以下几点：
> - **对话 ID 管理**：使用不可预测的 UUID 或加密 token，避免用自增 ID 导致 IDOR 漏洞
> - **数据保留策略**：设置合理的过期时间，定期清理过期对话数据，遵守数据保护法规（如 GDPR）
> - **访问控制**：确保用户只能访问自己的对话，在服务端验证对话 ID 的归属关系
> - **敏感信息**：对话中可能包含用户敏感信息，确保存储层有适当的加密和访问限制

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Chat` | 对话管理器，自动管理消息历史和持久化。构造函数要求 Store 满足 `MessageStoreInterface & ManagedStoreInterface` 交叉类型 |
| `initiate(MessageBag)` | 初始化新对话——先调用 `drop()` 清空存储，再 `save()` 保存初始消息（通常是系统提示） |
| `submit(UserMessage)` | 提交用户消息——自动 `load()` 历史、`add()` 用户消息、`Agent::call()` 获取回复、`add()` 助手消息、`save()` 全部历史，返回 `AssistantMessage` |
| `Agent` | AI 调用引擎，封装 `Platform::invoke()`。支持 `InputProcessor`/`OutputProcessor` 管线和命名 |
| `MessageStoreInterface` | 存储核心接口——`save(MessageBag)` 保存、`load(): MessageBag` 加载 |
| `ManagedStoreInterface` | 存储管理接口——`setup()` 初始化（建表等）、`drop()` 清空所有数据 |
| `MessageNormalizer` | Symfony Serializer normalizer——各 Store 内部使用，自动将消息对象序列化/反序列化为 JSON，保留 UUID、元数据和时间戳 |
| `InMemoryStore` | 进程内存存储，实现 `ResetInterface`。适合开发/测试 |
| 对话隔离 | 每个 `Chat` 绑定一个 `Store` 实例。通过为每个对话创建使用不同标识符的 Store 来实现多对话隔离 |
| 存储后端 | InMemory / Redis / Doctrine DBAL / MongoDB / Session / Cache / Cloudflare / SurrealDB / Meilisearch 等 |
| Token 追踪 | `$response->getMetadata()->get('token_usage')` 获取本轮 Token 消耗——Metadata 从 `TextResult` 自动继承到 `AssistantMessage` |

## 下一步

现在你的聊天机器人可以记住对话了。但如果用户问"今天天气怎么样"或"帮我查一下某个产品的库存"，AI 无法回答因为它没有工具。请看 [03-tool-augmented-assistant.md](./03-tool-augmented-assistant.md) 学习如何给 AI 添加工具能力。
