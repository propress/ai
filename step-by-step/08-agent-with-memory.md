# 带长期记忆的个性化助手

## 业务场景

你在做一个健身教练 AI。每个用户有不同的身体情况、健身目标和偏好。AI 需要记住这些信息，即使隔了几天用户再来对话，也能基于之前了解的信息给出个性化建议。

**典型应用：** 个性化推荐助手、健康顾问、学习辅导、个人秘书

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台（Anthropic / OpenAI / Gemini 等），发送消息并获取回复 |
| **Bridge（Anthropic）** | 本教程的主实现，通过 `PlatformFactory` 连接 Anthropic Claude |
| **Agent** | 智能体框架，管理输入处理器管线 |
| **Agent Memory** | 记忆系统（`Memory`、`MemoryProviderInterface`、`MemoryInputProcessor`） |
| **Store** | 向量存储抽象层，支持 25+ 后端（InMemory、PostgreSQL、Redis、Qdrant 等） |

## 项目流程图

```
┌──────────────┐
│   用户提问     │
└──────┬───────┘
       ▼
┌──────────────────────────────────────────────────────────┐
│              Agent InputProcessor 管线                     │
│                                                           │
│  ┌─────────────────────┐   ┌──────────────────────────┐  │
│  │ SystemPromptInput   │   │  MemoryInputProcessor    │  │
│  │ Processor           │   │                          │  │
│  │ (注入系统提示词)      │   │  遍历所有 MemoryProvider │  │
│  └─────────────────────┘   │  ┌────────────────────┐  │  │
│                             │  │ StaticMemory       │  │  │
│                             │  │ Provider           │  │  │
│                             │  │ (固定用户画像)      │  │  │
│                             │  └────────────────────┘  │  │
│                             │  ┌────────────────────┐  │  │
│                             │  │ EmbeddingProvider  │  │  │
│                             │  │                    │  │  │
│                             │  │ 用户消息 ──▶ 向量化  │  │  │
│                             │  │ 向量 ──▶ Store查询  │  │  │
│                             │  │ 结果 ──▶ Memory[]  │  │  │
│                             │  └────────────────────┘  │  │
│                             │  ┌────────────────────┐  │  │
│                             │  │ 自定义 Provider     │  │  │
│                             │  │ (数据库/缓存/API)   │  │  │
│                             │  └────────────────────┘  │  │
│                             └──────────────────────────┘  │
│                                        │                  │
│          记忆内容合并写入 System Message  │                  │
└──────────────────────────────────────────┼────────────────┘
                                           ▼
                                 ┌──────────────────┐
                                 │  Platform.invoke  │
                                 │ (Anthropic Claude)│
                                 └────────┬─────────┘
                                          ▼
                                 ┌──────────────────┐
                                 │    AI 生成回答     │
                                 │ (基于记忆 + 问题)  │
                                 └──────────────────┘
```

## 前置准备

### 环境要求

- PHP >= 8.2
- Composer

### 安装依赖

```bash
composer require symfony/ai-platform symfony/ai-anthropic-platform symfony/ai-agent symfony/ai-store
```

> **💡 提示：** 本教程使用 Anthropic Claude 作为主要平台。`symfony/ai-anthropic-platform` 是 Anthropic 的 Bridge 包。Symfony AI 采用 Bridge 架构——核心代码与平台实现分离，切换 AI 平台只需替换 Bridge 包，业务代码无需修改。

### 设置 API 密钥

```bash
export ANTHROPIC_API_KEY="sk-ant-your-api-key-here"
```

> **🔒 安全建议：** 永远不要将 API 密钥硬编码在源代码中。在 Symfony 项目中，应使用 `.env.local` 文件或服务器环境变量来管理密钥；在 CI/CD 中使用 Secrets 管理。

---

## Step 1：理解记忆系统的核心架构

在编写代码之前，先了解记忆系统的核心类和它们的协作关系。

### Memory 类 —— 记忆的值对象

`Memory` 是一个不可变的值对象，封装一段文本内容：

