# YouTube 视频知识库

## 业务场景

你的团队通过 YouTube 频道做技术教育，频道已积累上百个视频教程，每个 20–60 分钟不等。学员经常问："哪个视频讲了 XXX？""某某知识点在第几分钟？"但人工回忆和搜索效率极低。现在你要构建一个 **智能视频知识库系统**：自动获取 YouTube 视频字幕，对字幕内容分段并向量化索引，学员可以用自然语言提问，AI 从知识库中精准定位到具体视频和内容段落，并支持多轮追问和个性化学习推荐。

**典型应用：**

- 在线课程平台——学员按知识点检索视频内容
- 企业会议录像知识库——快速回溯历史会议要点
- 播客/技术演讲内容检索——从海量音视频中提取关键信息
- 培训资料智能问答——新员工入职自助学习
- 技术文档视频化——将视频教程转化为可搜索的文本知识库
- 内容创作者素材管理——跨视频检索和复用素材

## 涉及模块

| 模块 | Composer 包 | 用途 |
|------|------------|------|
| **Platform（核心）** | `symfony/ai-platform` | AI 平台统一接口、消息系统、StructuredOutput、事件调度 |
| **Mistral Bridge** | `symfony/ai-mistral-platform` | 连接 Mistral 大模型——主 LLM 引擎，用于问答生成和推理 |
| **OpenAI Bridge** | `symfony/ai-open-ai-platform` | 连接 OpenAI——用于 `text-embedding-3-small` 文本嵌入向量化 |
| **Cache Bridge** | `symfony/ai-cache-platform` | `CachePlatform` 装饰器——缓存 LLM 调用结果，降低重复请求成本 |
| **YouTube Bridge** | `symfony/ai-youtube-tool` | `YoutubeTranscriber`——自动获取 YouTube 视频字幕文本，同时也是 Agent 工具 |
| **Store（核心）** | `symfony/ai-store` | 文档模型（`TextDocument`）、`Vectorizer`、`TextSplitTransformer`、`DocumentProcessor` |
| **PostgreSQL Store** | `symfony/ai-postgres-store` | 基于 pgvector 的向量存储——支持向量检索、文本检索和混合检索 |
| **Agent** | `symfony/ai-agent` | Agent 框架——`InputProcessor` / `OutputProcessor` / `Toolbox` 管线编排 |
| **SimilaritySearch** | `symfony/ai-similarity-search-tool` | Agent 工具——从向量库中执行语义相似度检索 |
| **Chat** | `symfony/ai-chat` | 对话持久化——支持多轮追问，保持上下文连贯 |

## 核心概念

在开始编码前，简要了解本教程涉及的几个关键概念：

- **YoutubeTranscriber**：同时是一个独立工具和 `#[AsTool]` 标注的 Agent 工具。它通过 `mrmysql/youtube-transcript` 包获取 YouTube 视频的字幕文本（transcript）。接受视频 ID 作为参数，返回纯文本字幕。
- **TextSplitTransformer**：将长文本按指定大小分段（chunking），支持重叠（overlap）以保证上下文连贯性。分段是 RAG 流水线中至关重要的一步——太大导致检索不精准，太小导致丢失上下文。
- **DocumentProcessor**：处理管线——按 "过滤 → 转换 → 向量化 → 存储" 的顺序处理文档，是 `DocumentIndexer` 的内部引擎。
- **Vectorizer**：将文本或文档转换为向量表示。支持批量向量化（当模型支持 `INPUT_MULTIPLE` 时自动使用）。
- **SimilaritySearch**：`#[AsTool('similarity_search')]` 标注的 Agent 工具，让大模型在推理过程中自主决定何时从向量库中检索相关文档。
- **CachePlatform**：装饰器模式——包装任意 `PlatformInterface`，对相同输入缓存结果。在视频知识库场景中，同一问题的重复查询可显著降低 API 成本。

## 项目实现思路

整个系统分为三层：

1. **数据采集层**：通过 `YoutubeTranscriber` 批量获取视频字幕，处理错误和重试
2. **索引构建层**：字幕分段（`TextSplitTransformer`）→ 向量化（`Vectorizer` + OpenAI Embedding）→ 存入 PostgreSQL（pgvector）
3. **智能问答层**：Agent + `SimilaritySearch` 工具实现语义检索，结合 Chat 持久化支持多轮追问，通过 StructuredOutput 生成学习推荐

