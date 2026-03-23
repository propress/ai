# Chat 组件

## 概述

Chat 组件是 Symfony AI 单体仓库中负责**对话历史持久化**的核心组件。它提供了一套完整的会话管理抽象，让应用程序能够将对话历史存储到多种后端，并通过统一接口与 AI Agent 进行多轮对话交互。

### 主要功能

- **对话历史持久化**：将多轮对话消息存储到各种后端存储中
- **统一接口**：通过 `ChatInterface` 提供一致的对话 API
- **10+ 存储后端**：覆盖缓存、数据库、键值存储等主流方案
- **消息序列化**：通过 `MessageNormalizer` 支持各种消息类型的序列化/反序列化
- **生命周期管理**：支持会话的初始化、保存、加载和清除

---

## 架构

### 目录结构

```
src/chat/src/
├── ChatInterface.php               # 核心对话接口
├── Chat.php                        # Chat 主实现类
├── MessageStoreInterface.php       # 消息存储接口
├── ManagedStoreInterface.php       # 可管理存储接口
├── MessageNormalizer.php           # 消息序列化器
│
├── InMemory/
│   └── Store.php                   # 内存消息存储
│
├── Bridge/                         # 存储后端桥接器
│   ├── Cache/
│   │   └── MessageStore.php        # Symfony Cache 存储
│   ├── Cloudflare/
│   │   └── MessageStore.php        # Cloudflare KV 存储
│   ├── Doctrine/
│   │   └── DoctrineDbalMessageStore.php  # Doctrine DBAL 存储
│   ├── Meilisearch/
│   │   └── MessageStore.php        # Meilisearch 存储
│   ├── MongoDb/
│   │   └── MessageStore.php        # MongoDB 存储
│   ├── Pogocache/
│   │   └── MessageStore.php        # Pogocache 存储
│   ├── Redis/
│   │   └── MessageStore.php        # Redis 存储
│   ├── Session/
│   │   └── MessageStore.php        # Symfony Session 存储
│   └── SurrealDb/
│       └── MessageStore.php        # SurrealDB 存储
│
├── Command/
│   ├── SetupStoreCommand.php       # 初始化消息存储命令
│   └── DropStoreCommand.php        # 清空消息存储命令
│
└── Exception/
    ├── ExceptionInterface.php
    ├── InvalidArgumentException.php
    ├── LogicException.php
    └── RuntimeException.php
```

---

## 核心概念

### ChatInterface

`ChatInterface` 定义了对话的核心操作：

```php
interface ChatInterface
{
    // 初始化对话（清除历史并设置初始消息，如系统提示词）
    public function initiate(MessageBag $messages): void;

    // 提交用户消息，返回 AI 助手的响应
    public function submit(UserMessage $message): AssistantMessage;
}
```

### Chat 实现类

`Chat` 是 `ChatInterface` 的默认实现，整合了 Agent 和消息存储：

```php
use Symfony\AI\Chat\Chat;

$chat = new Chat(
    agent: $agent,           // AgentInterface 实例
    store: $messageStore,    // MessageStoreInterface & ManagedStoreInterface
);

// 初始化对话（设置系统提示词）
$chat->initiate(new MessageBag(
    Message::forSystem('你是一个友好的助手，请用中文回答问题。')
));

// 提交用户消息并获取响应
$response = $chat->submit(Message::ofUser('你好，请介绍一下 Symfony AI。'));
echo $response->getContent();

// 继续多轮对话（历史消息自动持久化）
$response2 = $chat->submit(Message::ofUser('它支持哪些向量数据库？'));
echo $response2->getContent();
```

**Chat 类内部流程**：

1. `initiate()` 调用 `store->drop()` 清除历史，然后调用 `store->save()` 保存初始消息
2. `submit()` 调用 `store->load()` 加载历史消息，添加新消息，调用 `agent->call()` 获取响应，将助手消息加入历史，调用 `store->save()` 保存

### MessageStoreInterface

`MessageStoreInterface` 定义了消息存储的基本操作：

```php
interface MessageStoreInterface
{
    // 保存完整的消息包
    public function save(MessageBag $messages): void;

    // 加载消息包（如果没有历史则返回空的 MessageBag）
    public function load(): MessageBag;
}
```

### ManagedStoreInterface

`ManagedStoreInterface` 扩展存储接口，支持基础设施管理：