```php
use Symfony\AI\Agent\Memory\Memory;

// Memory 只是一个简单的内容包装器
$memory = new Memory('用户偏好：不吃辣，对海鲜过敏');
echo $memory->getContent(); // "用户偏好：不吃辣，对海鲜过敏"
```

### MemoryProviderInterface —— 记忆来源的契约

所有记忆提供者都实现这个接口，核心方法只有一个：

```php
namespace Symfony\AI\Agent\Memory;

use Symfony\AI\Agent\Input;

interface MemoryProviderInterface
{
    /**
     * @return list<Memory>
     */
    public function load(Input $input): array;
}
```

`load()` 方法接收当前用户输入（`Input`），返回与之相关的 `Memory` 数组。不同实现的检索策略完全不同：

| 实现类 | 检索策略 |
|--------|---------|
| `StaticMemoryProvider` | 忽略输入内容，始终返回全部预设记忆 |
| `EmbeddingProvider` | 将输入向量化，从向量存储中检索语义相似的记忆 |
| 自定义实现 | 查询数据库、调用 API、读取缓存等任意逻辑 |

### MemoryInputProcessor —— 记忆注入引擎

`MemoryInputProcessor` 实现了 `InputProcessorInterface`，它是 Agent 输入处理管线的一环。在每次对话时：

1. 遍历所有注册的 `MemoryProviderInterface`
2. 调用每个 Provider 的 `load()` 方法收集记忆
3. 将所有记忆合并，以 `# Conversation Memory` 标题写入 System Message
4. AI 模型在生成回答时会参考这些记忆上下文

```php
use Symfony\AI\Agent\Memory\MemoryInputProcessor;

// 支持注入多个记忆源
$memoryProcessor = new MemoryInputProcessor([
    $staticProvider,
    $embeddingProvider,
    $customProvider,
]);
```

> **💡 提示：** `MemoryInputProcessor` 支持通过 `use_memory` 选项动态控制是否启用记忆。设置 `$input->setOptions(['use_memory' => false])` 可在特定请求中跳过记忆注入，适用于简单的一次性查询场景。

---

## Step 2：静态记忆 —— 最简单的个性化

如果用户的关键信息是已知的（从数据库加载），可以用 `StaticMemoryProvider` 直接注入。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;
use Symfony\AI\Agent\Memory\StaticMemoryProvider;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 使用 Anthropic Claude 平台
$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY']);

// 1. 系统提示
$systemPrompt = new SystemPromptInputProcessor(
    '你是一个专业的健身教练 AI。根据用户的个人情况给出定制化的建议。'
    . '回答风格：专业但友好，给出具体可执行的建议。'
);

// 2. 从数据库加载的用户画像作为静态记忆
// 在真实场景中，这些数据来自用户注册信息或之前的对话总结
$userProfile = new StaticMemoryProvider(
    '用户姓名：小王',
    '年龄：28岁，男性',
    '身高：175cm，体重：80kg',
    '健身目标：减脂增肌',
    '健身经验：初学者，刚开始健身两周',
    '可用器材：家里有一对哑铃和瑜伽垫',
    '饮食偏好：不吃辣，对海鲜过敏',
    '每周可用训练时间：工作日每天 30 分钟，周末 1 小时',
);

// 3. 创建记忆处理器
$memoryProcessor = new MemoryInputProcessor([$userProfile]);

// 4. 创建带记忆的 Agent
$agent = new Agent(
    $platform,
    'claude-sonnet-4-20250514',
    [$systemPrompt, $memoryProcessor], // 输入处理器：注入系统提示和记忆
);

// 5. 用户对话 —— AI 自动使用记忆信息
$questions = [
    '今天应该练什么？',
    '训练完吃什么比较好？',
    '我觉得哑铃太轻了，有没有什么动作可以加大难度？',
];

foreach ($questions as $question) {
    echo "小王：{$question}\n";

    $result = $agent->call(new MessageBag(Message::ofUser($question)));
    echo "教练：" . $result->getContent() . "\n\n";
}
```

**效果：** AI 回答时会自动考虑用户是初学者、只有哑铃和瑜伽垫、对海鲜过敏等信息，给出完全定制化的建议。

> **💡 提示：** `StaticMemoryProvider` 内部会将所有条目格式化为 `## Static Memory` 标题下的列表。这些内容会在每次对话中完整注入 System Message，不会根据用户提问做筛选。因此静态记忆条目不宜过多——建议控制在 20 条以内，避免占用过多上下文窗口。

