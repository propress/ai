# 多轮对话与会话持久化

## 业务场景

你正在开发一个在线客服系统。用户通过网页与 AI 客服对话，关闭页面后再打开，仍然能看到之前的对话并继续聊天。不同用户之间的对话互相隔离。

**典型应用：** 在线客服系统、咨询助手、教育辅导对话、医疗问诊预分诊

---

## 涉及模块

| 模块 | Composer 包 | 用途 |
|------|------------|------|
| **Platform** | `symfony/ai-platform` + Bridge 包 | 连接 AI 平台，发送消息获取回复 |
| **Agent** | `symfony/ai-agent` | 封装调用逻辑，运行 InputProcessor/OutputProcessor 管线，作为 Chat 的后端引擎 |
| **Chat** | `symfony/ai-chat` + 存储 Bridge | 管理对话状态，自动持久化消息历史 |

> **💡 提示：** Chat 模块与平台无关，切换 AI 平台只需更换 Platform Bridge 和模型名称。本指南主示例使用 Anthropic（Claude），Doctrine DBAL 示例使用 Gemini 来展示跨平台切换。

---

## 架构概述

Chat 采用**分层依赖模型**，每一层只依赖下一层的抽象接口：

```
Chat（对话管理器 — 负责持久化与生命周期）
 ├── AgentInterface（AI 调用引擎 — 负责处理管线与平台调用）
 │    └── PlatformInterface（AI 平台连接 — 负责 HTTP 通信与序列化）
 └── MessageStoreInterface & ManagedStoreInterface（消息持久化 — 负责存取消息）
      └── 具体实现：InMemory / Redis / DBAL / MongoDB / Session / Cache / Cloudflare / SurrealDb / Pogocache / Meilisearch
```

### 三个关键设计决策

1. **Chat 不直接调用 Platform**——它通过 `AgentInterface` 间接调用。这意味着 Agent 的完整处理管线（InputProcessors / OutputProcessors）在 Chat 内部仍然生效。例如，你可以给 Agent 注册 `SystemPromptInputProcessor`，Chat 里的每次 `submit()` 都会经过它。

2. **构造器使用交叉类型（Intersection Type）**——Chat 的 `$store` 参数类型是 `MessageStoreInterface&ManagedStoreInterface`，要求 Store 同时实现两个接口。这是 PHP 8.1 的交叉类型特性，确保 Chat 既能读写消息（`save`/`load`），又能管理存储生命周期（`setup`/`drop`）。

3. **Store 负责隔离**——Chat 没有 `conversationId` 概念。对话隔离完全由 Store 实例承担——每个对话对应一个独立的 Store 实例，使用不同的标识符。

> **📝 知识扩展：** Chat 构造器的交叉类型 `MessageStoreInterface&ManagedStoreInterface` 是一个值得注意的设计。PHP 8.1 之前，你需要创建一个合并接口或使用 `@param` 注解来表达"同时实现两个接口"。交叉类型让这一约束在类型系统层面就得到保证——如果你的自定义 Store 只实现了其中一个接口，IDE 和 PHP 都会在编译期报错，而不是等到运行时才发现 `drop()` 方法不存在。所有 10 个内置 Bridge 都同时实现了这两个接口。

---

## 项目流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        多轮对话完整流程                               │
└─────────────────────────────────────────────────────────────────────┘

  用户发消息                                              返回 AI 回复
      │                                                       ▲
      ▼                                                       │
┌───────────┐    ┌──────────────────────────────────┐    ┌──────────┐
│           │    │           Chat 对话管理器          │    │          │
│   用户    │───▶│                                    │───▶│  用户    │
│   界面    │    │  1. 从 Store 加载历史消息           │    │  界面    │
│           │    │  2. 附加新的用户消息（可变操作）    │    │          │
└───────────┘    │  3. 调用 Agent（含完整管线）        │    └──────────┘
                 │  4. 合并元数据到 AssistantMessage   │
                 │  5. 保存所有消息到 Store            │
                 └──────────┬───────────────┬─────────┘
                            │               │
                    ┌───────▼───────┐ ┌─────▼──────────────────────┐
                    │     Agent     │ │      MessageStore          │
                    │  ┌──────────┐│ │   消息持久化                 │
                    │  │ Input    ││ │                              │
                    │  │Processors││ │ InMemory / Redis / DBAL /   │
                    │  ├──────────┤│ │ MongoDB / Session / Cache / │
                    │  │ Platform ││ │ Cloudflare / SurrealDb /    │
                    │  ├──────────┤│ │ Pogocache / Meilisearch     │
                    │  │ Output   ││ └──────────────────────────────┘
                    │  │Processors││
                    │  └──────────┘│
                    └───────┬──────┘
                            │
                    ┌───────▼───────┐
                    │   AI 平台 API  │
                    │  Claude /     │
                    │  Gemini /     │
                    │  GPT / ...    │
                    └───────────────┘