```php
interface ManagedStoreInterface
{
    // 初始化存储基础设施（创建表、索引等）
    public function setup(array $options = []): void;

    // 清空存储中的消息数据
    public function drop(): void;
}
```

> **重要**：`Chat` 类要求传入的存储必须同时实现 `MessageStoreInterface` 和 `ManagedStoreInterface`，通过交叉类型 `MessageStoreInterface&ManagedStoreInterface` 强制约束。

---

## 存储后端

### InMemory Store（内存存储）

基于 PHP 内存的轻量级消息存储，实现了 `ResetInterface` 支持 Symfony 服务重置：

```php
use Symfony\AI\Chat\InMemory\Store;

$store = new Store(
    identifier: '_message_store_memory', // 可选，支持多实例隔离
);

// 基本操作
$store->setup();                    // 初始化空 MessageBag
$store->save($messageBag);          // 保存消息
$messageBag = $store->load();       // 加载消息
$store->drop();                     // 清空消息（重置为空 MessageBag）
$store->reset();                    // 完全重置（清除所有数据）
```

**适用场景**：
- 单元测试
- 无状态 API 请求中的临时对话
- 开发调试

### Cache Store（Symfony Cache 存储）

使用 PSR-6 `CacheItemPoolInterface` 的消息存储，支持任意 Symfony Cache 适配器：

```php
use Symfony\AI\Chat\Bridge\Cache\MessageStore;

$store = new MessageStore(
    cache: $cachePool,           // PSR-6 CacheItemPoolInterface
    cacheKey: '_message_store_cache',
    ttl: 86400,                  // 缓存 TTL（秒），默认 24 小时
);
```

**适用场景**：
- 使用文件、APCu、Memcached 等缓存后端
- 需要 TTL 自动过期的会话
- 轻量级部署，无需专门的数据库

**安装**：

```bash
composer require symfony/cache
```

**配置示例**：

```php
use Symfony\Component\Cache\Adapter\FilesystemAdapter;

$cachePool = new FilesystemAdapter('chat', 86400, '/tmp/cache');
$store = new MessageStore($cachePool);
```

### Session Store（Symfony Session 存储）

将消息存储在 Symfony HTTP Session 中，自动跟随用户会话：

```php
use Symfony\AI\Chat\Bridge\Session\MessageStore;

$store = new MessageStore(
    requestStack: $requestStack,
    sessionKey: 'messages',       // Session 中的键名
);
```

**适用场景**：
- Web 应用中的用户会话跟踪
- 无需外部存储的简单部署
- 每个用户的对话隔离

**注意事项**：
- `setup()` 会重置 Session 中的消息
- `drop()` 会从 Session 中移除消息键

### Redis Store（Redis 存储）

使用 Redis 的高性能消息存储，支持持久化和分布式部署：

```php
use Symfony\AI\Chat\Bridge\Redis\MessageStore;
use Symfony\AI\Chat\MessageNormalizer;
use Symfony\Component\Serializer\Serializer;
use Symfony\Component\Serializer\Encoder\JsonEncoder;
use Symfony\Component\Serializer\Normalizer\ArrayDenormalizer;

$store = new MessageStore(
    redis: $redis,                // \Redis 实例
    indexName: 'chat_messages',   // Redis 键名
    serializer: new Serializer(   // 可选，自定义序列化器
        [new ArrayDenormalizer(), new MessageNormalizer()],
        [new JsonEncoder()],
    ),
);
```

**适用场景**：
- 高并发 Web 应用
- 微服务架构中共享对话状态
- 需要高性能读写

**安装**：

```bash
composer require predis/predis
# 或使用 PHP Redis 扩展
```

### Doctrine DBAL Store（数据库存储）

使用 Doctrine DBAL 将消息持久化到关系型数据库，支持 MySQL、PostgreSQL、SQLite 等：

```php
use Symfony\AI\Chat\Bridge\Doctrine\DoctrineDbalMessageStore;

$store = new DoctrineDbalMessageStore(
    tableName: 'chat_messages',
    dbalConnection: $connection,   // Doctrine DBAL Connection
    clock: $clock,                 // PSR-20 ClockInterface（可选）
);

// 创建表结构（自动创建，包含 id、messages、added_at 列）
$store->setup();
```

**数据库表结构**：

| 列名 | 类型 | 说明 |
|------|------|------|
| `id` | BIGINT (AUTO_INCREMENT) | 主键 |
| `messages` | TEXT | JSON 序列化的消息数组 |
| `added_at` | INTEGER | 时间戳 |