---

## Step 3：深入理解 EmbeddingProvider —— 语义记忆的核心

静态记忆适合固定信息。但真实场景中，用户在对话中会透露很多信息，这些信息也需要被记住。`EmbeddingProvider` 是记忆系统中最强大的组件，它将历史对话向量化存入向量数据库，之后每次新对话都会自动检索语义相关的历史内容。

### EmbeddingProvider 的工作原理

`EmbeddingProvider` 构造函数需要三个核心依赖：

```php
use Symfony\AI\Agent\Memory\EmbeddingProvider;

$embeddingMemory = new EmbeddingProvider(
    $platform,        // PlatformInterface —— 用于调用嵌入模型生成向量
    $embeddingsModel, // Model —— 嵌入模型实例（如 text-embedding-3-small）
    $vectorStore,     // StoreInterface —— 存储和检索向量文档的后端
);
```

当 `load()` 被调用时，内部执行以下流程：

```
1. 从 Input 中提取最后一条用户消息的文本内容
2. 调用 Platform + 嵌入模型，将文本转换为向量
3. 用该向量向 StoreInterface 发起 VectorQuery 相似度查询
4. 将查询结果的 Metadata 序列化为 Memory 对象返回
```

> **⚠️ 注意：** `EmbeddingProvider` 每次调用都会触发一次嵌入 API 请求（将用户消息向量化）。在高并发场景下，这意味着额外的延迟和成本。可以考虑对热门查询的向量结果做缓存（如使用 `Symfony\AI\Store\Bridge\Cache\Store` 包装底层 Store）。

### 使用 InMemory Store 快速开发

先用内存存储验证记忆逻辑，再切换到生产级后端：

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Memory\EmbeddingProvider;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Store\Bridge\Cache\Store;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentIndexer;
use Symfony\AI\Store\Indexer\DocumentProcessor;
use Symfony\Component\Cache\Adapter\ArrayAdapter;
use Symfony\Component\Uid\Uuid;

$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY']);

// ===== 1. 建立记忆存储 =====

$store = new Store(new ArrayAdapter());
$vectorizer = new Vectorizer($platform, $embeddingModel = 'text-embedding-3-small');

// 模拟从数据库加载的过往对话记录（现实中这些来自 Chat 组件保存的历史）
$pastConversations = [
    ['role' => 'user', 'time' => '2025-01-10', 'content' => '我最近膝盖有点不舒服，深蹲的时候疼'],
    ['role' => 'assistant', 'time' => '2025-01-10', 'content' => '建议暂停深蹲，改做箱式深蹲或腿举机，注意膝盖不要超过脚尖'],
    ['role' => 'user', 'time' => '2025-01-12', 'content' => '我周三和周五晚上有空，想固定这两天训练'],
    ['role' => 'assistant', 'time' => '2025-01-12', 'content' => '好的，周三安排上肢训练，周五安排下肢训练'],
    ['role' => 'user', 'time' => '2025-01-14', 'content' => '我买了一条弹力带，可以加入训练吗？'],
    ['role' => 'assistant', 'time' => '2025-01-14', 'content' => '当然可以，弹力带适合热身和辅助训练'],
];

// 将历史对话索引到向量存储
$documents = [];
foreach ($pastConversations as $msg) {
    $documents[] = new TextDocument(
        id: Uuid::v4(),
        content: "时间：{$msg['time']}，角色：{$msg['role']}，内容：{$msg['content']}",
        metadata: new Metadata($msg),
    );
}

$indexer = new DocumentIndexer(new DocumentProcessor($vectorizer, $store));
$indexer->index($documents);

echo "已加载 " . count($documents) . " 条历史记录到记忆系统\n\n";

// ===== 2. 创建带嵌入记忆的 Agent =====