```

---

## 前置准备

```bash
# 核心依赖（本指南主示例使用 Anthropic）
composer require symfony/ai-platform symfony/ai-anthropic-platform
composer require symfony/ai-agent
composer require symfony/ai-chat                         # 含 InMemory 存储

# 根据你选择的存储后端安装对应的 Bridge（任选其一）
# composer require symfony/ai-cache-message-store        # PSR-6 缓存
# composer require symfony/ai-redis-message-store        # Redis
# composer require symfony/ai-doctrine-message-store     # 关系型数据库（MySQL/PostgreSQL）
# composer require symfony/ai-mongo-db-message-store     # MongoDB
# composer require symfony/ai-session-message-store      # Symfony Session
# composer require symfony/ai-cloudflare-message-store   # Cloudflare KV
# composer require symfony/ai-surrealdb-message-store    # SurrealDB
# composer require symfony/ai-pogocache-message-store    # Pogocache
# composer require symfony/ai-meilisearch-message-store  # Meilisearch
```

> **💡 提示：** `symfony/ai-chat` 包已内置 `InMemory\Store`，无需安装额外 Bridge 即可开始开发。其他存储后端需安装对应的 Bridge 包。

设置环境变量：

```bash
export ANTHROPIC_API_KEY="your-anthropic-api-key"
```

> **🔒 安全建议：** 不要将 API 密钥硬编码在代码中。在生产环境中使用 Symfony Secrets 或环境变量管理服务（如 Vault、AWS Secrets Manager）来管理敏感配置。

---

## Step 1：深入理解 Chat 的生命周期

在 [01-basic-chatbot.md](./01-basic-chatbot.md) 中，我们手动维护对话历史数组。`Chat` 模块自动化了这个过程。

### 对话生命周期

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

### `initiate()` 源码分析

```php
public function initiate(MessageBag $messages): void
{
    $this->store->drop();       // 先清空旧数据
    $this->store->save($messages); // 再保存初始消息
}
```

这两行代码意味着：`initiate()` 是**破坏性操作**——它会删除该 Store 中所有现有消息。如果你想保留旧对话，应该为新对话创建新的 Store 实例，而不是在同一个 Store 上再次调用 `initiate()`。

### `submit()` 源码分析

```php
public function submit(UserMessage $message): AssistantMessage
{
    $messages = $this->store->load();           // 1. 从 Store 加载全部历史
    $messages->add($message);                   // 2. 附加用户消息（可变操作）
    $result = $this->agent->call($messages);    // 3. 通过 Agent 调用 AI

    \assert($result instanceof TextResult);

    $assistantMessage = Message::ofAssistant($result->getContent());
    $assistantMessage->getMetadata()->merge($result->getMetadata()); // 4. 合并元数据
    $messages->add($assistantMessage);          // 5. 附加 AI 回复
    $this->store->save($messages);              // 6. 保存完整历史

    return $assistantMessage;
}
```

> **📝 知识扩展：** 注意 `submit()` 使用 `$messages->add()` 而不是 `$messages->with()`。`add()` 是**可变操作**，直接修改当前 MessageBag 实例；`with()` 是**不可变操作**，返回新的 MessageBag 副本。这里使用可变操作是刻意的设计——因为整个 MessageBag 最终会通过 `$this->store->save($messages)` 写回存储，没有必要创建中间副本。这比不可变方式更节省内存，在长对话（数十轮）场景下效果尤其明显。

### 元数据传递

`submit()` 中的 `$assistantMessage->getMetadata()->merge($result->getMetadata())` 这行代码将 Agent 返回结果中的元数据（包括 Token 用量、模型信息等）复制到 `AssistantMessage` 上。这意味着你可以从返回的 `AssistantMessage` 上直接获取调用元数据：

```php
$response = $chat->submit(Message::ofUser('你好'));