**特殊说明**：
- `setup()` 会自动检查表是否存在，不存在则创建
- `drop()` 只删除记录，不删除表结构
- `save()` 每次保存都插入新行（而非更新），`load()` 按时间顺序合并所有行的消息
- 支持 Oracle 数据库（自动处理序列和触发器）

**安装**：

```bash
composer require doctrine/dbal
```

### MongoDB Store（MongoDB 存储）

使用 MongoDB 存储会话消息：

```php
use Symfony\AI\Chat\Bridge\MongoDb\MessageStore;

$store = new MessageStore(
    client: $mongoClient,
    database: 'chat_db',
    collection: 'messages',
    indexName: 'chat_session_001',  // 文档 ID
);
```

**适用场景**：
- 已使用 MongoDB 的项目
- 需要灵活 Schema 的场景
- 大量用户会话的存储

### Meilisearch Store（Meilisearch 存储）

使用 Meilisearch 搜索引擎存储消息，支持按时间顺序检索：

```php
use Symfony\AI\Chat\Bridge\Meilisearch\MessageStore;
use Symfony\Component\Clock\NativeClock;

$store = new MessageStore(
    httpClient: $httpClient,
    endpointUrl: 'http://localhost:7700',
    apiKey: 'masterKey',
    clock: new NativeClock(),
    indexName: '_message_store_meilisearch',
);
```

**特殊说明**：
- `setup()` 创建 Meilisearch 索引并配置 `addedAt` 为排序字段
- 消息按 `addedAt` 升序加载，保证顺序正确

**安装**：

```bash
composer require symfony/clock
```

### Cloudflare KV Store（Cloudflare KV 存储）

使用 Cloudflare Workers KV 存储会话消息，适合无服务器架构：

```php
use Symfony\AI\Chat\Bridge\Cloudflare\MessageStore;

$store = new MessageStore(
    httpClient: $httpClient,
    namespace: 'chat_messages',
    accountId: 'your-account-id',
    apiKey: 'your-api-key',
    endpointUrl: 'https://api.cloudflare.com/client/v4/accounts',
);
```

**适用场景**：
- Cloudflare Workers 无服务器架构
- 全球分布式会话存储
- 边缘计算场景

**特殊说明**：
- `setup()` 检查 KV 命名空间是否存在，不存在则创建
- `drop()` 删除命名空间中的所有键值对

### Pogocache Store（Pogocache 存储）

使用 Pogocache 键值存储的轻量级消息存储：

```php
use Symfony\AI\Chat\Bridge\Pogocache\MessageStore;

$store = new MessageStore(
    httpClient: $httpClient,
    host: 'http://localhost:8080',
    password: 'your-password',
    key: '_message_store_pogocache',
);
```

### SurrealDB Store（SurrealDB 存储）

使用 SurrealDB 多模型数据库存储消息：

```php
use Symfony\AI\Chat\Bridge\SurrealDb\MessageStore;

$store = new MessageStore(
    httpClient: $httpClient,
    endpointUrl: 'http://localhost:8000',
    user: 'root',
    password: 'root',
    namespace: 'chat',
    database: 'messages',
    table: '_message_store_surrealdb',
    isNamespacedUser: false,
);
```

---

## 消息序列化

### MessageNormalizer

`MessageNormalizer` 实现了 Symfony Serializer 的 `NormalizerInterface` 和 `DenormalizerInterface`，负责将 Platform 消息对象与 PHP 数组之间相互转换，是所有需要持久化的消息存储的基础。

#### 支持的消息类型

| 消息类型 | 说明 |
|----------|------|
| `SystemMessage` | 系统提示词消息 |
| `AssistantMessage` | AI 助手消息（支持工具调用） |
| `UserMessage` | 用户消息（支持多模态内容） |
| `ToolCallMessage` | 工具调用消息 |

#### 序列化格式

序列化后的消息包含以下字段：

```json
{
    "id": "uuid-v4",
    "type": "Symfony\\AI\\Platform\\Message\\UserMessage",
    "content": "",
    "contentAsBase64": [
        {
            "type": "Symfony\\AI\\Platform\\Message\\Content\\Text",
            "content": "用户输入的文本"
        }
    ],
    "toolsCalls": [],
    "metadata": {},
    "addedAt": 1700000000
}
```