$embeddingsModel = $platform->getModelCatalog()->getModel($embeddingModel);
$embeddingMemory = new EmbeddingProvider($platform, $embeddingsModel, $store);
$memoryProcessor = new MemoryInputProcessor([$embeddingMemory]);

$systemPrompt = new SystemPromptInputProcessor(
    '你是一个专业的健身教练。根据用户的历史对话和当前问题给出建议。'
    . '参考记忆中的信息来个性化你的回答。'
);

$agent = new Agent($platform, 'claude-sonnet-4-20250514', [$systemPrompt, $memoryProcessor]);

// ===== 3. 新对话 —— AI 自动检索相关记忆 =====

$questions = [
    '今天是周五，帮我安排一下今晚的训练吧',
    // → AI 会检索到"周五安排下肢训练"以及"膝盖不舒服"的记忆

    '深蹲可以做了吗？',
    // → AI 会检索到膝盖问题的记忆

    '弹力带可以怎么用在热身里？',
    // → AI 会检索到买了弹力带的记忆
];

foreach ($questions as $question) {
    echo "用户：{$question}\n";

    $result = $agent->call(new MessageBag(Message::ofUser($question)));
    echo "教练：" . $result->getContent() . "\n\n";
}
```

**效果：** 用户问"今天是周五"时，AI 会想起之前约定了周五练下肢。用户问"深蹲可以做了吗"时，AI 会想起膝盖的问题。

> **🏭 生产建议：** `Cache\Store`（基于 `ArrayAdapter`）适合开发和测试。生产环境中应切换到持久化后端（PostgreSQL、Redis 等），参见 Step 5 的生产级向量存储方案。

---

## Step 4：组合静态记忆和嵌入记忆

在真实应用中，通常同时使用两种记忆源：用户档案作为固定上下文，历史对话作为动态检索补充。

```php
use Symfony\AI\Agent\Memory\EmbeddingProvider;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;
use Symfony\AI\Agent\Memory\StaticMemoryProvider;

// 静态记忆：用户档案（确定不变的信息）
$profileMemory = new StaticMemoryProvider(
    '姓名：小王，28岁男性',
    '身高175cm，体重80kg',
    '健身目标：减脂增肌',
);

// 嵌入记忆：历史对话（动态积累的信息）
$conversationMemory = new EmbeddingProvider($platform, $embeddingsModel, $store);

// 组合两种记忆 —— MemoryInputProcessor 会依次调用每个 Provider
$memoryProcessor = new MemoryInputProcessor([
    $profileMemory,        // 固定的用户画像，每次全量注入
    $conversationMemory,   // 基于语义检索的历史记忆，按相关性筛选
]);