## 项目流程图

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────────┐     ┌──────────────────┐
│  YouTube 视频列表 │ ──▶ │ YoutubeTranscriber│ ──▶ │  TextSplitTransformer │ ──▶ │  DocumentProcessor │
│ (视频 ID + 元数据)│     │   (获取字幕文本)   │     │  (分段 500 字/段)     │     │  (向量化 + 存储)   │
└──────────────────┘     └──────────────────┘     └──────────────────────┘     └────────┬─────────┘
                                                                                        │
                                                                                        ▼
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────────┐     ┌──────────────────┐
│  显示回答 + 来源  │ ◀── │  Mistral LLM 推理 │ ◀── │  SimilaritySearch    │ ◀── │  PostgreSQL 向量库 │
│ (视频标题 + 链接) │     │  (生成自然语言答案)│     │  (语义检索相关段落)   │     │  (pgvector 索引)   │
└──────────────────┘     └──────────────────┘     └──────────────────────┘     └──────────────────┘
        │                                                                              ▲
        │           ┌──────────────────┐     ┌──────────────────────┐                  │
        └─────────▶ │  Chat 持久化     │ ──▶ │  DoctrineDbalStore   │   多轮追问时加载历史
                    │  (保存对话上下文) │     │  (PostgreSQL 存储)   │   ─────────────────┘
                    └──────────────────┘     └──────────────────────┘
```

> **📝 知识扩展：** `YoutubeTranscriber` 使用 `__invoke(string $videoId)` 方法，参数是视频 ID（不是完整 URL）。它同时标注了 `#[AsTool('youtube_transcript', 'Fetches the transcript of a YouTube video')]`，因此既可以作为独立工具手动调用，也可以注册到 Agent 的 Toolbox 中让大模型自主决定何时获取字幕。

---

## 前置准备

### 环境要求

- PHP >= 8.2
- Composer
- PostgreSQL 15+（启用 pgvector 扩展）
- Mistral API 密钥（用于 LLM 推理）
- OpenAI API 密钥（用于文本嵌入向量化）

### 安装依赖

```bash
composer require symfony/ai-platform symfony/ai-mistral-platform \
    symfony/ai-open-ai-platform symfony/ai-cache-platform \
    symfony/ai-agent symfony/ai-youtube-tool \
    symfony/ai-similarity-search-tool \
    symfony/ai-store symfony/ai-postgres-store \
    symfony/ai-chat \
    symfony/event-dispatcher symfony/cache
```

> **📝 知识扩展：** `symfony/ai-youtube-tool` 内部依赖 `mrmysql/youtube-transcript` 包来解析 YouTube 字幕。如果安装时缺少该依赖，`YoutubeTranscriber` 构造函数会抛出 `LogicException` 并给出安装提示。

### 启动 PostgreSQL（pgvector）

```bash
docker run -d --name pgvector \
    -p 5432:5432 \
    -e POSTGRES_USER=app \
    -e POSTGRES_PASSWORD=secret \
    -e POSTGRES_DB=video_knowledge \
    pgvector/pgvector:pg16
```

### 设置 API 密钥

```bash
export MISTRAL_API_KEY="your-mistral-api-key-here"
export OPENAI_API_KEY="sk-your-openai-api-key-here"
```

> **🔒 安全建议：** 永远不要将 API 密钥硬编码在源代码中。在 Symfony 项目中，应使用 `.env.local` 文件或服务器环境变量来管理密钥；在 CI/CD 中使用 Secrets 管理。API 密钥一旦泄露会被自动扫描程序利用，导致意外费用。

---

## Step 1：创建 Platform 实例

本项目需要两个 Platform 实例：Mistral 用于 LLM 推理（问答生成），OpenAI 用于文本嵌入向量化。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\Mistral\PlatformFactory;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory as OpenAiPlatformFactory;

// Mistral 平台——用于 LLM 推理（问答、推荐等）
$platform = PlatformFactory::create($_ENV['MISTRAL_API_KEY']);

