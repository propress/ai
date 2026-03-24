# 多轮对话与会话持久化

## 业务场景

你正在开发一个在线客服系统。用户通过网页与 AI 客服对话，关闭页面后再打开，仍然能看到之前的对话并继续聊天。不同用户之间的对话互相隔离。

**典型应用：** 在线客服系统、咨询助手、教育辅导对话、医疗问诊预分诊

---

## 涉及模块

| 模块 | Composer 包 | 用途 |
|------|------------|------|
| **Platform** | `symfony/ai-platform` + `symfony/ai-anthropic-platform` | 连接 AI 平台，发送消息获取回复 |
| **Agent** | `symfony/ai-agent` | 封装调用逻辑，作为 Chat 的后端引擎 |
| **Chat** | `symfony/ai-chat` + 存储 Bridge | 管理对话状态，自动持久化消息历史 |

> **💡 提示：** 本指南使用 Anthropic（Claude）作为 AI 平台。如果你更熟悉 OpenAI，可参考 [01-basic-chatbot.md](./01-basic-chatbot.md)。Chat 模块与平台无关，切换只需更换 Platform 和模型名称。

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
│           │    │  2. 附加新的用户消息               │    │          │
└───────────┘    │  3. 调用 Agent 获取回复            │    └──────────┘
                 │  4. 保存所有消息到 Store            │
                 │  5. 返回 AI 回复                   │
                 └──────────┬───────────────┬─────────┘
                            │               │
                    ┌───────▼───────┐ ┌─────▼──────────────┐
                    │     Agent     │ │   MessageStore     │
                    │   调用引擎    │ │   消息持久化        │
                    └───────┬───────┘ └─────┬──────────────┘
                            │               │
                    ┌───────▼───────┐ ┌─────▼──────────────┐
                    │   Platform    │ │ 存储后端：           │
                    │  (Anthropic)  │ │ InMemory / Redis /  │
                    │               │ │ DBAL / MongoDB /    │
                    └───────┬───────┘ │ Session / Cache     │
                            │         └────────────────────┘
                    ┌───────▼───────┐
                    │  Claude API   │
                    └───────────────┘
```

**核心类的关系：**

```
Chat（对话管理器）
 ├── Agent（AI 调用引擎）
 │    └── Platform（AI 平台连接，如 Anthropic / OpenAI）
 └── MessageStore（消息持久化）
      └── 具体实现：InMemory / Redis / DBAL / MongoDB / Session / Cache
```

---

## 前置准备

```bash
# 核心依赖
composer require symfony/ai-platform symfony/ai-anthropic-platform
composer require symfony/ai-agent
composer require symfony/ai-chat                         # 含 InMemory 存储

# 根据你选择的存储后端安装对应的 Bridge（任选其一）
# composer require symfony/ai-cache-message-store        # PSR-6 缓存
# composer require symfony/ai-redis-message-store        # Redis
# composer require symfony/ai-doctrine-message-store     # 关系型数据库（MySQL/PostgreSQL）
# composer require symfony/ai-mongo-db-message-store     # MongoDB
# composer require symfony/ai-session-message-store      # Symfony Session
```

> **💡 提示：** `symfony/ai-chat` 包已内置 `InMemory\Store`，无需安装额外 Bridge 即可开始开发。其他存储后端需安装对应的 Bridge 包。

设置环境变量：

```bash
export ANTHROPIC_API_KEY="your-anthropic-api-key"
```

> **🔒 安全建议：** 不要将 API 密钥硬编码在代码中。在生产环境中使用 Symfony Secrets 或环境变量管理服务（如 Vault、AWS Secrets Manager）来管理敏感配置。

---

## Step 1：深入理解 Chat 架构

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
> // ...
> $platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);
> $agent = new Agent($platform, 'gpt-4o-mini');
> ```

**`submit()` 的内部流程：** 你不需要手动管理消息数组。Chat 在每次 `submit()` 时自动完成以下步骤：

1. 从 Store 加载之前所有消息
2. 附加新的用户消息
3. 调用 Agent 获取 AI 回复
4. 保存完整消息历史（含用户消息和 AI 回复）到 Store
5. 返回 AI 回复（`AssistantMessage` 对象）

> **⚠️ 注意：** `InMemoryStore` 的数据仅存在于当前进程的内存中。进程结束后数据丢失，不适合跨请求的 Web 场景。适合脚本、测试和开发环境。

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

