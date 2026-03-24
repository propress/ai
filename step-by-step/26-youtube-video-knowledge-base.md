# YouTube 视频知识库

## 业务场景

你的团队通过 YouTube 频道做技术教育。频道有上百个视频教程，每个 20-60 分钟。学员经常问："哪个视频讲了 XXX？""第几分钟讲到了这个知识点？"现在你要做一个智能问答系统：自动获取视频字幕，建立知识库索引，学员可以用自然语言提问，AI 精准定位到具体视频和时间段。

**典型应用：** 视频课程智能搜索、会议录像知识库、播客内容检索、培训资料问答

## 涉及模块

| 模块 | 包名 | 用途 |
|------|------|------|
| **Platform** | `symfony/ai-platform` | 核心抽象层——`PlatformInterface`、消息系统、`DeferredResult` |
| **Bridge（Mistral）** | `symfony/ai-mistral-platform` | 主问答引擎——Mistral 在知识问答场景性价比极高 |
| **Bridge（OpenAI）** | `symfony/ai-open-ai-platform` | 嵌入模型（`text-embedding-3-small`），向量化字幕内容 |
| **Agent** | `symfony/ai-agent` | 编排问答流程——自动调用搜索工具回答学员问题 |
| **Agent Bridge YouTube** | `symfony/ai-agent-youtube` | `YoutubeTranscriber` 工具——自动获取 YouTube 视频字幕 |
| **Agent Bridge SimilaritySearch** | `symfony/ai-similarity-search-tool` | `SimilaritySearch` 工具——从向量库中语义检索 |
| **Store** | `symfony/ai-store` | 向量存储抽象——文档索引、分块、语义检索 |
| **Store Bridge（Postgres）** | `symfony/ai-postgres-store` | 本教程的向量数据库——基于 pgvector，适合已有 PostgreSQL 的团队 |
| **Chat** | `symfony/ai-chat` | 对话管理——支持多轮追问，自动持久化消息历史 |

## 架构概述

本教程综合运用了 Symfony AI 的多个模块：

- **YoutubeTranscriber**：Agent Bridge 提供的工具，底层使用 `mrmysql/youtube-transcript` 库获取 YouTube 视频的自动字幕。标注了 `#[AsTool]` 属性，可以直接注册到 Agent 的 Toolbox 中。
- **ChunkTransformer**：Store 模块的文档转换器，将长文本按指定长度分段，支持重叠（overlap）确保段落间信息不丢失。每个分段保留原始文档的元数据。
- **Agent + Toolbox**：Agent 是 Symfony AI 的智能编排器。它接收用户消息后，自主决定是否以及何时调用工具（如 `SimilaritySearch`），然后基于工具返回的结果生成最终回答。
- **Chat 对话管理**：`ChatInterface` 负责加载和保存对话历史。配合 Agent 使用时，学员可以基于上一个回答继续追问——Chat 自动将历史消息传给 Agent。

## 项目流程图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                       YouTube 视频知识库完整流程                               │
└──────────────────────────────────────────────────────────────────────────────┘

 ┌────────── 知识库构建（离线）──────────┐   ┌────────── 智能问答（在线）──────────┐
 │                                       │   │                                    │
 │  YouTube 视频                          │   │  学员提问                           │
 │      │                                │   │      │                             │
 │      ▼                                │   │      ▼                             │
 │  YoutubeTranscriber                   │   │  Chat::loadOrCreate()              │
 │  (获取字幕文本)                        │   │  (加载对话历史)                     │
 │      │                                │   │      │                             │
 │      ▼                                │   │      ▼                             │
 │  ChunkTransformer                     │   │  Agent::call()                     │
 │  (分段 500 字/段，重叠 50 字)          │   │  (Mistral 推理)                    │
 │      │                                │   │      │                             │
 │      ▼                                │   │      ▼                             │
 │  Indexer                              │   │  SimilaritySearch 工具             │
 │  (OpenAI 嵌入 → pgvector 存储)        │   │  (从 pgvector 语义检索)            │
 │      │                                │   │      │                             │
 │      ▼                                │   │      ▼                             │
 │  知识库就绪 ✅                         │   │  生成回答 + 引用视频来源            │
 └───────────────────────────────────────┘   │      │                             │
                                             │      ▼                             │
                                             │  Chat::save()                      │
                                             │  (保存对话，支持多轮追问)           │
                                             └────────────────────────────────────┘