// OpenAI 平台——专门用于文本嵌入向量化
$embeddingPlatform = OpenAiPlatformFactory::create($_ENV['OPENAI_API_KEY']);
```

Mistral 的 `PlatformFactory::create()` 完整签名如下：

```php
public static function create(
    string $apiKey,                          // 必需：API 密钥
    ?HttpClientInterface $httpClient = null,  // 可选：自定义 HTTP 客户端
    ModelCatalogInterface $modelCatalog = new ModelCatalog(),  // 可选：模型目录
    ?Contract $contract = null,              // 可选：自定义序列化契约
    ?EventDispatcherInterface $eventDispatcher = null,  // 可选：事件调度器
): Platform
```

> **💡 设计说明：** 为什么使用两个不同平台？Mistral 在推理性价比上有优势，而 OpenAI 的 `text-embedding-3-small` 在嵌入质量和生态兼容性上表现稳定。Symfony AI 的 Bridge 架构让你可以自由组合不同平台的优势。

---

## Step 2：获取 YouTube 视频字幕

使用 `YoutubeTranscriber` 自动获取 YouTube 视频字幕。它接受 **视频 ID**（而非完整 URL）作为参数。

```php
<?php

use Symfony\AI\Agent\Bridge\Youtube\YoutubeTranscriber;
use Symfony\Component\HttpClient\HttpClient;

$transcriber = new YoutubeTranscriber(HttpClient::create());

// 注意：参数是视频 ID，不是完整 URL
$videoId = 'dQw4w9WgXcQ';

try {
    $transcript = ($transcriber)($videoId);
    echo "字幕内容（前 300 字）：\n";
    echo mb_substr($transcript, 0, 300)."...\n";
    echo "字幕总长度：".mb_strlen($transcript)." 字\n";
} catch (\Throwable $e) {
    echo "获取字幕失败：{$e->getMessage()}\n";
    // 常见原因：视频无字幕、字幕被禁用、视频不存在、网络超时
}
```

> **⚠️ 注意事项：** `YoutubeTranscriber::__invoke()` 的参数是视频 ID（如 `dQw4w9WgXcQ`），不是完整的 YouTube URL。如果你从 URL 中提取 ID，需要自行解析 `watch?v=` 参数。

下面封装一个批量获取字幕的辅助函数，包含错误处理和重试逻辑：

```php
<?php

/**
 * 批量获取视频字幕，带错误处理。
 *
 * @param array<array{id: string, title: string}> $videos 视频列表
 *
 * @return array<string, string> 视频 ID => 字幕文本
 */
function fetchTranscripts(YoutubeTranscriber $transcriber, array $videos): array
{
    $transcripts = [];
    $failedVideos = [];

    foreach ($videos as $video) {
        echo "📹 获取字幕：{$video['title']}（{$video['id']}）...\n";

        try {
            $transcript = ($transcriber)($video['id']);

            if ('' === trim($transcript)) {
                echo "  ⚠️ 字幕为空，跳过\n";
                $failedVideos[] = $video['id'];
                continue;
            }

            $transcripts[$video['id']] = $transcript;
            echo "  ✅ 成功，".mb_strlen($transcript)." 字\n";
        } catch (\Throwable $e) {
            echo "  ❌ 失败：{$e->getMessage()}\n";
            $failedVideos[] = $video['id'];
        }

        // 请求间隔，避免频率限制
        usleep(500_000);
    }

    if ([] !== $failedVideos) {
        echo "\n⚠️ 以下视频获取字幕失败：".implode(', ', $failedVideos)."\n";
    }

    return $transcripts;
}
```

> **🔒 版权与隐私警告：** YouTube 视频字幕属于内容创作者的知识产权。将字幕用于内部知识库索引通常属于合理使用，但请注意：1) 不要将字幕内容原文对外公开发布；2) 遵守 YouTube 服务条款；3) 如果用于商业用途，请确保获得内容创作者的许可。

---

## Step 3：配置向量存储（PostgreSQL + pgvector）

使用 PostgreSQL 的 pgvector 扩展作为向量数据库。它支持向量检索（`VectorQuery`）、文本检索（`TextQuery`）和混合检索（`HybridQuery`）。

```php
<?php

use Symfony\AI\Store\Bridge\Postgres\Distance;
use Symfony\AI\Store\Bridge\Postgres\Store;

// 创建 PDO 连接
$pdo = new \PDO(
    'pgsql:host=localhost;port=5432;dbname=video_knowledge',
    'app',
    'secret',
);

// 创建向量存储
$store = new Store(
    connection: $pdo,
    tableName: 'video_transcripts',
    vectorFieldName: 'embedding',
    distance: Distance::COSINE,  // 余弦相似度——适合文本语义检索
);

// 初始化数据库表结构（首次运行时执行）
// OpenAI text-embedding-3-small 的向量维度是 1536
$store->setup(['vector_size' => 1536]);