// 从 AssistantMessage 上获取 Token 用量
$tokenUsage = $response->getMetadata()->get('token_usage');
```

### 核心接口

Chat 的持久化依赖两个接口：

| 接口 | 方法 | 说明 |
|------|------|------|
| `MessageStoreInterface` | `save(MessageBag)` | 将完整消息列表保存到存储 |
| | `load(): MessageBag` | 从存储加载完整消息列表 |
| `ManagedStoreInterface` | `setup(array $options)` | 初始化存储（建表、建索引等） |
| | `drop()` | 清空存储中的所有消息 |

### 消息归一化

所有 Store 实现内部使用 `MessageNormalizer` 将消息对象序列化为 JSON 格式。每条消息存储的结构包含：类型（System/User/Assistant）、内容、元数据、时间戳等。这意味着你不需要关心序列化细节，Store 会自动处理。

### 对话隔离机制

> **⚠️ 注意：** `Chat` 类的 `initiate()` 和 `submit()` 方法**没有** `conversationId` 参数。每个 `Chat` 实例绑定一个 `MessageStore` 实例，一个 Store 实例管理一个对话。

要支持多个对话（如不同用户），需要为每个对话创建独立的 Store 实例：

```php
// 用户 A 的对话 —— 使用独立的 Store 实例
$storeA = new RedisMessageStore($redis, 'conversation:user-a:conv-001');
$chatA  = new Chat($agent, $storeA);

// 用户 B 的对话 —— 使用不同 key 的 Store 实例
$storeB = new RedisMessageStore($redis, 'conversation:user-b:conv-002');
$chatB  = new Chat($agent, $storeB);
```

各存储后端的隔离参数：

| Store | 隔离参数 | 示例 |
|-------|---------|------|
| `InMemory\Store` | `$identifier` | `new Store('conv-123')` |
| `Redis\MessageStore` | `$indexName` | `new MessageStore($redis, 'conv-123')` |
| `MongoDb\MessageStore` | `$collectionName` | `new MessageStore($client, 'mydb', 'conv-123')` |
| `Doctrine\DoctrineDbalMessageStore` | `$tableName` | `new DoctrineDbalMessageStore('conv_123', $conn)` |
| `Cache\MessageStore` | `$cacheKey` | `new MessageStore($cache, 'conv-123')` |
| `Session\MessageStore` | `$sessionKey` | `new MessageStore($requestStack, 'conv-123')` |
| `Cloudflare\MessageStore` | `$namespace` | `new MessageStore($httpClient, 'conv-123', $accountId, $apiKey)` |
| `SurrealDb\MessageStore` | `$table` | `new MessageStore($httpClient, $url, $user, $pw, $ns, $db)` |

---

## Step 2：使用内存存储的基础对话

先用内存存储来理解 Chat 的工作方式。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\InMemory\Store as InMemoryStore;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 1. 创建 Platform 和 Agent（使用 Anthropic Claude）
$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY']);
$agent = new Agent($platform, 'claude-sonnet-4-20250514');

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

> **💡 提示：** 如果你使用 OpenAI 而非 Anthropic，只需替换两行代码：
> ```php
> use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
> $platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);
> $agent = new Agent($platform, 'gpt-4.1-mini');
> ```
> Gemini 同理：
> ```php
> use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
> $platform = PlatformFactory::create($_ENV['GEMINI_API_KEY']);
> $agent = new Agent($platform, 'gemini-2.5-flash');
> ```

> **⚠️ 注意：** `InMemoryStore` 的数据仅存在于当前进程的内存中。进程结束后数据丢失，不适合跨请求的 Web 场景。适合脚本、测试和开发环境。`InMemory\Store` 还实现了 Symfony 的 `ResetInterface`——在长驻进程（如 Messenger Worker）中，框架会在每次消息处理后调用 `reset()` 自动清空内存存储，防止内存泄漏。

> **📝 知识扩展：** `Chat` 通过 `AgentInterface` 调用 AI，而不是直接调用 `PlatformInterface`。这意味着 Agent 的完整处理管线在 Chat 内部依然生效。如果你给 Agent 注册了 InputProcessor（如 `SystemPromptInputProcessor`）或 OutputProcessor，它们会在每次 `submit()` 时执行。例如，你可以通过 `SystemPromptInputProcessor` 动态替换系统提示词，而不用修改 Chat 的 `initiate()` 调用。Agent 构造器接受 `$inputProcessors` 和 `$outputProcessors` 两个可迭代参数——Chat 对此完全透明，它只调用 `$this->agent->call($messages)` 并期望得到 `TextResult`。

---

## Step 3：使用 Redis 实现跨请求持久化

在 Web 应用中，每个 HTTP 请求是独立的进程。我们需要将对话存入 Redis，这样跨请求也能保持对话上下文。

```bash
composer require symfony/ai-redis-message-store
```

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\Bridge\Redis\MessageStore as RedisMessageStore;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY']);
$agent = new Agent($platform, 'claude-sonnet-4-20250514');

// 连接 Redis
$redis = new \Redis();
$redis->connect('127.0.0.1', 6379);
```