```

**核心类的关系：**

```
Agent（智能编排器）
 ├── Platform（Mistral，问答推理引擎）
 ├── SystemPromptInputProcessor（角色设定）
 ├── AgentProcessor（工具调用处理）
 │    └── Toolbox
 │         └── SimilaritySearch 工具
 │              ├── VectorizerInterface（查询向量化）
 │              └── StoreInterface（pgvector 检索）
 └── Chat（对话持久化）
      └── MessageStoreInterface（Redis / DBAL / InMemory）
```

## 前置准备

### 环境要求

- PHP >= 8.2
- Composer
- PostgreSQL 15+（启用 pgvector 扩展）

### 安装依赖

```bash
# 核心平台 + Mistral Bridge（问答引擎）
composer require symfony/ai-platform symfony/ai-mistral-platform

# OpenAI Bridge（嵌入模型）
composer require symfony/ai-open-ai-platform

# Agent + 工具
composer require symfony/ai-agent
composer require symfony/ai-agent-youtube           # YouTube 字幕获取
composer require symfony/ai-similarity-search-tool  # 向量相似搜索

# 向量存储（PostgreSQL + pgvector）
composer require symfony/ai-store symfony/ai-postgres-store

# 对话持久化
composer require symfony/ai-chat
```

> **💡 提示：** 为什么用 Mistral 做问答而不是 OpenAI？Mistral 的 `mistral-small` 模型在知识问答场景下性价比极高（约为 GPT-4o 的 1/5 价格），且对中文支持良好。Bridge 架构下切换模型只需改一行代码。

### 启用 pgvector 扩展

```sql
-- 在 PostgreSQL 中启用向量扩展
CREATE EXTENSION IF NOT EXISTS vector;
```

### 设置 API 密钥

```bash
export MISTRAL_API_KEY="your-mistral-api-key"
export OPENAI_API_KEY="sk-your-api-key-here"  # 嵌入模型
```

> **🔒 安全建议：** 如果你的视频内容包含付费课程，API 密钥泄露可能导致课程内容被批量提取。建议在生产环境中通过 Symfony Vault 或 HashiCorp Vault 管理密钥，并对知识库 API 设置访问鉴权。

---

## Step 1：创建多平台环境

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\Mistral\PlatformFactory as MistralPlatformFactory;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory as OpenAiPlatformFactory;
use Symfony\AI\Platform\Event\ResultEvent;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create();
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

// Token 统计
$tokenStats = ['mistral' => 0, 'openai' => 0];
$dispatcher->addListener(ResultEvent::class, function (ResultEvent $event) use (&$tokenStats) {
    $model = $event->result->getMetadata()->get('model') ?? '';
    $usage = $event->result->getMetadata()->get('token_usage');
    if (null === $usage) {
        return;
    }
    if (str_contains($model, 'mistral')) {
        $tokenStats['mistral'] += $usage->totalTokens;
    } else {
        $tokenStats['openai'] += $usage->totalTokens;
    }
});

// Mistral —— 问答推理引擎
$mistralPlatform = MistralPlatformFactory::create(
    $_ENV['MISTRAL_API_KEY'],
    $httpClient,
    eventDispatcher: $dispatcher,
);

// OpenAI —— 嵌入模型
$openaiPlatform = OpenAiPlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    $httpClient,
);
```

> **📝 知识扩展：** 所有 Bridge 的 `PlatformFactory::create()` 都接受 `EventDispatcherInterface` 参数。事件系统贯穿整个调用链——`InvocationEvent`（请求发出前）和 `ResultEvent`（响应返回后）——可用于日志、Token 统计、动态参数修改、错误拦截等。共享同一个 `EventDispatcher` 实例意味着所有平台的调用都能被统一监控。