echo "✅ 向量存储初始化完成\n";
```

> **📝 知识扩展：** `Distance::COSINE` 是文本语义检索的推荐距离度量。`Store::setup()` 会自动创建 pgvector 扩展、数据表和索引。`vector_size` 参数必须与嵌入模型的输出维度匹配——`text-embedding-3-small` 是 1536 维。

---

## Step 4：字幕分段与向量化索引

这是 RAG 流水线的核心——将长字幕文本分段，然后通过 `DocumentProcessor` 自动完成 "转换 → 向量化 → 存储" 的全流程。

```php
<?php

use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentIndexer;
use Symfony\AI\Store\Indexer\DocumentProcessor;

// 创建 Vectorizer——使用 OpenAI 嵌入平台
$vectorizer = new Vectorizer($embeddingPlatform, 'text-embedding-3-small');

// 创建文本分段器——每 500 字一段，重叠 100 字
$splitter = new TextSplitTransformer(chunkSize: 500, overlap: 100);

// 组装文档处理管线：分段 → 向量化 → 存储
$processor = new DocumentProcessor(
    vectorizer: $vectorizer,
    store: $store,
    transformers: [$splitter],
);

$indexer = new DocumentIndexer($processor);

// 视频列表
$videos = [
    ['id' => 'abc123', 'title' => 'PHP 8.4 新特性详解'],
    ['id' => 'def456', 'title' => 'Symfony 7 快速上手'],
    ['id' => 'ghi789', 'title' => 'AI Agent 开发实战'],
];

// 获取字幕（使用 Step 2 的方法）
$transcripts = fetchTranscripts($transcriber, $videos);

// 为每个视频创建文档并索引
$documents = [];
foreach ($videos as $video) {
    if (!isset($transcripts[$video['id']])) {
        continue; // 跳过获取字幕失败的视频
    }

    $documents[] = new TextDocument(
        id: $video['id'],
        content: $transcripts[$video['id']],
        metadata: new Metadata([
            'video_id' => $video['id'],
            'title' => $video['title'],
            'url' => 'https://youtube.com/watch?v='.$video['id'],
            'source' => 'youtube',
        ]),
    );
}

echo "开始索引 ".count($documents)." 个视频...\n";
$indexer->index($documents);
echo "✅ 索引完成！\n";
```

> **💡 分段策略提示：** `chunkSize: 500` 和 `overlap: 100` 是视频字幕的推荐配置。视频字幕通常是连续的口语文本，重叠 100 字可以保证跨段内容的上下文连贯性。如果视频内容结构性更强（如编程教程），可以适当增大 `chunkSize` 到 800–1000。

> **📝 知识扩展：** `TextSplitTransformer` 在分段时会自动为每个分段设置 `Metadata::KEY_PARENT_ID`（指向原始文档 ID）和 `Metadata::KEY_TEXT`（分段文本内容）。原始文档的所有 Metadata 也会被继承到每个分段中，因此你可以在检索结果中直接获取 `video_id`、`title` 等信息。

---

## Step 5：构建智能视频问答 Agent

使用 Agent + SimilaritySearch 工具，让大模型自主决定何时从知识库中检索内容。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 创建语义检索工具
$searchTool = new SimilaritySearch($vectorizer, $store);

// 创建 Toolbox 并注册检索工具
$toolbox = new Toolbox([$searchTool]);
$agentProcessor = new AgentProcessor($toolbox);

// 系统提示词——定义 Agent 的行为规范
$systemPrompt = <<<PROMPT
你是一个专业的视频课程助手。你的任务是回答学员关于视频教程内容的问题。

## 工作流程
1. 当学员提问时，使用 similarity_search 工具从视频字幕知识库中搜索相关内容
2. 根据搜索结果，组织自然语言回答
3. 回答时必须包含：
   - 具体的知识点解释
   - 来源视频的标题
   - 视频链接（从元数据中提取 url 字段）

## 注意事项
- 如果知识库中找不到相关内容，诚实告知学员
- 不要编造视频中没有提到的内容
- 如果多个视频涉及同一主题，综合多个来源回答
PROMPT;

// 创建 Agent
$agent = new Agent(
    platform: $platform,
    model: 'mistral-large-latest',
    inputProcessors: [
        new SystemPromptInputProcessor($systemPrompt),
        $agentProcessor,
    ],
    outputProcessors: [$agentProcessor],
);

// 学员提问
$result = $agent->call(new MessageBag(
    Message::ofUser('PHP 8.4 有哪些新的数组函数？请详细介绍。'),
));

echo "🎓 回答：\n".$result->getContent()."\n";

// 检查检索到的文档（用于调试）
if ([] !== $searchTool->usedDocuments) {
    echo "\n📚 参考来源：\n";
    foreach ($searchTool->usedDocuments as $doc) {
        $meta = $doc->getMetadata();
        echo "  - {$meta['title']} ({$meta['url']})\n";
    }
}
```