$agent = new Agent($platform, 'claude-sonnet-4-20250514', [$systemPrompt, $memoryProcessor]);
```

> **💡 提示：** Provider 的注册顺序即记忆在 System Message 中的排列顺序。建议将静态记忆放在前面（提供稳定上下文），嵌入记忆放在后面（提供针对性信息）。AI 模型通常对 System Message 开头和结尾的内容赋予更高权重。

---

## Step 5：生产级向量存储 —— PostgreSQL pgvector

InMemory / Cache Store 适合开发测试，生产环境需要持久化、可扩展的向量数据库。

### 使用 PostgreSQL + pgvector

pgvector 是 PostgreSQL 的向量搜索扩展，适合已使用 PostgreSQL 的项目——无需引入新的数据库基础设施。

```bash
composer require symfony/ai-postgres-store
```

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Memory\EmbeddingProvider;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;
use Symfony\AI\Agent\Memory\StaticMemoryProvider;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Store\Bridge\Postgres\Store;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentIndexer;
use Symfony\AI\Store\Indexer\DocumentProcessor;
use Symfony\Component\Uid\Uuid;

// ===== 1. 连接 Anthropic 平台 =====

$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY']);

// ===== 2. 配置 PostgreSQL pgvector 存储 =====

$pdo = new \PDO(
    'pgsql:host=localhost;dbname=fitness_ai',
    $_ENV['DB_USER'],
    $_ENV['DB_PASSWORD'],
);

$store = Store::fromPdo(
    $pdo,
    'conversation_memories', // 表名
    'embedding',             // 向量列名
);

// 首次运行：创建表和索引
// vector_size 取决于嵌入模型：text-embedding-3-small = 1536 维
$store->setup(['vector_size' => 1536]);

// ===== 3. 索引历史对话 =====

$vectorizer = new Vectorizer($platform, 'text-embedding-3-small');

$pastConversations = [
    ['role' => 'user', 'time' => '2025-01-10', 'content' => '我最近膝盖有点不舒服，深蹲的时候疼'],
    ['role' => 'assistant', 'time' => '2025-01-10', 'content' => '建议暂停深蹲，改做箱式深蹲或腿举机'],
    ['role' => 'user', 'time' => '2025-01-12', 'content' => '我周三和周五晚上有空，想固定这两天训练'],
    ['role' => 'assistant', 'time' => '2025-01-12', 'content' => '好的，周三安排上肢训练，周五安排下肢训练'],
    ['role' => 'user', 'time' => '2025-01-14', 'content' => '我买了一条弹力带，可以加入训练吗？'],
    ['role' => 'assistant', 'time' => '2025-01-14', 'content' => '当然可以，弹力带适合热身和辅助训练'],
    ['role' => 'user', 'time' => '2025-01-15', 'content' => '我对海鲜过敏，蛋白粉可以替代吗？'],
    ['role' => 'assistant', 'time' => '2025-01-15', 'content' => '可以，推荐乳清蛋白粉，训练后 30 分钟内服用效果最佳'],
];

$documents = [];
foreach ($pastConversations as $msg) {
    $documents[] = new TextDocument(
        id: Uuid::v4(),
        content: "时间：{$msg['time']}，角色：{$msg['role']}，内容：{$msg['content']}",
        metadata: new Metadata($msg),
    );
}

$indexer = new DocumentIndexer(new DocumentProcessor($vectorizer, $store));
$indexer->index($documents);

echo "已索引 " . count($documents) . " 条记忆到 PostgreSQL\n\n";

// ===== 4. 创建带混合记忆的 Agent =====

$embeddingsModel = $platform->getModelCatalog()->getModel('text-embedding-3-small');

$profileMemory = new StaticMemoryProvider(
    '用户姓名：小王，28岁男性',
    '身高175cm，体重80kg',
    '健身目标：减脂增肌',
    '健身经验：初学者',
);

$conversationMemory = new EmbeddingProvider($platform, $embeddingsModel, $store);

$memoryProcessor = new MemoryInputProcessor([
    $profileMemory,
    $conversationMemory,
]);

$systemPrompt = new SystemPromptInputProcessor(
    '你是一个专业的健身教练。根据用户的个人档案和历史对话给出个性化建议。'
    . '回答风格：专业但友好，给出具体可执行的建议。'
);

$agent = new Agent($platform, 'claude-sonnet-4-20250514', [$systemPrompt, $memoryProcessor]);

// ===== 5. 对话 =====

$questions = [
    '今天是周五，帮我安排一下今晚的训练吧',
    '深蹲可以做了吗？',
    '训练后吃什么补充蛋白质？',
];

foreach ($questions as $question) {
    echo "小王：{$question}\n";

    $result = $agent->call(new MessageBag(Message::ofUser($question)));
    echo "教练：" . $result->getContent() . "\n\n";
}
```

> **⚠️ 注意：** `$store->setup()` 会创建表和 pgvector 扩展。只需在首次部署时执行一次。生产环境建议通过数据库迁移（如 Doctrine Migrations）管理 schema，而非在运行时调用 `setup()`。

### 其他生产级向量存储选项

Symfony AI Store 组件支持 25+ 种向量数据库后端。以下是常用选项：