---

## Step 2：获取视频字幕

`YoutubeTranscriber` 是 Agent Bridge 提供的工具，底层通过 YouTube 的内置字幕 API 获取转录文本。

```php
<?php

use Symfony\AI\Agent\Bridge\Youtube\YoutubeTranscriber;

$transcriber = new YoutubeTranscriber($httpClient);

// 获取单个视频的字幕
$videoId = 'dQw4w9WgXcQ';
$transcript = ($transcriber)($videoId);

echo "字幕内容（前 500 字）：\n";
echo mb_substr($transcript, 0, 500) . "...\n";
echo '总长度：' . mb_strlen($transcript) . " 字\n";
```

> **⚠️ 注意：** `YoutubeTranscriber` 依赖视频是否有字幕（自动生成或手动上传）。没有字幕的视频会返回空结果。另外，YouTube 的自动字幕质量因语言而异——中文的自动字幕准确率约 85-90%，英文约 95%+。

> **📝 知识扩展：** `YoutubeTranscriber` 标注了 `#[AsTool('youtube_transcript', 'Fetches the transcript of a YouTube video')]`——这意味着它可以直接注册到 Agent 的 Toolbox 中，让 Agent 自主决定何时获取字幕。但在知识库构建场景中，我们通常在离线批处理中直接调用它。

---

## Step 3：字幕分段与向量化索引

将长字幕按段落切分，每段关联视频 ID 和标题等元数据，然后通过 `Indexer` 自动嵌入并存入 pgvector。

```php
<?php

use Symfony\AI\Store\Bridge\Postgres\Store;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Transformer\ChunkTransformer;
use Symfony\AI\Store\Indexer;

// PostgreSQL 向量存储（pgvector）
$pdo = new \PDO('pgsql:host=localhost;dbname=video_kb', 'postgres', 'password');
$store = new Store($pdo, 'video_chunks');
$store->setup(['vector_size' => 1536]); // text-embedding-3-small 维度

// 文档分块器：每 500 字一段，重叠 50 字
$chunker = new ChunkTransformer(maxLength: 500, overlap: 50);

// 创建 Indexer（嵌入 + 存储一步完成）
$indexer = new Indexer($openaiPlatform, 'text-embedding-3-small', $store);

// 视频列表（实际可从 YouTube Data API 获取）
$videos = [
    ['id' => 'abc123', 'title' => 'PHP 8.4 新特性详解'],
    ['id' => 'def456', 'title' => 'Symfony 7 快速上手'],
    ['id' => 'ghi789', 'title' => 'AI Agent 开发实战'],
];

$totalChunks = 0;

foreach ($videos as $video) {
    echo "📹 处理视频：{$video['title']}...\n";

    // 获取字幕
    $transcript = ($transcriber)($video['id']);
    if ('' === $transcript) {
        echo "  ⚠️ 无字幕，跳过\n";
        continue;
    }

    // 创建原始文档
    $fullDoc = new TextDocument(
        id: $video['id'],
        content: $transcript,
        metadata: new Metadata([
            'video_id' => $video['id'],
            'title' => $video['title'],
            'url' => 'https://youtube.com/watch?v=' . $video['id'],
        ]),
    );

    // 分段
    $chunks = $chunker->transform([$fullDoc]);

    // 为每个分段设置独立 ID 和元数据
    $indexDocuments = [];
    foreach ($chunks as $i => $chunk) {
        $indexDocuments[] = new TextDocument(
            id: "{$video['id']}-chunk-{$i}",
            content: $chunk->getContent(),
            metadata: new Metadata([
                'video_id' => $video['id'],
                'title' => $video['title'],
                'url' => 'https://youtube.com/watch?v=' . $video['id'],
                'chunk_index' => $i,
            ]),
        );
    }

    $indexer->index($indexDocuments);
    $chunkCount = count($indexDocuments);
    $totalChunks += $chunkCount;
    echo "  ✅ 已索引 {$chunkCount} 个段落\n";
}

echo "\n知识库构建完成：{$totalChunks} 个段落已索引\n";
```