#### 支持的内容类型

`UserMessage` 的内容支持以下类型（`contentAsBase64` 数组）：

| 类型 | 存储方式 |
|------|----------|
| `Text` | 文本字符串 |
| `Image` | Base64 编码 |
| `Audio` | Base64 编码 |
| `File` | Base64 编码 |
| `Document` | Base64 编码 |
| `ImageUrl` | URL 字符串 |
| `DocumentUrl` | URL 字符串 |

#### 使用示例

```php
use Symfony\AI\Chat\MessageNormalizer;
use Symfony\Component\Serializer\Encoder\JsonEncoder;
use Symfony\Component\Serializer\Normalizer\ArrayDenormalizer;
use Symfony\Component\Serializer\Serializer;

$serializer = new Serializer(
    [new ArrayDenormalizer(), new MessageNormalizer()],
    [new JsonEncoder()],
);

// 序列化消息数组为 JSON
$json = $serializer->serialize($messageBag->getMessages(), 'json');

// 反序列化 JSON 为消息数组
use Symfony\AI\Platform\Message\MessageInterface;
$messages = $serializer->deserialize($json, MessageInterface::class.'[]', 'json');
$messageBag = new MessageBag(...$messages);
```

#### 自定义 ID 键

部分存储后端使用不同的 ID 字段名（如 MongoDB），可通过上下文指定：

```php
$normalizer->normalize($message, context: ['identifier' => '_id']);
$normalizer->denormalize($data, MessageInterface::class, context: ['identifier' => '_id']);
```

---

## CLI 命令

### ai:message-store:setup

初始化消息存储的基础设施（如创建数据库表）：

```bash
php bin/console ai:message-store:setup <store>
```

**参数**：
- `store`：要初始化的存储名称（自动补全可用存储列表）

**示例**：

```bash
# 初始化名为 "doctrine" 的消息存储
php bin/console ai:message-store:setup doctrine

# 初始化 Redis 消息存储
php bin/console ai:message-store:setup redis
```

**特性**：
- 支持 Shell 自动补全（列出所有可用存储）
- 在 `initialize()` 阶段验证存储是否存在且支持 setup

### ai:message-store:drop

清空或删除消息存储中的数据：

```bash
php bin/console ai:message-store:drop <store> [--force]
```

**参数**：
- `store`：要清空的存储名称

**选项**：
- `--force` / `-f`：必须指定此选项才会执行删除（防止误操作）

**示例**：

```bash
# 必须加 --force 才能执行
php bin/console ai:message-store:drop doctrine --force

# 不加 --force 会显示警告并返回失败
php bin/console ai:message-store:drop redis
```

---

## 异常处理

Chat 组件定义了以下异常层次结构：

```php
// 基础接口，所有 Chat 异常都实现此接口
Symfony\AI\Chat\Exception\ExceptionInterface

// 无效参数（如不支持的选项、无效配置）
Symfony\AI\Chat\Exception\InvalidArgumentException

// 运行时错误（如网络请求失败、数据库连接失败）
Symfony\AI\Chat\Exception\RuntimeException

// 逻辑错误（如 MessageNormalizer 遇到未知消息类型）
Symfony\AI\Chat\Exception\LogicException
```

**异常处理示例**：

```php
use Symfony\AI\Chat\Exception\RuntimeException;
use Symfony\AI\Agent\Exception\ExceptionInterface as AgentException;

try {
    $response = $chat->submit(Message::ofUser('问题'));
} catch (AgentException $e) {
    // Agent 调用失败（如 API 超时、速率限制等）
    $logger->error('Agent 调用失败: ' . $e->getMessage());
} catch (RuntimeException $e) {
    // 消息存储操作失败
    $logger->error('消息存储失败: ' . $e->getMessage());
}
```

---

## 与 Agent 集成模式

### 基础集成

```php
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\InMemory\Store;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 创建内存存储（开发/测试场景）
$store = new Store();
$store->setup();

// 创建 Chat 实例
$chat = new Chat(agent: $agent, store: $store);

// 初始化对话
$chat->initiate(new MessageBag(
    Message::forSystem('你是专业的 PHP 开发助手，请提供简洁、准确的技术建议。')
));

// 多轮对话
$response1 = $chat->submit(Message::ofUser('如何安装 Symfony AI？'));
echo $response1->getContent();

$response2 = $chat->submit(Message::ofUser('它支持哪些 AI 平台？'));
echo $response2->getContent();
```