| 后端 | 安装包 | 适用场景 |
|------|--------|---------|
| **PostgreSQL pgvector** | `symfony/ai-postgres-store` | 已有 PostgreSQL 基础设施，适合中小规模 |
| **Redis** | `symfony/ai-redis-store` | 低延迟要求，已有 Redis 集群 |
| **Qdrant** | `symfony/ai-qdrant-store` | 专业向量数据库，大规模高性能检索 |
| **Elasticsearch** | `symfony/ai-elasticsearch-store` | 需要混合文本 + 向量搜索 |
| **Pinecone** | `symfony/ai-pinecone-store` | 全托管云服务，零运维 |
| **MongoDB Atlas** | `symfony/ai-mongodb-store` | 已有 MongoDB，利用 Atlas Vector Search |
| **SQLite** | `symfony/ai-sqlite-store` | 嵌入式场景、边缘设备 |

> **🏭 生产建议：** 向量存储的选择主要取决于现有基础设施。如果你已经用 PostgreSQL，pgvector 是零成本方案；如果需要毫秒级延迟和十亿级向量，考虑 Qdrant 或 Pinecone 等专用方案。所有后端都实现了 `StoreInterface`，切换时只需更换实例化代码，`EmbeddingProvider` 的使用方式完全不变。

---

## Step 6：自定义 MemoryProvider —— 灵活接入任意数据源

内置的 `StaticMemoryProvider` 和 `EmbeddingProvider` 覆盖了最常见的场景。当你需要从特殊数据源加载记忆时，可以实现 `MemoryProviderInterface` 创建自定义 Provider。

### 示例：从数据库加载用户偏好的 Provider

```php
<?php

namespace App\Memory;

use Symfony\AI\Agent\Input;
use Symfony\AI\Agent\Memory\Memory;
use Symfony\AI\Agent\Memory\MemoryProviderInterface;

final class DatabaseUserProfileProvider implements MemoryProviderInterface
{
    public function __construct(
        private readonly \PDO $pdo,
        private readonly string $userId,
    ) {
    }

    /**
     * @return list<Memory>
     */
    public function load(Input $input): array
    {
        $stmt = $this->pdo->prepare(
            'SELECT key, value FROM user_preferences WHERE user_id = :userId'
        );
        $stmt->execute(['userId' => $this->userId]);

        $rows = $stmt->fetchAll(\PDO::FETCH_ASSOC);
        if ([] === $rows) {
            return [];
        }

        $content = '## 用户偏好设置' . \PHP_EOL;
        foreach ($rows as $row) {
            $content .= \PHP_EOL . '- ' . $row['key'] . '：' . $row['value'];
        }

        return [new Memory($content)];
    }
}
```

### 示例：带 TTL 的时效性记忆 Provider

```php
<?php

namespace App\Memory;

use Symfony\AI\Agent\Input;
use Symfony\AI\Agent\Memory\Memory;
use Symfony\AI\Agent\Memory\MemoryProviderInterface;

/**
 * 只加载指定天数内的记忆，自动过滤过期内容。
 */
final class RecentEventsProvider implements MemoryProviderInterface
{
    public function __construct(
        private readonly \PDO $pdo,
        private readonly string $userId,
        private readonly int $retentionDays = 30,
    ) {
    }

    /**
     * @return list<Memory>
     */
    public function load(Input $input): array
    {
        $stmt = $this->pdo->prepare(
            'SELECT event_type, description, created_at FROM user_events '
            . 'WHERE user_id = :userId AND created_at > :since '
            . 'ORDER BY created_at DESC LIMIT 20'
        );
        $stmt->execute([
            'userId' => $this->userId,
            'since' => (new \DateTimeImmutable("-{$this->retentionDays} days"))->format('Y-m-d'),
        ]);

        $rows = $stmt->fetchAll(\PDO::FETCH_ASSOC);
        if ([] === $rows) {
            return [];
        }

        $content = '## 近期事件（最近 ' . $this->retentionDays . ' 天）' . \PHP_EOL;
        foreach ($rows as $row) {
            $content .= \PHP_EOL . '- [' . $row['created_at'] . '] '
                . $row['event_type'] . '：' . $row['description'];
        }

        return [new Memory($content)];
    }
}
```

### 将自定义 Provider 注册到 Agent