> **💡 调试提示：** `SimilaritySearch` 的 `usedDocuments` 属性是公开的，记录了最近一次检索使用的文档列表。你可以用它来验证检索质量——如果返回的文档与问题不相关，考虑调整分段大小或嵌入模型。

---

## Step 6：多轮对话与 Chat 持久化

使用 Chat 组件保存对话历史，让学员可以基于上一个问题继续追问。

```php
<?php

use Doctrine\DBAL\DriverManager;
use Symfony\AI\Chat\Bridge\Doctrine\DoctrineDbalMessageStore;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 使用 Doctrine DBAL 持久化对话
$dbalConnection = DriverManager::getConnection([
    'url' => 'pgsql://app:secret@localhost:5432/video_knowledge',
]);

$messageStore = new DoctrineDbalMessageStore(
    tableName: 'chat_messages',
    dbalConnection: $dbalConnection,
);

// 首次运行时初始化表结构
$messageStore->setup();

// 会话 ID——每个学员一个独立会话
$conversationId = 'student-session-'.uniqid();

// === 第一轮提问 ===
$messages = new MessageBag(
    Message::ofUser('Symfony 7 的路由系统有什么变化？'),
);

$result = $agent->call($messages);

// 保存完整对话历史（包括 AI 回复）
$fullMessages = $result->getMessages();
$messageStore->save($fullMessages);
echo "Q1 回答：\n".$result->getContent()."\n\n";

// === 第二轮追问 ===
// 加载历史消息并追加新问题
$messages = $result->getMessages();
$messages->add(Message::ofUser('那控制器属性的写法呢？有代码示例吗？'));

$result = $agent->call($messages);
$messageStore->save($result->getMessages());
echo "Q2 回答：\n".$result->getContent()."\n";
```

> **📝 知识扩展：** `DoctrineDbalMessageStore` 基于 Doctrine DBAL 实现对话持久化。它序列化 `MessageBag` 为 JSON 存储在 PostgreSQL 中。调用 `setup()` 会自动创建所需的表结构（如果不存在的话）。

> **🏭 生产建议：** 在生产环境中，建议为会话 ID 绑定用户身份（如用户 ID + 时间戳），并设置会话过期清理策略。长对话历史会增加 Token 消耗——考虑在一定轮次后进行上下文摘要压缩。

---

## Step 7：视频推荐与学习路径（StructuredOutput）

结合学员的提问历史，使用 StructuredOutput 让大模型返回结构化的学习推荐。

首先定义 DTO 类：

```php
<?php

namespace App\Dto;

final class VideoRecommendation
{
    /**
     * @param string $videoTitle 视频标题
     * @param string $videoUrl   视频链接
     * @param string $reason     推荐理由
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
     * @param VideoRecommendation[] $recommendedVideos 推荐视频列表
     * @param string[]              $knowledgeGaps     知识盲区
     * @param string                $nextSteps         下一步学习建议
     */
    public function __construct(
        public readonly array $recommendedVideos,
        public readonly array $knowledgeGaps,
        public readonly string $nextSteps,
    ) {
    }
}
```

然后使用 StructuredOutput 生成推荐：