> **💡 提示：** `ChunkTransformer` 的参数选择很重要：
> - `maxLength: 500`：每段 500 字，既保留足够上下文，又避免检索结果过长
> - `overlap: 50`：段落间重叠 50 字，确保跨段落的知识点不被截断
> - 对于技术教程，500 字通常对应 2-3 分钟的讲解内容

> **🏭 生产建议：** 批量索引时，`Indexer` 会为每个文档调用嵌入 API。100 个视频、每个 50 段 = 5000 次嵌入请求。建议：
> 1. 收集所有文档后批量传入 `$indexer->index($allDocuments)` 减少 API 调用次数
> 2. 使用 `CachePlatform` 包装嵌入平台，避免重复索引相同内容
> 3. 记录已索引的视频 ID，增量更新而非全量重建

---

## Step 4：智能视频问答（Agent + SimilaritySearch）

Agent 是 Symfony AI 的智能编排器——它接收用户问题后，自主决定是否调用 `SimilaritySearch` 工具，然后基于检索到的内容生成回答。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 创建 SimilaritySearch 工具
$searchTool = new SimilaritySearch($openaiPlatform, 'text-embedding-3-small', $store);

// Toolbox 管理所有工具
$toolbox = new Toolbox([$searchTool]);
$processor = new AgentProcessor($toolbox);

// 创建 Agent
$agent = new Agent(
    $mistralPlatform,
    'mistral-small-latest',
    inputProcessors: [
        new SystemPromptInputProcessor(
            '你是视频课程助手。用户会问关于 YouTube 技术教程的问题。'
            . "\n\n规则："
            . "\n1. 使用 similarity_search 工具从知识库中搜索相关内容"
            . "\n2. 回答时必须指出来自哪个视频（标题 + 链接）"
            . "\n3. 引用具体的字幕内容作为依据"
            . "\n4. 如果知识库中找不到答案，诚实告知"
            . "\n5. 用清晰的中文回答，适当使用 Markdown 格式"
        ),
        $processor,
    ],
    outputProcessors: [$processor],
);

// 学员提问
$result = $agent->call(new MessageBag(
    Message::ofUser('PHP 8.4 有哪些新的数组函数？'),
));

echo "🎓 回答：\n" . $result->getContent() . "\n";
```

> **📝 知识扩展：** Agent 的工具调用流程：
> 1. Agent 将用户消息 + 系统提示 + 工具列表发送给 Mistral
> 2. Mistral 决定调用 `similarity_search` 工具，返回 `ToolCall`
> 3. `AgentProcessor` 执行 `SimilaritySearch::__invoke('PHP 8.4 数组函数')`
> 4. `SimilaritySearch` 将搜索词向量化，查询 pgvector，返回最相似的文档元数据
> 5. Agent 将工具结果作为上下文，再次调用 Mistral 生成最终回答
>
> 整个过程对开发者透明——你只需定义工具和系统提示，Agent 自主完成推理和调用。

### 检查工具使用的文档

`SimilaritySearch` 会将使用过的文档保存在 `$usedDocuments` 属性中，你可以用它来展示引用来源：

```php
echo "\n📚 引用来源：\n";
foreach ($searchTool->usedDocuments as $doc) {
    $meta = $doc->getMetadata();
    echo "  - {$meta['title']}（段落 #{$meta['chunk_index']}）\n";
    echo "    🔗 {$meta['url']}\n";
}
```

---

## Step 5：多轮对话支持（Chat）

学员可以基于上一个问题继续追问。`ChatInterface` 自动管理对话历史——加载、追加、保存。

```php
<?php

use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\Bridge\InMemory\InMemoryMessageStore;

// 创建 Chat（InMemory 适合演示，生产环境用 Redis/DBAL）
$messageStore = new InMemoryMessageStore();
$chat = new Chat($agent, $messageStore);

$conversationId = 'student-session-001';