```php
use App\Memory\DatabaseUserProfileProvider;
use App\Memory\RecentEventsProvider;
use Symfony\AI\Agent\Memory\EmbeddingProvider;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;

$memoryProcessor = new MemoryInputProcessor([
    new DatabaseUserProfileProvider($pdo, $userId),  // 从数据库加载用户画像
    new RecentEventsProvider($pdo, $userId, 30),     // 最近 30 天的事件
    new EmbeddingProvider($platform, $model, $store), // 语义检索历史对话
]);

$agent = new Agent($platform, 'claude-sonnet-4-20250514', [$systemPrompt, $memoryProcessor]);
```

> **💡 提示：** 自定义 Provider 的 `load()` 方法可以访问 `$input->getMessageBag()` 获取完整的用户消息，根据消息内容做条件检索。例如，只有当用户提到"饮食"相关话题时才加载饮食偏好记忆，减少不必要的上下文注入。

---

## 完整示例：个人学习辅导助手

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Memory\EmbeddingProvider;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;
use Symfony\AI\Agent\Memory\StaticMemoryProvider;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Store\Bridge\Cache\Store;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentIndexer;
use Symfony\AI\Store\Indexer\DocumentProcessor;
use Symfony\Component\Cache\Adapter\ArrayAdapter;
use Symfony\Component\Uid\Uuid;

// ===== 平台初始化 =====

$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY']);

// ===== 学生档案（静态记忆）=====

$studentProfile = new StaticMemoryProvider(
    '学生姓名：小李，高二学生',
    '目标：准备高考，目标大学是浙江大学计算机系',
    '强势科目：数学和物理',
    '弱势科目：英语（尤其是阅读理解和写作）',
    '每天可学习时间：放学后 3 小时',
    '学习风格偏好：喜欢通过做题来学习，不喜欢纯看书',
    '上次模考成绩：数学 135，物理 92，英语 105（满分 150）',
);

// ===== 历史辅导记录（嵌入记忆）=====

$store = new Store(new ArrayAdapter());
$vectorizer = new Vectorizer($platform, 'text-embedding-3-small');

$tutorHistory = [
    ['time' => '2025-01-06', 'content' => '老师建议小李每天用 40 分钟做英语阅读真题，重点练习长难句分析'],
    ['time' => '2025-01-08', 'content' => '小李反馈数学函数题很顺利，但英语完形填空错了 5 个'],
    ['time' => '2025-01-10', 'content' => '给小李推荐了《高考英语阅读理解满分攻略》，让他每天精读一篇'],
    ['time' => '2025-01-12', 'content' => '物理力学部分小李掌握很好，电磁学需要加强'],
];

$documents = [];
foreach ($tutorHistory as $record) {
    $documents[] = new TextDocument(
        id: Uuid::v4(),
        content: "辅导记录 [{$record['time']}]：{$record['content']}",
        metadata: new Metadata($record),
    );
}

$indexer = new DocumentIndexer(new DocumentProcessor($vectorizer, $store));
$indexer->index($documents);

// ===== 创建 Agent =====

$embeddingsModel = $platform->getModelCatalog()->getModel('text-embedding-3-small');
$historyMemory = new EmbeddingProvider($platform, $embeddingsModel, $store);

$memoryProcessor = new MemoryInputProcessor([
    $studentProfile,  // 固定的学生档案
    $historyMemory,   // 动态检索的辅导历史
]);

$systemPrompt = new SystemPromptInputProcessor(
    '你是一个高考备考辅导老师。根据学生的情况和历史辅导记录制定学习建议。'
    . '特别关注学生的弱势科目。建议要具体、可执行，考虑到学生的时间安排。'
);

$agent = new Agent($platform, 'claude-sonnet-4-20250514', [$systemPrompt, $memoryProcessor]);

// ===== 模拟跨天的对话 =====

echo "=== 周一 ===\n\n";

$result = $agent->call(new MessageBag(
    Message::ofUser('老师，这周的学习计划怎么安排？'),
));
echo "老师：" . $result->getContent() . "\n\n";

echo "=== 周三 ===\n\n";

$result = $agent->call(new MessageBag(
    Message::ofUser('我觉得英语阅读还是看不懂长句子，有什么办法？'),
));
echo "老师：" . $result->getContent() . "\n\n";