```php
<?php

use App\Dto\LearningPath;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

// 启用 StructuredOutput 支持
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

// 创建支持 StructuredOutput 的 Mistral 平台实例
$structuredPlatform = \Symfony\AI\Platform\Bridge\Mistral\PlatformFactory::create(
    $_ENV['MISTRAL_API_KEY'],
    eventDispatcher: $dispatcher,
);

$messages = new MessageBag(
    Message::forSystem(
        '根据学员的学习历史和提问记录，推荐相关视频并制定学习路径。'
        .' 请按照给定的 JSON 格式返回结果。'
    ),
    Message::ofUser(
        "学员最近的提问：\n"
        ."1. PHP 8.4 新特性\n"
        ."2. Symfony 路由系统\n"
        ."3. 依赖注入怎么用\n\n"
        ."可用视频库：\n"
        ."- PHP 8.4 新特性详解 (https://youtube.com/watch?v=abc123)\n"
        ."- Symfony 7 快速上手 (https://youtube.com/watch?v=def456)\n"
        ."- AI Agent 开发实战 (https://youtube.com/watch?v=ghi789)\n"
        ."- PHP 设计模式 (https://youtube.com/watch?v=jkl012)\n"
        ."- 数据库优化实战 (https://youtube.com/watch?v=mno345)\n\n"
        .'请推荐学习路径。'
    ),
);

$result = $structuredPlatform->invoke('mistral-large-latest', $messages, [
    'response_format' => LearningPath::class,
]);

$path = $result->asObject();

echo "=== 🎯 个性化学习路径 ===\n\n";
echo "📺 推荐视频：\n";
foreach ($path->recommendedVideos as $i => $rec) {
    echo ($i + 1).". [{$rec->relevance}] {$rec->videoTitle}\n";
    echo "   🔗 {$rec->videoUrl}\n";
    echo "   💡 {$rec->reason}\n\n";
}

echo "📚 知识盲区：\n";
foreach ($path->knowledgeGaps as $gap) {
    echo "  - {$gap}\n";
}
echo "\n🎯 下一步：{$path->nextSteps}\n";
```

> **⚠️ 注意事项：** StructuredOutput 需要通过 `EventDispatcher` 注册 `PlatformSubscriber`。如果忘记注册，`asObject()` 方法会失败。在 Symfony Bundle 环境中，`symfony/ai-bundle` 会自动完成这一注册。

---

## Step 8：生产环境优化——CachePlatform 与事件监控

在生产环境中，使用 `CachePlatform` 缓存 LLM 调用结果，并通过事件系统监控 Token 消耗。

```php
<?php

use Symfony\AI\Platform\Bridge\Cache\CachePlatform;
use Symfony\AI\Platform\Bridge\Mistral\PlatformFactory;
use Symfony\Component\Cache\Adapter\TagAwareAdapter;
use Symfony\Component\Cache\Adapter\FilesystemAdapter;

// 创建缓存适配器（生产环境建议使用 Redis）
$cacheAdapter = new TagAwareAdapter(new FilesystemAdapter(
    namespace: 'ai_video_kb',
    defaultLifetime: 3600,  // 缓存 1 小时
));

// 创建 Mistral 平台
$basePlatform = PlatformFactory::create($_ENV['MISTRAL_API_KEY']);

// 用 CachePlatform 包装——相同输入直接返回缓存结果
$cachedPlatform = new CachePlatform(
    platform: $basePlatform,
    cache: $cacheAdapter,
    cacheTtl: 3600,
);

// 现在使用 $cachedPlatform 替代 $platform
// 相同问题的重复查询将直接返回缓存结果，不会调用 API
```

添加事件监听器来追踪 Token 使用量：

```php
<?php

use Symfony\AI\Platform\Event\InvokeEvent;
use Symfony\Component\EventDispatcher\EventDispatcher;

$dispatcher = new EventDispatcher();

// 监听 LLM 调用事件
$dispatcher->addListener(InvokeEvent::class, function (InvokeEvent $event): void {
    $result = $event->result;
    $metadata = $result->getMetadata();
    echo \sprintf(
        "[监控] 模型: %s | Token 输入: %s | Token 输出: %s\n",
        $metadata->get('model') ?? 'unknown',
        $metadata->get('input_tokens') ?? '?',
        $metadata->get('output_tokens') ?? '?',
    );
});

// 创建带事件监控的平台
$monitoredPlatform = PlatformFactory::create(
    $_ENV['MISTRAL_API_KEY'],
    eventDispatcher: $dispatcher,
);
```

> **🏭 生产建议：** 在视频知识库场景中，`CachePlatform` 效果显著——学员的热门问题往往集中在少数主题上。结合 Redis 缓存和 1 小时 TTL，可以减少 60–80% 的 API 调用。同时建议记录 Token 使用日志，以便追踪成本和优化提示词。

---

## Step 9：完整 CLI 示例——视频知识库问答系统

将前面所有步骤整合为一个完整的命令行交互程序：

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch;
use Symfony\AI\Agent\Bridge\Youtube\YoutubeTranscriber;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\Mistral\PlatformFactory;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory as OpenAiPlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Store\Bridge\Postgres\Distance;
use Symfony\AI\Store\Bridge\Postgres\Store;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentIndexer;
use Symfony\AI\Store\Indexer\DocumentProcessor;
use Symfony\Component\HttpClient\HttpClient;

// ========================================
// 1. 初始化所有组件
// ========================================