### 模拟 Web 场景：跨请求的对话

```php
// ============================================
// 模拟第一个 HTTP 请求：用户开始新对话
// ============================================
// conversationId 由前端生成并存在 cookie/session 中
$conversationId = 'user-123-conv-456';

// 为该对话创建专属的 Store 实例
$store = new RedisMessageStore($redis, $conversationId);
$chat = new Chat($agent, $store);

// 初始化对话：设置系统提示
$chat->initiate(new MessageBag(
    Message::forSystem('你是一个旅行顾问。帮用户规划旅行。'),
));

$response = $chat->submit(Message::ofUser('我想去日本玩 5 天，预算 1 万元'));
echo "顾问：" . $response->getContent() . "\n";
// 请求结束，对话已保存到 Redis

// ============================================
// 模拟第二个 HTTP 请求：用户回来继续对话
// ============================================
// 从 cookie/session 中取回 conversationId
$conversationId = 'user-123-conv-456';

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

> **🔒 安全建议：** `conversationId` 不要使用可预测的值（如自增 ID）。建议使用 UUID 或加密 token，防止用户篡改 ID 访问他人对话。

> **🏭 生产建议：** Redis 适合对话量大、响应速度要求高的场景。建议设置合理的 TTL（过期时间）来自动清理过期对话数据，避免 Redis 内存无限增长。

---

## Step 4：使用 Doctrine DBAL 存入数据库

如果你的应用使用关系型数据库（MySQL/PostgreSQL），可以将对话存入数据库表。这里使用 **Gemini** 来展示平台切换的便捷性——Chat 和 Store 的代码完全不变，只需更换 Platform 和模型。

```bash
composer require symfony/ai-doctrine-message-store
composer require symfony/ai-gemini-platform    # 使用 Gemini 替代 Anthropic
```

```php
<?php

require 'vendor/autoload.php';

use Doctrine\DBAL\DriverManager;
use Symfony\AI\Agent\Agent;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\Bridge\Doctrine\DoctrineDbalMessageStore;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 使用 Gemini —— 只需更换 PlatformFactory 和模型名称
$platform = PlatformFactory::create($_ENV['GEMINI_API_KEY']);
$agent = new Agent($platform, 'gemini-2.5-flash');

// 创建数据库连接
$connection = DriverManager::getConnection([
    'url' => 'mysql://user:password@127.0.0.1:3306/myapp',
]);

$conversationId = 'user-123-conv-456';

// 创建 Chat Store（使用对话 ID 作为表名）
$store = new DoctrineDbalMessageStore($conversationId, $connection);

// 初始化存储（首次使用时建表）
$store->setup();

// 后续使用方式与 Redis 完全相同——Chat 不关心平台是什么
$chat = new Chat($agent, $store);

$chat->initiate(new MessageBag(
    Message::forSystem('你是一个旅行顾问。帮用户规划旅行。'),
));

$response = $chat->submit(Message::ofUser('我想去日本玩 5 天，预算 1 万元'));
echo "顾问：" . $response->getContent() . "\n";
```

> **💡 提示：** 对话数据存入数据库后，可以配合其他业务数据做分析查询，比如"过去一个月用户最常问的问题是什么"。`DoctrineDbalMessageStore` 构造器还接受可选的 `SerializerInterface` 和 `ClockInterface` 参数，用于自定义序列化和时间戳生成。

> **⚠️ 注意：** Doctrine DBAL Store 使用表名来隔离对话，因此每个对话会对应一张表。对于海量对话场景，请考虑使用 MongoDB 或 Redis 等更适合的方案。

---

## Step 5：使用 MongoDB 存储

MongoDB 是文档型数据库，天然适合存储对话这种半结构化数据。每条消息作为一个文档存入集合中。

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
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY']);
$agent = new Agent($platform, 'claude-sonnet-4-20250514');

// 连接 MongoDB
$mongoClient = new Client('mongodb://127.0.0.1:27017');

$conversationId = 'user-123-conv-456';

// 使用 conversationId 作为集合名来隔离对话
$store = new MongoMessageStore($mongoClient, 'myapp_chat', $conversationId);

// 初始化存储（创建集合）
$store->setup();

$chat = new Chat($agent, $store);

// 初始化并开始对话
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

> **🏭 生产建议：** MongoDB 适合消息结构可能变化、需要灵活查询的场景。它的文档模型天然适配对话数据，且支持水平扩展（分片），适合大规模应用。

---

## Step 6：使用 Symfony Session 存储

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
    $store = new SessionMessageStore($requestStack);
    $chat = new Chat($agent, $store);

    // 不同用户的 Session 天然隔离，无需手动管理对话 ID
    $response = $chat->submit(Message::ofUser('你好'));

    return new JsonResponse(['reply' => $response->getContent()]);
}
```