echo "=== 周五 ===\n\n";

$result = $agent->call(new MessageBag(
    Message::ofUser('这次物理小测考了 88 分，感觉退步了怎么办？'),
));
echo "老师：" . $result->getContent() . "\n\n";
```

---

## 其他实现方案

### 使用 OpenAI 平台

将 Anthropic 替换为 OpenAI，只需更换 Bridge 包和平台初始化代码：

```bash
# 替换 Bridge 包
composer require symfony/ai-open-ai-platform
# composer remove symfony/ai-anthropic-platform  # 如果不再需要
```

```php
// 修改前（Anthropic）
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY']);
$agent = new Agent($platform, 'claude-sonnet-4-20250514', $inputProcessors);

// 修改后（OpenAI）
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);
$agent = new Agent($platform, 'gpt-4o-mini', $inputProcessors);
```

> **💡 提示：** 注意对比两个方案——除了 `use` 语句、API 密钥和模型名称不同，其余代码（`Agent`、`MemoryInputProcessor`、`EmbeddingProvider`、`MessageBag`）完全一致。这就是 Symfony AI Platform 抽象层的价值。

### 使用轻量模型降低成本

对于成本敏感的场景，可以选择更轻量的模型：

```php
// Anthropic 轻量模型
$agent = new Agent($platform, 'claude-haiku-4-20250514', $inputProcessors);

// OpenAI 轻量模型
$agent = new Agent($platform, 'gpt-4o-mini', $inputProcessors);
```

> **🏭 生产建议：** 聊天模型和嵌入模型可以来自不同平台。例如，用 Anthropic Claude 做对话生成，用 OpenAI `text-embedding-3-small` 做向量嵌入——只需为嵌入单独创建一个 OpenAI Platform 实例传给 `Vectorizer` 和 `EmbeddingProvider`。

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Memory` | 不可变值对象，封装一段记忆文本内容 |
| `MemoryProviderInterface` | 记忆源的契约，核心方法 `load(Input): array<Memory>` |
| `StaticMemoryProvider` | 静态记忆，始终返回全部预设内容，适合固定的用户画像 |
| `EmbeddingProvider` | 嵌入记忆，依赖 Platform + Model + Store 三个组件，按语义相似度检索 |
| `MemoryInputProcessor` | 记忆注入引擎，遍历所有 Provider 并将记忆写入 System Message |
| 自定义 Provider | 实现 `MemoryProviderInterface` 即可接入任意数据源 |
| 向量存储后端 | 25+ 种选择，InMemory 用于开发，pgvector / Redis / Qdrant 用于生产 |
| 记忆注入流程 | Agent 管线中：用户输入 → Provider 检索 → 合并到 System Message → AI 推理 |

### 生产环境最佳实践

| 主题 | 建议 |
|------|------|
| **嵌入模型选择** | `text-embedding-3-small`（1536 维）性价比最高；`text-embedding-3-large`（3072 维）精度更高但成本翻倍 |
| **记忆数量控制** | 静态记忆 ≤ 20 条；嵌入检索结果建议限制在 5-10 条（通过 Store 查询的 limit 参数控制） |
| **记忆 TTL** | 对过期记忆做清理或降权，避免"三年前说过一次喜欢跑步"影响当前建议 |
| **嵌入缓存** | 高频查询做向量缓存，避免重复调用嵌入 API（使用 `Cache\Store` 包装） |
| **记忆相关性** | 向量检索结果有相似度评分（`VectorDocument::getScore()`），可在自定义 Provider 中按阈值过滤低相关性结果 |
| **记忆规模化** | 十万级记忆考虑 pgvector + HNSW 索引；百万级以上考虑 Qdrant / Pinecone 等专用方案 |
| **成本优化** | 嵌入调用按 Token 计费，每次对话一次嵌入请求；批量索引用 `Vectorizer` 的批处理模式减少 API 调用次数 |

## 下一步

如果你的 AI 需要上网搜索实时信息并整理成报告，请看 [09-web-research-assistant.md](./09-web-research-assistant.md)。