// 第一轮提问
$response1 = $chat->submit($conversationId, new MessageBag(
    Message::ofUser('Symfony 7 的路由系统有什么变化？'),
));
echo "Q1 回答：\n" . $response1->getContent() . "\n\n";

// 第二轮追问——Chat 自动带上历史消息
$response2 = $chat->submit($conversationId, new MessageBag(
    Message::ofUser('那控制器注解的写法呢？有代码示例吗？'),
));
echo "Q2 回答：\n" . $response2->getContent() . "\n";
```

> **💡 提示：** `Chat::submit()` 内部流程：
> 1. 从 `MessageStore` 加载该 `conversationId` 的历史消息
> 2. 将新的用户消息追加到历史中
> 3. 调用 `Agent::call()` 获取回复
> 4. 将回复保存到 `MessageStore`
> 5. 返回 AI 的回复
>
> 不同的 `MessageStore` 实现决定了对话的持久化方式——`InMemory` 只在进程内有效，`Redis` 支持跨请求，`DBAL` 持久化到数据库。

### 生产环境：Redis 消息存储

```php
use Symfony\AI\Chat\Bridge\Redis\RedisMessageStore;

$redis = new \Redis();
$redis->connect('127.0.0.1', 6379);
$messageStore = new RedisMessageStore($redis);
$chat = new Chat($agent, $messageStore);

// 现在对话历史会持久化到 Redis
// 学员关闭页面后再打开，仍能继续之前的对话
```

---

## Step 6：视频推荐与学习路径（StructuredOutput）

结合学员的提问历史，用 StructuredOutput 生成个性化的学习路径推荐。

```php
<?php

namespace App\Dto;

final class VideoRecommendation
{
    /**
     * @param string $videoTitle 视频标题
     * @param string $videoUrl   视频链接
     * @param string $reason     推荐理由（与学员已学内容的关联）
     * @param string $relevance  相关度（high/medium/low）
     */
    public function __construct(
        public readonly string $videoTitle,
        public readonly string $videoUrl,
        public readonly string $reason,
        public readonly string $relevance,
    ) {
    }
}

final class LearningPath
{
    /**
     * @param VideoRecommendation[] $recommendedVideos 推荐视频列表（按学习顺序排列）
     * @param string[]              $knowledgeGaps     当前知识盲区
     * @param string                $nextSteps         下一步学习建议（2-3 句话）
     */
    public function __construct(
        public readonly array $recommendedVideos,
        public readonly array $knowledgeGaps,
        public readonly string $nextSteps,
    ) {
    }
}
```

```php
<?php

use App\Dto\LearningPath;

$messages = new MessageBag(
    Message::forSystem(
        '根据学员的学习历史和提问记录，推荐最相关的视频并制定学习路径。'
        . '推荐顺序应该由浅入深，先填补基础知识盲区。'
    ),
    Message::ofUser(
        "学员最近的提问：\n"
        . "1. PHP 8.4 新特性\n"
        . "2. Symfony 路由系统\n"
        . "3. 依赖注入怎么用\n\n"
        . "可用视频：\n"
        . "- PHP 8.4 新特性详解 (https://youtube.com/watch?v=abc123)\n"
        . "- Symfony 7 快速上手 (https://youtube.com/watch?v=def456)\n"
        . "- AI Agent 开发实战 (https://youtube.com/watch?v=ghi789)\n"
        . "- PHP 设计模式 (https://youtube.com/watch?v=jkl012)\n"
        . "- 数据库优化实战 (https://youtube.com/watch?v=mno345)\n\n"
        . '请推荐学习路径。'
    ),
);

$result = $mistralPlatform->invoke('mistral-small-latest', $messages, [
    'response_format' => LearningPath::class,
]);

/** @var LearningPath $path */
$path = $result->asObject();

echo "=== 学习路径推荐 ===\n\n";
foreach ($path->recommendedVideos as $i => $rec) {
    echo ($i + 1) . ". [{$rec->relevance}] {$rec->videoTitle}\n";
    echo "   🔗 {$rec->videoUrl}\n";
    echo "   💡 {$rec->reason}\n\n";
}