$platform = PlatformFactory::create($_ENV['MISTRAL_API_KEY']);
$embeddingPlatform = OpenAiPlatformFactory::create($_ENV['OPENAI_API_KEY']);

$pdo = new \PDO('pgsql:host=localhost;port=5432;dbname=video_knowledge', 'app', 'secret');
$store = new Store($pdo, 'video_transcripts', 'embedding', Distance::COSINE);
$store->setup(['vector_size' => 1536]);

$vectorizer = new Vectorizer($embeddingPlatform, 'text-embedding-3-small');
$transcriber = new YoutubeTranscriber(HttpClient::create());

// ========================================
// 2. 索引视频字幕（首次运行时执行）
// ========================================

$videos = [
    ['id' => 'abc123', 'title' => 'PHP 8.4 新特性详解'],
    ['id' => 'def456', 'title' => 'Symfony 7 快速上手'],
    ['id' => 'ghi789', 'title' => 'AI Agent 开发实战'],
];

echo "=== 📹 开始索引视频字幕 ===\n\n";

$documents = [];
foreach ($videos as $video) {
    try {
        echo "获取字幕：{$video['title']}... ";
        $transcript = ($transcriber)($video['id']);
        $documents[] = new TextDocument(
            id: $video['id'],
            content: $transcript,
            metadata: new Metadata([
                'video_id' => $video['id'],
                'title' => $video['title'],
                'url' => 'https://youtube.com/watch?v='.$video['id'],
            ]),
        );
        echo "✅\n";
    } catch (\Throwable $e) {
        echo "❌ {$e->getMessage()}\n";
    }
}

if ([] !== $documents) {
    $processor = new DocumentProcessor(
        vectorizer: $vectorizer,
        store: $store,
        transformers: [new TextSplitTransformer(chunkSize: 500, overlap: 100)],
    );
    $indexer = new DocumentIndexer($processor);
    $indexer->index($documents);
    echo "\n✅ 已索引 ".count($documents)." 个视频\n";
}

// ========================================
// 3. 启动交互问答
// ========================================

$searchTool = new SimilaritySearch($vectorizer, $store);
$toolbox = new Toolbox([$searchTool]);
$agentProcessor = new AgentProcessor($toolbox);

$agent = new Agent(
    platform: $platform,
    model: 'mistral-large-latest',
    inputProcessors: [
        new SystemPromptInputProcessor(
            '你是视频课程助手。使用 similarity_search 工具从知识库中检索内容，'
            .'回答时指出来源视频标题和链接。找不到则诚实告知。'
        ),
        $agentProcessor,
    ],
    outputProcessors: [$agentProcessor],
);

echo "\n=== 🎓 视频知识库问答系统 ===\n";
echo "输入问题（输入 'exit' 退出）：\n\n";

// 保持对话上下文
$messages = new MessageBag();

while (true) {
    echo '> ';
    $input = trim(fgets(\STDIN));

    if ('exit' === $input || '' === $input) {
        echo "再见！\n";
        break;
    }

    $messages->add(Message::ofUser($input));

    try {
        $result = $agent->call($messages);
        $messages = $result->getMessages();

        echo "\n🤖 ".$result->getContent()."\n\n";

        // 显示参考来源
        if ([] !== $searchTool->usedDocuments) {
            echo "📚 参考来源：\n";
            foreach ($searchTool->usedDocuments as $doc) {
                $meta = $doc->getMetadata();
                echo "  - ".($meta['title'] ?? '未知')." (".($meta['url'] ?? '').") \n";
            }
            echo "\n";
        }
    } catch (\Throwable $e) {
        echo "\n❌ 错误：{$e->getMessage()}\n\n";
    }
}
```

> **💡 运行方式：** 将上述代码保存为 `video-kb.php`，执行 `php video-kb.php` 即可启动交互式问答。首次运行会索引字幕（耗时较长），之后可以注释掉索引部分直接进入问答。

---

## Step 10：替代实现——使用 DeepSeek 作为 LLM

如果你更偏好 DeepSeek，只需更换 Platform Bridge 即可——业务代码完全不需要改动：

```php
<?php

use Symfony\AI\Platform\Bridge\DeepSeek\PlatformFactory as DeepSeekPlatformFactory;

// 替换 Mistral 为 DeepSeek
$platform = DeepSeekPlatformFactory::create($_ENV['DEEPSEEK_API_KEY']);