如果你的应用使用关系型数据库（MySQL/PostgreSQL），可以将对话存入数据库表。

```bash
composer require symfony/ai-doctrine-message-store
```

```php
<?php

use Doctrine\DBAL\DriverManager;
use Symfony\AI\Chat\Bridge\Doctrine\DoctrineDbalMessageStore;

// 创建数据库连接
$connection = DriverManager::getConnection([
    'url' => 'mysql://user:password@127.0.0.1:3306/myapp',
]);

$conversationId = 'user-123-conv-456';

// 创建 Chat Store（使用对话 ID 作为表名前缀）
$store = new DoctrineDbalMessageStore($conversationId, $connection);

// 初始化存储（首次使用时建表）
$store->setup();

// 后续使用方式与 Redis 相同
$chat = new Chat($agent, $store);
```

> **💡 提示：** 对话数据存入数据库后，可以配合其他业务数据做分析查询，比如"过去一个月用户最常问的问题是什么"。

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

除了上面详细讲解的存储方案，Chat 模块还支持 PSR-6 Cache 存储。以下是各存储后端的对比，帮助你选择最适合项目需求的方案。

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

### 存储后端选型指南

| 存储后端 | 适用场景 | 持久化 | 跨请求 | 性能 | 可查询 | 生产推荐 |
|---------|---------|--------|--------|------|--------|---------|
| **InMemory** | 开发、测试、CLI 脚本 | ❌ | ❌ | ⚡ 极快 | ❌ | 仅开发 |
| **Session** | Symfony 轻量应用 | ⚠️ 随 Session | ✅ | ⚡ 快 | ❌ | 原型/内部工具 |
| **Cache (PSR-6)** | 短期对话、有现成缓存 | ⚠️ 有 TTL | ✅ | ⚡ 快 | ❌ | 中小规模 |
| **Redis** | 高并发、实时应用 | ✅ | ✅ | ⚡ 极快 | ⚠️ 有限 | ✅ 推荐 |
| **MongoDB** | 灵活查询、大规模对话 | ✅ | ✅ | 🔶 快 | ✅ 强 | ✅ 推荐 |
| **Doctrine DBAL** | 关系型数据库为主的应用 | ✅ | ✅ | 🔶 中等 | ✅ 强 | ✅ 推荐 |

> **🏭 生产建议：** 根据你的实际需求选择存储后端：
> - **追求低延迟**（如实时聊天）→ Redis
> - **需要灵活查询和分析**（如对话审计）→ MongoDB 或 Doctrine DBAL
> - **已有关系型数据库**，不想引入新组件 → Doctrine DBAL
> - **需要水平扩展**（百万级对话）→ MongoDB（支持分片）
> - **临时/短期对话**（如表单助手）→ Cache (PSR-6)

> **🔒 安全建议：** 无论选择哪种存储后端，都要注意以下几点：
> - **对话 ID 管理**：使用不可预测的 UUID 或加密 token，避免用自增 ID 导致 IDOR 漏洞
> - **数据保留策略**：设置合理的过期时间，定期清理过期对话数据，遵守数据保护法规（如 GDPR）
> - **访问控制**：确保用户只能访问自己的对话，在服务端验证对话 ID 的归属关系
> - **敏感信息**：对话中可能包含用户敏感信息，确保存储层有适当的加密和访问限制

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Chat` | 对话管理器，自动管理消息历史和持久化 |
| `initiate(MessageBag)` | 初始化新对话（设置系统提示），会清空存储中的旧数据 |
| `submit(UserMessage)` | 提交用户消息，自动加载历史、调用 AI、保存结果，返回 `AssistantMessage` |
| `MessageStoreInterface` | 存储核心接口，定义 `save()` 和 `load()` 方法 |
| `ManagedStoreInterface` | 存储管理接口，定义 `setup()` 和 `drop()` 方法 |
| `MessageBag` | 消息集合，包含系统/用户/助手消息列表，自带 UUID 标识 |
| `MessageNormalizer` | 消息序列化器，各 Store 内部使用，自动处理 JSON 转换 |
| 对话隔离 | 通过为每个对话创建独立的 Store 实例来实现，每个 Store 使用不同的标识符 |

## 下一步

现在你的聊天机器人可以记住对话了。但如果用户问"今天天气怎么样"或"帮我查一下某个产品的库存"，AI 无法回答因为它没有工具。请看 [03-tool-augmented-assistant.md](./03-tool-augmented-assistant.md) 学习如何给 AI 添加工具能力。