echo "📚 知识盲区：\n";
foreach ($path->knowledgeGaps as $gap) {
    echo "  - {$gap}\n";
}
echo "\n🎯 {$path->nextSteps}\n";
```

---

## 完整示例

```php
<?php

require 'vendor/autoload.php';

use App\Dto\LearningPath;
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch;
use Symfony\AI\Agent\Bridge\Youtube\YoutubeTranscriber;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Chat\Bridge\InMemory\InMemoryMessageStore;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Platform\Bridge\Mistral\PlatformFactory as MistralPlatformFactory;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory as OpenAiPlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\AI\Store\Bridge\Postgres\Store;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Transformer\ChunkTransformer;
use Symfony\AI\Store\Indexer;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

// === 初始化 ===

$httpClient = HttpClient::create();
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

$mistral = MistralPlatformFactory::create($_ENV['MISTRAL_API_KEY'], $httpClient, eventDispatcher: $dispatcher);
$openai = OpenAiPlatformFactory::create($_ENV['OPENAI_API_KEY'], $httpClient);

$pdo = new \PDO('pgsql:host=localhost;dbname=video_kb', 'postgres', 'password');
$store = new Store($pdo, 'video_chunks');
$store->setup(['vector_size' => 1536]);

$transcriber = new YoutubeTranscriber($httpClient);
$chunker = new ChunkTransformer(maxLength: 500, overlap: 50);
$indexer = new Indexer($openai, 'text-embedding-3-small', $store);

// === 1. 知识库构建 ===

$videos = [
    ['id' => 'abc123', 'title' => 'PHP 8.4 新特性详解'],
    ['id' => 'def456', 'title' => 'Symfony 7 快速上手'],
];

foreach ($videos as $video) {
    $transcript = ($transcriber)($video['id']);
    if ('' === $transcript) {
        continue;
    }

    $fullDoc = new TextDocument($video['id'], $transcript, new Metadata([
        'video_id' => $video['id'],
        'title' => $video['title'],
        'url' => 'https://youtube.com/watch?v=' . $video['id'],
    ]));

    $chunks = $chunker->transform([$fullDoc]);
    $docs = [];
    foreach ($chunks as $i => $chunk) {
        $docs[] = new TextDocument(
            "{$video['id']}-{$i}",
            $chunk->getContent(),
            new Metadata(['video_id' => $video['id'], 'title' => $video['title'], 'url' => 'https://youtube.com/watch?v=' . $video['id'], 'chunk_index' => $i]),
        );
    }
    $indexer->index($docs);
    echo "✅ 索引 {$video['title']}：" . count($docs) . " 段\n";
}

// === 2. 智能问答 ===

$searchTool = new SimilaritySearch($openai, 'text-embedding-3-small', $store);
$toolbox = new Toolbox([$searchTool]);
$processor = new AgentProcessor($toolbox);

$agent = new Agent(
    $mistral, 'mistral-small-latest',
    [
        new SystemPromptInputProcessor(
            '你是视频课程助手。用 similarity_search 搜索知识库回答问题，必须引用视频来源。'
        ),
        $processor,
    ],
    [$processor],
);

// === 3. 多轮对话 ===

$chat = new Chat($agent, new InMemoryMessageStore());

$answer1 = $chat->submit('session-1', new MessageBag(Message::ofUser('PHP 8.4 有什么新特性？')));
echo "\nQ1：" . $answer1->getContent() . "\n";

$answer2 = $chat->submit('session-1', new MessageBag(Message::ofUser('那 Fibers 在 8.4 中有更新吗？')));
echo "\nQ2：" . $answer2->getContent() . "\n";

// === 4. 学习推荐 ===

$path = $mistral->invoke('mistral-small-latest', new MessageBag(
    Message::forSystem('根据学习历史推荐视频和学习路径。'),
    Message::ofUser("提问历史：PHP 8.4 新特性、Fibers\n可用视频：PHP 8.4 详解、Symfony 7 上手、AI Agent 实战"),
), ['response_format' => LearningPath::class])->asObject();