// 创建 Agent 时使用 DeepSeek 的模型名
$agent = new Agent(
    platform: $platform,
    model: 'deepseek-chat',
    inputProcessors: [
        new SystemPromptInputProcessor($systemPrompt),
        $agentProcessor,
    ],
    outputProcessors: [$agentProcessor],
);
```

安装 DeepSeek Bridge：

```bash
composer require symfony/ai-deep-seek-platform
```

> **💡 平台切换提示：** Symfony AI 的 Bridge 架构让平台切换变得极其简单——只需更换 `PlatformFactory` 和模型名称。所有业务逻辑（消息构建、Agent 编排、Store 操作）完全不受影响。你甚至可以在运行时根据配置动态选择平台。

也可以将 `YoutubeTranscriber` 注册到 Agent 的 Toolbox 中，让大模型自主决定何时获取字幕：

```php
<?php

// YoutubeTranscriber 本身标注了 #[AsTool]，可以直接作为 Agent 工具
$toolbox = new Toolbox([
    $searchTool,       // 语义检索工具
    $transcriber,      // YouTube 字幕获取工具——大模型可自主调用
]);

// 现在大模型可以自己决定是否需要获取新视频的字幕
// 例如学员问"帮我总结一下这个视频的内容：https://youtube.com/watch?v=xxx"
// Agent 会自动调用 youtube_transcript 工具
```

> **⚠️ 注意事项：** 将 `YoutubeTranscriber` 注册为 Agent 工具时，大模型会自主决定何时调用。这意味着如果学员的问题中包含视频链接，Agent 可能会实时获取字幕。注意控制调用频率和成本。

---

## 完整流程

```
YouTube 视频库
    │
    ▼
[YoutubeTranscriber] ─────────── 获取字幕文本（参数为视频 ID）
    │
    ▼
[TextSplitTransformer] ──────── 分段（500 字/段，重叠 100 字）
    │                            自动设置 parent_id 和 _text 元数据
    ▼
[DocumentProcessor] ─────────── 管线：过滤 → 转换 → 向量化 → 存储
    │                            Vectorizer 使用 OpenAI text-embedding-3-small
    ▼
[PostgreSQL + pgvector] ──────── 向量索引构建完成 ✅
    │
    ├──► [学员提问] ──▶ Agent + SimilaritySearch ──▶ Mistral 推理 ──▶ 精准回答 + 视频来源
    │
    ├──► [多轮追问] ──▶ Chat + DoctrineDbalStore ──▶ 保持上下文连贯性
    │
    ├──► [学习推荐] ──▶ StructuredOutput ──▶ LearningPath DTO（结构化推荐）
    │
    └──► [生产优化] ──▶ CachePlatform ──▶ 缓存高频问题，降低 60–80% API 成本
```

---

## 知识点总结

| 概念 | 类/接口 | 说明 |
|------|---------|------|
| YouTube 字幕获取 | `YoutubeTranscriber` | 通过视频 ID 获取字幕文本，同时也是 `#[AsTool]` 标注的 Agent 工具 |
| 文本分段 | `TextSplitTransformer` | 按指定大小分段，支持重叠，自动继承父文档元数据 |
| 向量化 | `Vectorizer` | 将文本/文档转换为向量，支持批量处理 |
| 文档处理管线 | `DocumentProcessor` | 过滤 → 转换 → 向量化 → 存储的完整管线 |
| 文档索引 | `DocumentIndexer` | `DocumentProcessor` 的上层封装，接受文档并触发处理管线 |
| 向量存储 | `Store`（PostgreSQL） | 基于 pgvector，支持 `VectorQuery`、`TextQuery`、`HybridQuery` |
| 语义检索 | `SimilaritySearch` | Agent 工具，向量化查询后从 Store 中检索相似文档 |
| Agent 编排 | `Agent` + `AgentProcessor` | InputProcessor / OutputProcessor 管线，Toolbox 工具调用 |
| 对话持久化 | `DoctrineDbalMessageStore` | 基于 Doctrine DBAL 的消息存储，支持多轮对话 |
| 结构化输出 | `StructuredOutput` | 大模型返回与 PHP DTO 类匹配的 JSON，自动反序列化 |
| 缓存优化 | `CachePlatform` | 装饰器模式，缓存 LLM 调用结果，降低重复请求成本 |
| 平台切换 | `PlatformFactory` | 更换 Bridge 即可切换 AI 平台，业务代码零改动 |

## 下一步

如果你需要自动抓取和分析竞品网站信息，请看 [27-competitor-intelligence-monitoring.md](./27-competitor-intelligence-monitoring.md)。