### Web 应用集成（Session 存储）

```php
// src/Controller/ChatController.php
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\Bridge\Session\MessageStore;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\JsonResponse;

class ChatController extends AbstractController
{
    public function __construct(
        private readonly Chat $chat,
    ) {}

    public function chat(Request $request): JsonResponse
    {
        $userInput = $request->getPayload()->getString('message');
        $response = $this->chat->submit(Message::ofUser($userInput));

        return $this->json([
            'response' => $response->getContent(),
        ]);
    }

    public function reset(): JsonResponse
    {
        $this->chat->initiate(new MessageBag(
            Message::forSystem('你好，有什么可以帮助你的？')
        ));

        return $this->json(['status' => 'reset']);
    }
}
```

### 生产环境集成（Redis + 用户隔离）

```php
// 每个用户使用独立的 Redis 键
$userId = $security->getUser()->getId();

$store = new \Symfony\AI\Chat\Bridge\Redis\MessageStore(
    redis: $redis,
    indexName: "chat_messages_{$userId}",
);

$chat = new Chat(agent: $agent, store: $store);
```

### Symfony 服务配置

```yaml
# config/services.yaml
services:
    # 消息存储（使用 Redis）
    Symfony\AI\Chat\Bridge\Redis\MessageStore:
        arguments:
            $redis: '@Redis'
            $indexName: 'chat_messages'

    # 或使用 Symfony Cache
    Symfony\AI\Chat\Bridge\Cache\MessageStore:
        arguments:
            $cache: '@cache.app'
            $cacheKey: 'chat_messages'
            $ttl: 3600

    # Chat 服务
    Symfony\AI\Chat\Chat:
        arguments:
            $agent: '@Symfony\AI\Agent\Agent'
            $store: '@Symfony\AI\Chat\Bridge\Redis\MessageStore'
```

---

## 存储后端对比

| 存储后端 | 持久化 | 分布式 | 自动过期 | 需要安装 | 适用场景 |
|----------|:------:|:------:|:--------:|----------|----------|
| InMemory | ❌ | ❌ | ❌ | 无 | 测试/开发 |
| Cache | ✅ | 取决于驱动 | ✅ | symfony/cache | 简单部署 |
| Session | ❌ | ❌ | 随 Session | symfony/http-foundation | Web 应用 |
| Redis | ✅ | ✅ | 可配置 | predis/predis | 高并发 |
| Doctrine | ✅ | ✅ | ❌ | doctrine/dbal | 已有数据库 |
| MongoDB | ✅ | ✅ | ✅ | mongodb/mongodb | NoSQL 场景 |
| Meilisearch | ✅ | ✅ | ❌ | 需要 Meilisearch 服务 | 需要全文搜索 |
| Cloudflare | ✅ | ✅ | ✅ | 需要 CF 账户 | 无服务器/边缘 |
| Pogocache | ✅ | ✅ | ❌ | 需要 Pogocache 服务 | 专用缓存 |
| SurrealDB | ✅ | ✅ | ❌ | 需要 SurrealDB 服务 | 多模型数据库 |

---

## 代码示例

### 完整的多轮对话应用

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\Bridge\Cache\MessageStore;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\Cache\Adapter\FilesystemAdapter;

// 创建存储
$cache = new FilesystemAdapter('chat', 3600);
$store = new MessageStore($cache, '_chat_session', 3600);
$store->setup();

// 创建 Chat 实例
$chat = new Chat(agent: $agent, store: $store);

// 初始化
$chat->initiate(new MessageBag(
    Message::forSystem(
        '你是一个 Symfony 专家助手。' .
        '你的回答应当简洁、专业，并给出可运行的代码示例。'
    ),
));

// 模拟对话
$questions = [
    '什么是 Symfony AI？',
    '它与 OpenAI 如何集成？',
    '给我一个简单的代码示例',
];

foreach ($questions as $question) {
    echo "用户: {$question}\n";
    $response = $chat->submit(Message::ofUser($question));
    echo "助手: {$response->getContent()}\n\n";
}

// 检查对话历史
$history = $store->load();
echo '历史消息数: ' . count($history->getMessages()) . "\n";
```

### 带有工具调用的对话

```php
use Symfony\AI\Platform\Message\AssistantMessage;
use Symfony\AI\Platform\Result\ToolCall;