echo "\n=== 推荐 ===\n";
foreach ($path->recommendedVideos as $rec) {
    echo "- [{$rec->relevance}] {$rec->videoTitle}：{$rec->reason}\n";
}
```

---

## 替代实现方案

### 方案 A：Anthropic Claude 高质量问答

对于需要更高回答质量（如付费课程客服）的场景，使用 Anthropic Claude：

```php
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory as AnthropicPlatformFactory;

$anthropic = AnthropicPlatformFactory::create(
    $_ENV['ANTHROPIC_API_KEY'],
    $httpClient,
    eventDispatcher: $dispatcher,
);

$agent = new Agent(
    $anthropic,
    'claude-sonnet',
    // ... 工具和处理器配置不变
);
```

> **💡 提示：** Claude 的上下文窗口更大（200K tokens），适合需要引用大量字幕内容的深度问答。Mistral 更适合快速简洁的问答场景。

### 方案 B：Ollama 本地部署（离线使用）

对于企业内训场景（无法访问外网），可以用 Ollama 在本地运行：

```php
use Symfony\AI\Platform\Bridge\Ollama\PlatformFactory as OllamaPlatformFactory;

$ollama = OllamaPlatformFactory::create(
    model: 'llama3.1:8b',
    url: 'http://localhost:11434',
    httpClient: $httpClient,
    eventDispatcher: $dispatcher,
);

// 嵌入模型也用 Ollama 的本地嵌入
$ollamaEmbedding = OllamaPlatformFactory::create(
    model: 'nomic-embed-text',
    url: 'http://localhost:11434',
    httpClient: $httpClient,
);
```

### 方案 C：CachePlatform 缓存热门问题

视频知识库中很多问题是重复的（"怎么安装 Symfony？"），使用缓存大幅提升响应速度和降低成本：

```php
use Symfony\AI\Platform\Bridge\Cache\CachePlatform;
use Symfony\Component\Cache\Adapter\RedisAdapter;

$cachedMistral = new CachePlatform(
    $mistral,
    cache: RedisAdapter::createConnection('redis://localhost'),
);

// Agent 使用缓存后的平台——热门问题直接返回缓存答案
$agent = new Agent($cachedMistral, 'mistral-small-latest', ...);
```

> **🏭 生产建议：** 视频知识库场景下缓存命中率通常很高（>60%），因为学员提问集中在核心知识点上。建议将缓存 TTL 设为视频更新周期（如 7 天），视频更新后清除相关缓存。

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `YoutubeTranscriber` | Agent Bridge 工具，`#[AsTool]` 标注，获取 YouTube 视频字幕 |
| `ChunkTransformer` | Store 文档转换器，按长度分段并支持重叠，保留原始元数据 |
| `Indexer` | 索引管线：文档 → 嵌入（OpenAI）→ 存储（pgvector） |
| `SimilaritySearch` | Agent 工具，`#[AsTool]` 标注，从向量库中语义检索并返回元数据 |
| `Agent` + `Toolbox` | 智能编排：Agent 自主决定何时调用工具，基于结果生成回答 |
| `AgentProcessor` | 双重角色——InputProcessor（注册工具列表）+ OutputProcessor（执行工具调用） |
| `Chat` + `MessageStore` | 对话管理器——自动加载/保存消息历史，支持多轮追问 |
| pgvector Store | PostgreSQL 向量存储，适合已有 PG 基础设施的团队 |
| `LearningPath` 嵌套 DTO | StructuredOutput 递归 Schema，实现类型安全的推荐结果 |
| 事件系统 Token 统计 | `ResultEvent` 监听器按平台统计成本 |
| `CachePlatform` | 缓存包装器，对热门问答显著降低成本和延迟 |

## 下一步

如果你需要自动抓取和分析竞品网站信息，请看 [27-competitor-intelligence-monitoring.md](./27-competitor-intelligence-monitoring.md)。