> **⚠️ 注意：** Session 存储受限于 Session 的大小和生命周期。对话过长时可能导致 Session 数据膨胀。适合原型开发和轻量级应用，不建议用于高并发生产环境。

---

## 完整示例：模拟一个在线教育辅导场景

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\InMemory\Store as InMemoryStore;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY']);
$agent = new Agent($platform, 'claude-sonnet-4-20250514');
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

foreach ($dialogue as $question) {
    echo "学生：{$question}\n\n";

    $response = $chat->submit(Message::ofUser($question));
    echo "老师：" . $response->getContent() . "\n\n";
    echo str_repeat('-', 50) . "\n\n";
}
```

---

## 其他实现方案：存储后端对比

除了上面详细讲解的存储方案，Chat 模块还提供了多个额外的 Bridge。以下是各存储后端的对比和代码示例。

### PSR-6 Cache 存储

```bash
composer require symfony/ai-cache-message-store
```

```php
use Symfony\AI\Chat\Bridge\Cache\MessageStore as CacheMessageStore;
use Symfony\Component\Cache\Adapter\FilesystemAdapter;

$cache = new FilesystemAdapter();
$store = new CacheMessageStore($cache, 'conv-123', 86400); // TTL: 24 小时
$chat = new Chat($agent, $store);
```

### Cloudflare KV 存储

适合已使用 Cloudflare 生态的应用，对话数据存入 Cloudflare Workers KV：

```bash
composer require symfony/ai-cloudflare-message-store
```

```php
use Symfony\AI\Chat\Bridge\Cloudflare\MessageStore as CloudflareMessageStore;
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create();
$store = new CloudflareMessageStore(
    $httpClient,
    'conv-123',                          // namespace（用于隔离对话）
    $_ENV['CLOUDFLARE_ACCOUNT_ID'],      // Cloudflare 账户 ID
    $_ENV['CLOUDFLARE_API_KEY'],         // API 密钥
);
$chat = new Chat($agent, $store);
```

### SurrealDB 存储

SurrealDB 是一个多模型数据库，支持关系型、文档型和图数据库特性：

```bash
composer require symfony/ai-surrealdb-message-store
```

```php
use Symfony\AI\Chat\Bridge\SurrealDb\MessageStore as SurrealDbMessageStore;
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create();
$store = new SurrealDbMessageStore(
    $httpClient,
    'http://127.0.0.1:8000',   // SurrealDB 端点
    'root',                     // 用户名
    'root',                     // 密码
    'myapp',                    // 命名空间
    'chat',                     // 数据库名
);
$chat = new Chat($agent, $store);
```

### 存储后端选型指南

| 存储后端 | 适用场景 | 持久化 | 跨请求 | 性能 | 可查询 | 生产推荐 |
|---------|---------|--------|--------|------|--------|---------|
| **InMemory** | 开发、测试、CLI 脚本 | ❌ | ❌ | ⚡ 极快 | ❌ | 仅开发 |
| **Session** | Symfony 轻量应用 | ⚠️ 随 Session | ✅ | ⚡ 快 | ❌ | 原型/内部工具 |
| **Cache (PSR-6)** | 短期对话、有现成缓存 | ⚠️ 有 TTL | ✅ | ⚡ 快 | ❌ | 中小规模 |
| **Redis** | 高并发、实时应用 | ✅ | ✅ | ⚡ 极快 | ⚠️ 有限 | ✅ 推荐 |
| **MongoDB** | 灵活查询、大规模对话 | ✅ | ✅ | 🔶 快 | ✅ 强 | ✅ 推荐 |
| **Doctrine DBAL** | 关系型数据库为主的应用 | ✅ | ✅ | 🔶 中等 | ✅ 强 | ✅ 推荐 |
| **Cloudflare KV** | Cloudflare 生态、边缘部署 | ✅ | ✅ | ⚡ 快 | ⚠️ 有限 | ✅ 边缘场景 |
| **SurrealDB** | 多模型查询、图关系分析 | ✅ | ✅ | 🔶 快 | ✅ 强 | ✅ 推荐 |
| **Pogocache** | Pogocache 托管缓存 | ✅ | ✅ | ⚡ 快 | ❌ | 中小规模 |
| **Meilisearch** | 全文搜索对话内容 | ✅ | ✅ | 🔶 快 | ✅ 搜索优化 | 特定场景 |

> **🏭 生产建议：** 根据你的实际需求选择存储后端：
> - **追求低延迟**（如实时聊天）→ Redis
> - **需要灵活查询和分析**（如对话审计）→ MongoDB 或 Doctrine DBAL
> - **已有关系型数据库**，不想引入新组件 → Doctrine DBAL
> - **需要水平扩展**（百万级对话）→ MongoDB（支持分片）
> - **边缘部署**（全球低延迟）→ Cloudflare KV
> - **需要全文搜索对话**（如知识回溯）→ Meilisearch
> - **临时/短期对话**（如表单助手）→ Cache (PSR-6)

> **🔒 安全建议：** 无论选择哪种存储后端，都要注意以下几点：
> - **对话 ID 管理**：使用不可预测的 UUID 或加密 token，避免用自增 ID 导致 IDOR 漏洞
> - **数据保留策略**：设置合理的过期时间，定期清理过期对话数据，遵守数据保护法规（如 GDPR）
> - **访问控制**：确保用户只能访问自己的对话，在服务端验证对话 ID 的归属关系
> - **敏感信息**：对话中可能包含用户敏感信息，确保存储层有适当的加密和访问限制
> - **API 密钥保护**：Cloudflare、SurrealDb、Pogocache 等远程 Bridge 的构造器使用 `#[\SensitiveParameter]` 标记密码参数，确保它们不会出现在堆栈跟踪中

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Chat` 构造器交叉类型 | `$store` 参数类型为 `MessageStoreInterface&ManagedStoreInterface`，利用 PHP 8.1 交叉类型确保 Store 同时支持数据读写和生命周期管理 |
| `initiate(MessageBag)` | 初始化新对话——先调用 `$store->drop()` 清空旧数据，再调用 `$store->save()` 保存初始消息。是破坏性操作 |
| `submit(UserMessage)` | 核心方法：加载历史 → 附加用户消息 → 调用 Agent → 合并元数据 → 附加 AI 回复 → 保存 → 返回 `AssistantMessage` |
| AssistantMessage 元数据合并 | `submit()` 内部执行 `$assistantMessage->getMetadata()->merge($result->getMetadata())`，将 Token 用量等元数据从 Agent 结果复制到返回的消息上 |
| 可变 vs 不可变操作 | `submit()` 内部使用 `$messages->add()`（可变）而非 `$messages->with()`（不可变），因为整个 MessageBag 最终写回存储，无需创建中间副本 |
| Agent 管线透传 | Chat 通过 `AgentInterface` 调用 AI，Agent 的 InputProcessors/OutputProcessors 在每次 `submit()` 时完整执行 |
| `MessageStoreInterface` | 数据接口，定义 `save(MessageBag)` 和 `load(): MessageBag` 方法 |
| `ManagedStoreInterface` | 管理接口，定义 `setup(array $options)` 和 `drop()` 方法 |
| `InMemory\Store` 与 `ResetInterface` | 内存存储实现 Symfony 的 `ResetInterface`，长驻进程中框架会自动调用 `reset()` 清空数据 |
| `MessageNormalizer` | 消息序列化器，各 Store 内部使用 Symfony Serializer 自动处理 JSON 转换 |
| 对话隔离 | 通过为每个对话创建独立的 Store 实例来实现，每个 Store 使用不同的标识符（无全局 conversationId） |
| 10 个内置 Bridge | InMemory、Cache、Redis、Doctrine DBAL、MongoDB、Session、Cloudflare、SurrealDb、Pogocache、Meilisearch——覆盖从开发到生产的各种场景 |
| 平台无关性 | Chat 和 Store 代码不依赖具体平台。切换 Anthropic → Gemini → OpenAI 只需更换 `PlatformFactory` 和模型名称 |

## 下一步

现在你的聊天机器人可以记住对话了。但如果用户问"今天天气怎么样"或"帮我查一下某个产品的库存"，AI 无法回答因为它没有工具。请看 [03-tool-augmented-assistant.md](./03-tool-augmented-assistant.md) 学习如何给 AI 添加工具能力。