// Chat 类内部处理工具调用
// AssistantMessage 可能包含工具调用信息
$response = $chat->submit(Message::ofUser('今天的天气怎么样？'));

// 响应消息中包含元数据
$metadata = $response->getMetadata();

// 检查工具调用（如果 Agent 配置了工具）
if ($response->hasToolCalls()) {
    foreach ($response->getToolCalls() as $toolCall) {
        echo $toolCall->getName();        // 工具名称
        echo json_encode($toolCall->getArguments()); // 工具参数
    }
}
```

### 重置和切换对话主题

```php
// 开始新对话主题（清除历史）
$chat->initiate(new MessageBag(
    Message::forSystem('你现在是一个代码审查助手，专注于 PHP 代码质量。'),
));

$response = $chat->submit(Message::ofUser('请审查以下代码：' . $phpCode));
```

### 加载现有对话历史

```php
use Symfony\AI\Chat\Bridge\Doctrine\DoctrineDbalMessageStore;

$store = new DoctrineDbalMessageStore(
    tableName: 'chat_history',
    dbalConnection: $connection,
);

// 加载已有历史（无需 setup，直接使用）
$existingHistory = $store->load();
echo '已有消息数: ' . count($existingHistory->getMessages());

// 继续对话（不调用 initiate，保留历史）
$chat = new Chat(agent: $agent, store: $store);
$response = $chat->submit(Message::ofUser('继续我们之前的话题...'));
```

### 自定义序列化器

```php
use Symfony\AI\Chat\MessageNormalizer;
use Symfony\Component\Serializer\Serializer;
use Symfony\Component\Serializer\Encoder\JsonEncoder;
use Symfony\Component\Serializer\Normalizer\ArrayDenormalizer;

// 创建自定义序列化器
$serializer = new Serializer(
    normalizers: [
        new ArrayDenormalizer(),
        new MessageNormalizer(),
    ],
    encoders: [new JsonEncoder()],
);

// 直接序列化消息
$messages = $messageBag->getMessages();
$json = $serializer->serialize($messages, 'json');

// 反序列化
$restored = $serializer->deserialize(
    $json,
    \Symfony\AI\Platform\Message\MessageInterface::class . '[]',
    'json',
);
```

---

## 消息存储实现指南

如需实现自定义消息存储后端，需要实现 `MessageStoreInterface` 和 `ManagedStoreInterface`：

```php
use Symfony\AI\Chat\ManagedStoreInterface;
use Symfony\AI\Chat\MessageNormalizer;
use Symfony\AI\Chat\MessageStoreInterface;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\Message\MessageInterface;
use Symfony\Component\Serializer\Encoder\JsonEncoder;
use Symfony\Component\Serializer\Normalizer\ArrayDenormalizer;
use Symfony\Component\Serializer\Serializer;

final class CustomMessageStore implements ManagedStoreInterface, MessageStoreInterface
{
    private readonly Serializer $serializer;

    public function __construct(
        private readonly CustomStorageClient $client,
        private readonly string $key = 'messages',
    ) {
        $this->serializer = new Serializer(
            [new ArrayDenormalizer(), new MessageNormalizer()],
            [new JsonEncoder()],
        );
    }

    public function setup(array $options = []): void
    {
        // 初始化存储结构（如创建表、容器等）
        $this->client->initialize($this->key);
    }

    public function drop(): void
    {
        // 清空消息数据（通常不删除结构）
        $this->client->clear($this->key);
    }

    public function save(MessageBag $messages): void
    {
        $json = $this->serializer->serialize($messages->getMessages(), 'json');
        $this->client->set($this->key, $json);
    }

    public function load(): MessageBag
    {
        $json = $this->client->get($this->key);

        if (null === $json || '' === $json) {
            return new MessageBag();
        }

        $messages = $this->serializer->deserialize(
            $json,
            MessageInterface::class . '[]',
            'json',
        );

        return new MessageBag(...$messages);
    }
}
```

**实现要点**：
1. `save()` 应完全替换存储中的消息（不是追加）
2. `load()` 在没有历史时应返回空的 `MessageBag`，不能抛出异常
3. `drop()` 应清空消息数据但保留存储结构
4. 使用 `MessageNormalizer` 进行序列化，确保消息类型信息被正确保留
5. 对敏感参数（如密钥、密码）使用 `#[\SensitiveParameter]` 注解
