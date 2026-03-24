# AI 会议纪要助手

## 业务场景

企业每天都在开会：产品评审、技术方案讨论、客户沟通、项目周会、管理层决策会。会后整理纪要是一项繁琐但重要的工作——谁说了什么、做了哪些决定、有哪些待办事项，通常由轮值的同事花费 30-60 分钟手动整理，还经常出现遗漏或理解偏差。

本教程将构建一个完整的 AI 会议纪要助手，实现从音频录音到结构化纪要的全自动化流程，并进一步实现会议历史的向量化存储与智能检索，让你能够跨会议查询"上次谁负责了什么"、"某个决策的背景是什么"。

**典型应用场景：**

- 会议纪要自动生成与归档
- 跨会议行动项追踪与复盘
- 客户沟通记录结构化存储
- 培训课程笔记提取
- 播客/访谈内容结构化分析

---

## 涉及模块

| 模块 | 类 / 组件 | 用途 |
|------|-----------|------|
| **Platform (ElevenLabs)** | `ElevenLabs\PlatformFactory` | 语音转文字（STT） |
| **Platform (Anthropic)** | `Anthropic\PlatformFactory` | Claude 模型生成结构化纪要 |
| **Platform (Gemini)** | `Gemini\PlatformFactory` | 原生音频分析（替代方案） |
| **Audio Content** | `Message\Content\Audio` | 音频文件加载 |
| **StructuredOutput** | `PlatformSubscriber` | 将 AI 输出映射为 PHP 对象 |
| **TextSplitTransformer** | `Document\Transformer\TextSplitTransformer` | 长文本分块处理 |
| **Vectorizer** | `Document\Vectorizer` | 文本向量化 |
| **DocumentIndexer** | `Indexer\DocumentIndexer` | 文档索引入库 |
| **Store (PostgreSQL)** | `Bridge\Postgres\Store` | 向量数据库持久化存储 |
| **Retriever** | `Store\Retriever` | 向量相似度检索 |
| **Agent** | `Agent\Agent` | 智能会议回顾助手 |
| **EmbeddingProvider** | `Agent\Memory\EmbeddingProvider` | 语义记忆（历史会议） |
| **MemoryInputProcessor** | `Agent\Memory\MemoryInputProcessor` | 记忆注入到 Agent 上下文 |
| **SimilaritySearch** | `Bridge\SimilaritySearch\SimilaritySearch` | Agent 搜索工具 |
| **FaultTolerantToolbox** | `Agent\Toolbox\FaultTolerantToolbox` | 容错工具调用 |
| **Chat** | `Chat\Chat` | 对话管理 |
| **Chat Store (Doctrine)** | `Bridge\Doctrine\DoctrineDbalMessageStore` | 对话历史持久化 |

---

## 项目实现思路

整个项目分为三个层次：

1. **基础处理层**：音频转录 → 长文本分块 → 结构化提取（Step 1-4）
2. **分析增强层**：情感分析 → 向量化存储 → 历史检索（Step 5-6）
3. **智能交互层**：Agent 回顾助手 → 对话式查询 → 替代方案（Step 7-10）

每一层都在前一层的基础上递进，从"生成纪要"到"理解纪要"再到"与纪要对话"。

---

## 项目流程图

```
+-------------------+
|  会议录音文件     |
|  (mp3/wav/m4a)    |
+--------+----------+
         |
         v
+--------+----------+     +---------------------+
| ElevenLabs STT    |     | Gemini 原生音频分析 |
| (scribe_v1)       |     | (替代方案，Step 9)  |
+--------+----------+     +----------+----------+
         |                            |
         v                            v
+--------+----------+     +-----------+---------+
| 文字转录文本      |     | 直接输出分析结果    |
+--------+----------+     +---------------------+
         |
         v
+--------+----------+
| TextSplitTransformer |
| 长文本分块 (Step 2)  |
+--------+--------------+
         |
    +----+----+
    |         |
    v         v
+---+------+  +--------+-----------+
| Claude   |  | Vectorizer         |
| 结构化   |  | 文本向量化 (Step 6)|
| (Step 4) |  +--------+-----------+
+---+------+           |
    |                  v
    v         +--------+-----------+
+---+------+  | PostgreSQL pgvector|
| 会议纪要 |  | 持久化存储         |
| DTO 对象 |  +--------+-----------+
+---+------+           |
    |                  v
    v         +--------+-----------+
+---+-------+ | Retriever 检索    |
| 情感分析  | +--------+-----------+
| (Step 5)  |          |
+---+-------+          v
    |         +--------+-----------+
    v         | Agent 智能回顾     |
+---+-------+ | + EmbeddingProvider|
| 格式化   | | + SimilaritySearch |
| 输出     | | (Step 7)           |
+-----------+ +--------+-----------+
                       |
                       v
              +--------+-----------+
              | Chat 对话查询      |
              | Doctrine DBAL 持久 |
              | (Step 8)           |
              +--------------------+
```

---

## 前置准备

### 安装依赖

```bash
# 核心平台 + Anthropic Claude（本教程主力 LLM）
composer require symfony/ai-platform symfony/ai-platform-anthropic

# ElevenLabs 语音转文字
composer require symfony/ai-platform-elevenlabs

# Gemini 原生音频分析（替代方案）
composer require symfony/ai-platform-gemini

# 向量存储 + PostgreSQL pgvector
composer require symfony/ai-store symfony/ai-store-postgres

# Agent 智能助手
composer require symfony/ai-agent symfony/ai-agent-similarity-search

# Chat 对话持久化
composer require symfony/ai-chat symfony/ai-chat-doctrine

# Symfony 基础组件
composer require symfony/event-dispatcher symfony/http-client
```

> **提示：** 本教程使用 Anthropic Claude 作为主力 LLM，与其他教程中常用的 OpenAI 形成互补。Claude 在长文本理解和结构化提取方面表现优秀，非常适合会议纪要场景。

### 环境变量

```bash
# .env
ELEVENLABS_API_KEY=your-elevenlabs-key      # ElevenLabs STT
ANTHROPIC_API_KEY=your-anthropic-key         # Claude 结构化分析
GEMINI_API_KEY=your-gemini-key               # Gemini 音频分析（可选）
DATABASE_URL=pgsql://user:pass@localhost:5432/meetings  # PostgreSQL + pgvector
```

> **注意：** PostgreSQL 需要安装 pgvector 扩展。可以使用 `CREATE EXTENSION IF NOT EXISTS vector;` 启用。Docker 用户可直接使用 `pgvector/pgvector:pg16` 镜像。

---

## Step 1：音频转录（Speech-to-Text）

第一步是将会议录音转换为文字。我们使用 ElevenLabs 的 `scribe_v1` 模型，它支持多语言识别且转录质量较高。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\ElevenLabs\PlatformFactory as ElevenLabsFactory;
use Symfony\AI\Platform\Message\Content\Audio;

// 创建 ElevenLabs 平台实例
$elevenLabs = ElevenLabsFactory::create(
    endpoint: 'https://api.elevenlabs.io/v1/',
    apiKey: $_ENV['ELEVENLABS_API_KEY'],
);

// 加载会议录音文件
$audio = Audio::fromFile('/path/to/meeting-recording.mp3');

// 调用 STT 模型转录
$result = $elevenLabs->invoke('scribe_v1', $audio);

$transcript = $result->asText();

echo "转录完成（" . mb_strlen($transcript) . " 字）\n";
echo "前 500 字预览：\n";
echo mb_substr($transcript, 0, 500) . "...\n";
```

> **提示：** ElevenLabs 的 `scribe_v1` 支持 mp3、wav、m4a、webm 等常见格式。单次请求最大文件为 1GB。对于超长会议（3 小时以上），建议先用 ffmpeg 按固定时长切分后分段转录。

> **注意：** `Audio::fromFile()` 继承自 `File` 基类，会自动检测 MIME 类型。如果文件不存在或不可读，将抛出 `InvalidArgumentException`。

---

## Step 2：长文本分块处理

一场 60 分钟的会议转录通常有 8000-15000 字。直接将全文发送给 LLM 可能超出上下文窗口，且不利于后续向量化索引。`TextSplitTransformer` 可以将长文本按语义边界分割为合适的块。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;

// 假设 $transcript 是 Step 1 中获取的完整转录文本
$transcript = '张经理：好，我们开始今天的产品评审会...(完整转录内容)';

// 创建原始文档
$metadata = new Metadata();
$metadata->setSource('meeting-2025-01-15.mp3');
$metadata->setTitle('产品评审会 2025-01-15');
$document = new TextDocument('meeting-20250115', $transcript, $metadata);

// 配置分块参数：每块 2000 字符，重叠 300 字符
$splitter = new TextSplitTransformer(chunkSize: 2000, overlap: 300);

// 执行分块
$chunks = iterator_to_array($splitter->transform([$document]));

echo "原文长度：" . mb_strlen($transcript) . " 字\n";
echo "分块数量：" . count($chunks) . "\n\n";

foreach ($chunks as $i => $chunk) {
    echo "--- 分块 " . ($i + 1) . " ---\n";
    echo "长度：" . mb_strlen($chunk->getContent()) . " 字\n";
    echo "预览：" . mb_substr($chunk->getContent(), 0, 100) . "...\n\n";
}
```

> **提示：** `overlap`（重叠）参数确保分块边界处的上下文不丢失。会议转录建议使用 200-400 的重叠值，因为一句话可能跨越分块边界。对于结构化提取（Step 4），建议将完整文本发送给 Claude（其 200K 上下文窗口足够处理大多数会议）；分块主要用于后续的向量化索引（Step 6）。

> **注意：** `TextSplitTransformer` 按字符数分割，不是按 Token 数。中文字符通常 1 字符 = 1-2 Token，因此 2000 字符约等于 2000-4000 Token。

---

## Step 3：定义会议纪要结构

使用 PHP DTO（Data Transfer Object）定义会议纪要的嵌套结构。Symfony AI 的 StructuredOutput 会根据这些类的属性和 PHPDoc 注解自动生成 JSON Schema，引导 LLM 输出符合格式的数据。

### 基础数据结构

```php
<?php

namespace App\Dto;

final class ActionItem
{
    /**
     * @param string $owner    负责人姓名
     * @param string $task     任务描述
     * @param string $deadline 截止时间，如 "2025-01-20" 或 "未提及"
     * @param string $priority 优先级：high / medium / low
     */
    public function __construct(
        public readonly string $owner,
        public readonly string $task,
        public readonly string $deadline,
        public readonly string $priority,
    ) {
    }
}

final class Decision
{
    /**
     * @param string $topic     决策主题
     * @param string $outcome   决策结果
     * @param string $rationale 决策依据或讨论过程
     */
    public function __construct(
        public readonly string $topic,
        public readonly string $outcome,
        public readonly string $rationale,
    ) {
    }
}

final class DiscussionTopic
{
    /**
     * @param string   $topic        讨论主题
     * @param string   $summary      讨论摘要（2-3 句话）
     * @param string[] $keyPoints    关键观点列表
     * @param string[] $participants 参与讨论的人员
     */
    public function __construct(
        public readonly string $topic,
        public readonly string $summary,
        public readonly array $keyPoints,
        public readonly array $participants,
    ) {
    }
}

final class MeetingMinutes
{
    /**
     * @param string            $title           会议标题
     * @param string            $date            会议日期（YYYY-MM-DD 格式）
     * @param string            $duration        时长，如 "45 分钟"
     * @param string[]          $attendees       参会人员列表
     * @param string            $executiveSummary 一段话摘要（100 字以内）
     * @param DiscussionTopic[] $discussions     讨论主题列表
     * @param Decision[]        $decisions       决议列表
     * @param ActionItem[]      $actionItems     行动项列表
     * @param string            $nextMeeting     下次会议安排
     */
    public function __construct(
        public readonly string $title,
        public readonly string $date,
        public readonly string $duration,
        public readonly array $attendees,
        public readonly string $executiveSummary,
        public readonly array $discussions,
        public readonly array $decisions,
        public readonly array $actionItems,
        public readonly string $nextMeeting,
    ) {
    }
}
```

> **提示：** PHPDoc 中的 `@param` 描述会被 StructuredOutput 用作 JSON Schema 的 `description` 字段，直接影响 LLM 的输出质量。描述越具体（如"YYYY-MM-DD 格式"），输出越规范。嵌套的数组类型（如 `ActionItem[]`）会自动递归解析。

---

## Step 4：生成结构化会议纪要

使用 Anthropic Claude 从转录文本中提取结构化的会议纪要。通过 `PlatformSubscriber` 启用 StructuredOutput，将 LLM 的输出直接映射为 Step 3 中定义的 DTO 对象。

```php
<?php

require 'vendor/autoload.php';

use App\Dto\MeetingMinutes;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

// 创建 Anthropic Claude 平台实例（启用 StructuredOutput）
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

$platform = PlatformFactory::create(
    apiKey: $_ENV['ANTHROPIC_API_KEY'],
    eventDispatcher: $dispatcher,
);

// 构建提示词
$messages = new MessageBag(
    Message::forSystem(
        "你是专业的会议纪要整理专家。请从会议转录文本中提取以下信息：\n"
        . "1. 参会人员：从对话中识别所有发言者\n"
        . "2. 讨论主题：每个议题的摘要和关键观点\n"
        . "3. 决议：所有明确达成的决定及其依据\n"
        . "4. 行动项：谁负责、做什么、什么时候完成\n"
        . "5. 下次会议安排\n\n"
        . "注意：如果某项信息在转录中未明确提及，标注为'未提及'而非猜测。"
    ),
    Message::ofUser("请整理以下会议转录的纪要：\n\n{$transcript}"),
);

// 调用 Claude 并指定输出格式为 MeetingMinutes DTO
$result = $platform->invoke('claude-sonnet-4-20250514', $messages, [
    'response_format' => MeetingMinutes::class,
]);

/** @var MeetingMinutes $minutes */
$minutes = $result->asObject();

// 格式化输出
echo "============================================\n";
echo "  会议纪要\n";
echo "============================================\n\n";

echo "标题：{$minutes->title}\n";
echo "日期：{$minutes->date} | 时长：{$minutes->duration}\n";
echo "参会：" . implode('、', $minutes->attendees) . "\n\n";

echo "【摘要】\n{$minutes->executiveSummary}\n\n";

echo "---- 讨论内容 ----\n\n";
foreach ($minutes->discussions as $i => $topic) {
    echo ($i + 1) . ". {$topic->topic}\n";
    echo "   {$topic->summary}\n";
    foreach ($topic->keyPoints as $point) {
        echo "   - {$point}\n";
    }
    echo "   参与者：" . implode('、', $topic->participants) . "\n\n";
}

if ([] !== $minutes->decisions) {
    echo "---- 决议 ----\n\n";
    foreach ($minutes->decisions as $decision) {
        echo "* {$decision->topic}\n";
        echo "  结果：{$decision->outcome}\n";
        echo "  依据：{$decision->rationale}\n\n";
    }
}

if ([] !== $minutes->actionItems) {
    echo "---- 行动项 ----\n\n";
    foreach ($minutes->actionItems as $item) {
        $flag = match ($item->priority) {
            'high' => '[!!!]',
            'medium' => '[!! ]',
            'low' => '[!  ]',
            default => '[   ]',
        };
        echo "{$flag} [{$item->owner}] {$item->task}\n";
        echo "      截止：{$item->deadline}\n\n";
    }
}

echo "下次会议：{$minutes->nextMeeting}\n";
```

> **提示：** `PlatformSubscriber` 是 StructuredOutput 的核心。它监听平台的调用事件，在请求发送前将 DTO 类转换为 JSON Schema 注入到 LLM 请求中，在响应返回后将 JSON 反序列化为 PHP 对象。你只需要 `'response_format' => MyDto::class` 和 `$result->asObject()` 两步即可。

> **注意：** Claude 的上下文窗口为 200K Token，足以处理 3-4 小时的会议转录。如果你的会议更长，才需要结合 Step 2 的分块策略，对每个分块分别提取后合并结果。

---

## Step 5：会议情感与参与度分析

除了基本的纪要提取，我们还可以分析每位参与者的发言比例、情感倾向和会议整体效率。这对团队管理和会议优化很有价值。

### 分析数据结构

```php
<?php

namespace App\Dto;

final class ParticipantAnalysis
{
    /**
     * @param string $name             参与者姓名
     * @param int    $speakingShare    发言占比（0-100）
     * @param string $role             角色：facilitator / contributor / observer
     * @param string $sentiment        情感基调：positive / neutral / concerned / frustrated
     * @param string $keyContribution  该参与者的主要贡献（一句话）
     */
    public function __construct(
        public readonly string $name,
        public readonly int $speakingShare,
        public readonly string $role,
        public readonly string $sentiment,
        public readonly string $keyContribution,
    ) {
    }
}

final class MeetingAnalysis
{
    /**
     * @param string                $meetingEfficiency      会议效率评级：high / medium / low
     * @param string                $efficiencyExplanation  效率评级的依据说明
     * @param ParticipantAnalysis[] $participantAnalysis    各参与者分析
     * @param string[]              $improvementSuggestions 会议改进建议列表
     * @param int                   $actionableRate         行动项明确率（0-100）
     */
    public function __construct(
        public readonly string $meetingEfficiency,
        public readonly string $efficiencyExplanation,
        public readonly array $participantAnalysis,
        public readonly array $improvementSuggestions,
        public readonly int $actionableRate,
    ) {
    }
}
```

### 执行分析

```php
<?php

require 'vendor/autoload.php';

use App\Dto\MeetingAnalysis;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

$platform = PlatformFactory::create(
    apiKey: $_ENV['ANTHROPIC_API_KEY'],
    eventDispatcher: $dispatcher,
);

$messages = new MessageBag(
    Message::forSystem(
        "你是组织效能顾问。分析会议转录文本，评估会议效率和各参与者的表现。\n"
        . "注意：发言占比之和应为 100。角色判断基于实际发言内容而非职位。"
    ),
    Message::ofUser("请分析以下会议的效能：\n\n{$transcript}"),
);

$result = $platform->invoke('claude-sonnet-4-20250514', $messages, [
    'response_format' => MeetingAnalysis::class,
]);

/** @var MeetingAnalysis $analysis */
$analysis = $result->asObject();

echo "=== 会议效能分析 ===\n\n";
echo "效率评级：{$analysis->meetingEfficiency}\n";
echo "评级依据：{$analysis->efficiencyExplanation}\n";
echo "行动项明确率：{$analysis->actionableRate}%\n\n";

echo "--- 参与者分析 ---\n";
foreach ($analysis->participantAnalysis as $p) {
    echo "  {$p->name}：{$p->speakingShare}% 发言 | {$p->role} | {$p->sentiment}\n";
    echo "    贡献：{$p->keyContribution}\n";
}

echo "\n--- 改进建议 ---\n";
foreach ($analysis->improvementSuggestions as $suggestion) {
    echo "  - {$suggestion}\n";
}
```

> **提示：** 情感分析结果可以帮助管理者发现团队中的潜在问题。如果某位成员持续表现为 "frustrated" 或 "concerned"，可能需要私下沟通。但请注意 AI 的情感分析并非 100% 准确，应作为参考而非依据。

---

## Step 6：会议纪要向量化存储与检索

将会议纪要存入向量数据库后，就能实现跨会议的语义搜索——比如查询"关于移动端重构的所有讨论"或"张经理负责的待办事项"。本步骤使用 PostgreSQL + pgvector 作为生产级存储方案。

### 6.1 初始化向量存储

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory as OpenAiPlatformFactory;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\AI\Store\Bridge\Postgres\Distance;
use Symfony\AI\Store\Bridge\Postgres\Store as PostgresStore;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\Component\EventDispatcher\EventDispatcher;

// 连接 PostgreSQL（需已安装 pgvector 扩展）
$pdo = new \PDO(
    'pgsql:host=localhost;port=5432;dbname=meetings',
    'user',
    'pass',
);
$vectorStore = new PostgresStore(
    connection: $pdo,
    tableName: 'meeting_vectors',
    vectorFieldName: 'embedding',
    distance: Distance::Cosine,
);

// 创建表结构（首次运行）
$vectorStore->setup();

// 创建向量化工具（使用 Anthropic 平台的嵌入功能并非最佳选择，
// 这里我们单独创建一个 OpenAI 平台用于 Embedding）
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

$embeddingPlatform = OpenAiPlatformFactory::create(
    apiKey: $_ENV['OPENAI_API_KEY'],
    eventDispatcher: $dispatcher,
);

$vectorizer = new Vectorizer(
    platform: $embeddingPlatform,
    model: 'text-embedding-3-small',
);

echo "向量存储初始化完成\n";
```

> **注意：** 目前主流 Embedding 模型来自 OpenAI（text-embedding-3-small/large）。即使主力 LLM 使用 Claude，Embedding 部分仍推荐 OpenAI 的模型，性价比最高。这也体现了 Symfony AI 的多平台混用能力——不同任务使用不同平台的最佳模型。

### 6.2 索引会议文档

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentIndexer;
use Symfony\AI\Store\Indexer\DocumentProcessor;

// 使用 Step 6.1 中创建的 $vectorizer 和 $vectorStore

// 创建文档处理器：分块 -> 向量化 -> 存储
$processor = new DocumentProcessor(
    vectorizer: $vectorizer,
    store: $vectorStore,
    transformers: [
        new TextSplitTransformer(chunkSize: 1500, overlap: 200),
    ],
);

$indexer = new DocumentIndexer(processor: $processor);

// 假设 $minutes 是 Step 4 中生成的 MeetingMinutes 对象
// 将纪要摘要和行动项作为可检索文档索引

// 索引会议摘要
$summaryMeta = new Metadata();
$summaryMeta->setSource('meeting-2025-01-15');
$summaryMeta->setTitle($minutes->title);
$summaryMeta['type'] = 'summary';
$summaryMeta['date'] = $minutes->date;

$indexer->index(new TextDocument(
    id: 'meeting-20250115-summary',
    content: "会议：{$minutes->title}\n日期：{$minutes->date}\n"
        . "参会：" . implode('、', $minutes->attendees) . "\n"
        . "摘要：{$minutes->executiveSummary}",
    metadata: $summaryMeta,
));

// 索引每个讨论主题
foreach ($minutes->discussions as $i => $topic) {
    $topicMeta = new Metadata();
    $topicMeta->setSource('meeting-2025-01-15');
    $topicMeta->setTitle($topic->topic);
    $topicMeta['type'] = 'discussion';
    $topicMeta['date'] = $minutes->date;

    $indexer->index(new TextDocument(
        id: "meeting-20250115-topic-{$i}",
        content: "讨论主题：{$topic->topic}\n"
            . "摘要：{$topic->summary}\n"
            . "关键观点：" . implode('；', $topic->keyPoints) . "\n"
            . "参与者：" . implode('、', $topic->participants),
        metadata: $topicMeta,
    ));
}

// 索引行动项
foreach ($minutes->actionItems as $i => $item) {
    $actionMeta = new Metadata();
    $actionMeta->setSource('meeting-2025-01-15');
    $actionMeta['type'] = 'action_item';
    $actionMeta['date'] = $minutes->date;
    $actionMeta['owner'] = $item->owner;
    $actionMeta['priority'] = $item->priority;

    $indexer->index(new TextDocument(
        id: "meeting-20250115-action-{$i}",
        content: "行动项：{$item->task}\n"
            . "负责人：{$item->owner}\n"
            . "截止日期：{$item->deadline}\n"
            . "优先级：{$item->priority}",
        metadata: $actionMeta,
    ));
}

echo "会议文档索引完成\n";
```

### 6.3 检索历史会议

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Store\Retriever;

// 使用 Step 6.1 中创建的 $vectorStore 和 $vectorizer
$retriever = new Retriever(
    store: $vectorStore,
    vectorizer: $vectorizer,
);

// 语义搜索：查找与查询最相关的会议内容
$query = '移动端重构的进展和负责人';
$results = $retriever->retrieve($query, ['limit' => 5]);

echo "查询：{$query}\n";
echo "============================================\n\n";

foreach ($results as $doc) {
    $meta = $doc->getMetadata();
    $score = $doc->getScore();
    $type = $meta['type'] ?? 'unknown';
    $date = $meta['date'] ?? 'unknown';
    $title = $meta->getTitle() ?? '';

    echo "[{$type}] {$title} ({$date}) - 相关度：" . round($score, 4) . "\n";
    echo $meta->getText() ?? '(无文本)';
    echo "\n\n";
}
```

> **提示：** `Retriever` 是 Store 模块的高级检索接口，它封装了"查询文本向量化 -> 向量相似度搜索 -> 返回排序结果"的完整流程。你只需传入自然语言查询，无需手动处理向量转换。

> **警告：** 生产环境中，确保 PostgreSQL 的 pgvector 索引配置合理。对于超过 10 万条向量，建议使用 IVFFlat 或 HNSW 索引加速查询。参考命令：`CREATE INDEX ON meeting_vectors USING hnsw (embedding vector_cosine_ops);`

---

## Step 7：智能会议回顾 Agent

将向量检索能力封装为 Agent 工具，构建一个能够回答"上周的决议有哪些"、"李工负责了哪些任务"之类问题的智能助手。Agent 通过 `EmbeddingProvider` 自动从历史会议中加载语义相关的上下文。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Memory\EmbeddingProvider;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;
use Symfony\AI\Agent\Memory\StaticMemoryProvider;
use Symfony\AI\Agent\Toolbox\FaultTolerantToolbox;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\Model;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

// --- 平台初始化 ---
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

$claudePlatform = PlatformFactory::create(
    apiKey: $_ENV['ANTHROPIC_API_KEY'],
    eventDispatcher: $dispatcher,
);

// 使用 Step 6.1 中创建的 $embeddingPlatform, $vectorizer, $vectorStore

// --- 工具配置 ---

// SimilaritySearch：让 Agent 能搜索历史会议
$searchTool = new SimilaritySearch(
    vectorizer: $vectorizer,
    store: $vectorStore,
);

// 包装为容错工具箱（工具调用失败时不会中断整个 Agent 流程）
$toolbox = new FaultTolerantToolbox(
    innerToolbox: new Toolbox(tools: [$searchTool]),
);

// --- 记忆系统 ---

// EmbeddingProvider：根据用户输入自动检索相关历史会议
$embeddingMemory = new EmbeddingProvider(
    platform: $embeddingPlatform,
    model: new Model('text-embedding-3-small'),
    vectorStore: $vectorStore,
);

// StaticMemoryProvider：固定上下文信息
$staticMemory = new StaticMemoryProvider(
    '当前日期：2025-01-20',
    '团队成员：张经理（产品）、李工（前端）、王工（后端）、赵工（测试）',
    '当前迭代：Sprint 12，截止日期 2025-01-31',
);

// --- 创建 Agent ---
$agent = new Agent(
    platform: $claudePlatform,
    model: 'claude-sonnet-4-20250514',
    inputProcessors: [
        new SystemPromptInputProcessor(
            systemPrompt: "你是团队的会议回顾助手。你可以查询历史会议记录来回答问题。\n"
                . "回答时请引用具体的会议日期和讨论内容。\n"
                . "如果找不到相关信息，请如实说明。",
            toolbox: $toolbox,
        ),
        new MemoryInputProcessor(
            memoryProviders: [$embeddingMemory, $staticMemory],
        ),
    ],
);

// --- 使用 Agent ---
$messages = new MessageBag(
    Message::ofUser('李工最近负责了哪些任务？进展如何？'),
);

$response = $agent->call($messages);

echo "助手回复：\n";
echo $response->getContent() . "\n";

// 查看 SimilaritySearch 引用了哪些文档
if ([] !== $searchTool->usedDocuments) {
    echo "\n--- 引用来源 ---\n";
    foreach ($searchTool->usedDocuments as $doc) {
        $meta = $doc->getMetadata();
        echo "- [{$meta['type']}] " . ($meta->getTitle() ?? '无标题')
            . " ({$meta['date']})\n";
    }
}
```

> **提示：** `FaultTolerantToolbox` 包装了内部工具箱，当某个工具调用因参数错误或运行异常而失败时，它会将错误信息返回给 Agent 而非抛出异常，让 Agent 有机会修正参数后重试。这在生产环境中非常重要，避免因单次工具调用失败导致整个对话中断。

> **注意：** `MemoryInputProcessor` 是记忆系统的入口。它接收一组 `MemoryProviderInterface` 实现，在每次 Agent 调用时遍历所有 Provider 加载相关记忆，注入到系统消息中。`EmbeddingProvider` 会根据每次用户输入动态检索相关内容；`StaticMemoryProvider` 则固定注入相同的背景信息。两者配合使用可以兼顾动态相关性和固定背景知识。

---

## Step 8：对话式会议查询（Chat 持久化）

Step 7 的 Agent 每次调用都是独立的，没有对话历史。通过 Chat 模块和 Doctrine DBAL 存储，可以实现多轮对话式查询，用户能够追问、澄清、深入讨论。

```php
<?php

require 'vendor/autoload.php';

use Doctrine\DBAL\DriverManager;
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Chat\Bridge\Doctrine\DoctrineDbalMessageStore;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

// --- 初始化 Doctrine DBAL 连接 ---
$dbalConnection = DriverManager::getConnection([
    'url' => $_ENV['DATABASE_URL'],
]);

// 创建 Chat 消息存储（使用 Doctrine DBAL）
$chatStore = new DoctrineDbalMessageStore(
    tableName: 'chat_messages',
    dbalConnection: $dbalConnection,
);

// 首次运行创建表
$chatStore->setup();

// --- 创建 Agent ---
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

$platform = PlatformFactory::create(
    apiKey: $_ENV['ANTHROPIC_API_KEY'],
    eventDispatcher: $dispatcher,
);

$agent = new Agent(
    platform: $platform,
    model: 'claude-sonnet-4-20250514',
    inputProcessors: [
        new SystemPromptInputProcessor(
            systemPrompt: "你是会议纪要查询助手。用户会询问关于过去会议的问题。\n"
                . "请基于已有信息回答，支持多轮追问。\n"
                . "如果信息不足，建议用户提供更具体的查询条件。",
        ),
    ],
);

// --- 创建 Chat 实例 ---
$chat = new Chat(
    agent: $agent,
    store: $chatStore,
);

// --- 多轮对话示例 ---

// 第一轮：提交问题并获取回复
$response = $chat->submit(Message::ofUser('帮我回顾一下上周产品评审会的主要决议'));
echo "助手：" . $response->getContent() . "\n\n";

// 第二轮：追问细节（Chat 自动携带上一轮对话上下文）
$response = $chat->submit(Message::ofUser('其中关于移动端的决议，具体是谁负责的？'));
echo "助手：" . $response->getContent() . "\n\n";

// 第三轮：继续深入
$response = $chat->submit(Message::ofUser('他的截止日期是什么时候？需要哪些资源支持？'));
echo "助手：" . $response->getContent() . "\n\n";
```

> **提示：** `DoctrineDbalMessageStore` 将完整的对话历史（包括系统消息、用户消息、助手回复）序列化为 JSON 存入关系数据库。它使用 Symfony Serializer 组件处理 `MessageBag` 的序列化/反序列化，支持所有消息类型。

> **注意：** Chat 模块提供了多种存储后端。以下是选择建议：
> - **开发/测试**：`Symfony\AI\Chat\InMemory\Store`，无需外部依赖
> - **生产（关系型数据库）**：`DoctrineDbalMessageStore`，适合已有 PostgreSQL/MySQL 的项目
> - **生产（高并发）**：`Symfony\AI\Chat\Bridge\Redis\MessageStore`，适合需要快速会话读写的场景
> - **生产（文档型）**：`Symfony\AI\Chat\Bridge\MongoDb\Store`，适合已有 MongoDB 的项目

---

## Step 9：Gemini 直接分析音频（替代方案）

Google Gemini 支持原生音频输入，可以直接从音频中理解内容，无需先转录为文字。这种方案更简洁，且能捕捉语音中的语气、停顿等非文字信息。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory as GeminiFactory;
use Symfony\AI\Platform\Message\Content\Audio;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$gemini = GeminiFactory::create(
    apiKey: $_ENV['GEMINI_API_KEY'],
);

// Gemini 直接分析音频（不需要先转录）
$messages = new MessageBag(
    Message::forSystem(
        "你是会议纪要专家。请直接从音频中提取：\n"
        . "1. 会议标题和日期\n"
        . "2. 参会人员\n"
        . "3. 主要讨论内容和关键观点\n"
        . "4. 所有决议\n"
        . "5. 行动项（负责人、任务、截止日期）\n"
        . "注意聆听语气变化，标注讨论中的分歧和共识。"
    ),
    Message::ofUser(
        Audio::fromFile('/path/to/meeting-recording.mp3'),
    ),
);

$result = $gemini->invoke('gemini-2.0-flash', $messages);

echo "Gemini 直接音频分析结果：\n";
echo "============================================\n";
echo $result->asText() . "\n";
```

> **提示：** Gemini 的原生音频分析有以下优势：（1）省去转录步骤，流程更简单；（2）能感知语调变化（如犹豫、强调）；（3）多语言混合会议处理更自然。劣势是：（1）输出为自由文本，不如 StructuredOutput 精确；（2）长音频处理可能受限于模型上下文窗口。建议在需要精确结构化输出时使用 ElevenLabs + Claude 方案，在快速摘要场景使用 Gemini 方案。

---

## Step 10：语音播报会议摘要（TTS）

生成的会议纪要还可以转换为语音，方便参会者在通勤时收听。ElevenLabs 同时提供 TTS 能力。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\ElevenLabs\PlatformFactory as ElevenLabsFactory;
use Symfony\AI\Platform\Message\Content\Text;

$elevenLabs = ElevenLabsFactory::create(
    endpoint: 'https://api.elevenlabs.io/v1/',
    apiKey: $_ENV['ELEVENLABS_API_KEY'],
);

// 假设 $minutes 来自 Step 4
$spokenSummary = "会议纪要摘要：{$minutes->title}，{$minutes->date}。"
    . "{$minutes->executiveSummary}。"
    . "共有 " . count($minutes->actionItems) . " 项待办事项。";

$result = $elevenLabs->invoke('eleven_multilingual_v2', new Text($spokenSummary));

// 保存为音频文件（BinaryResult 提供 asFile() 快捷方法）
$result->asFile('/path/to/meeting-summary.mp3');

echo "语音摘要已生成：meeting-summary.mp3\n";
```

> **提示：** TTS 适合生成 1-2 分钟的精简摘要，而非完整纪要。建议只朗读 `executiveSummary` 和高优先级行动项。ElevenLabs 的 `eleven_multilingual_v2` 模型支持中文。

---

## 完整流程

```
+------------------------------------------------------------+
|                    AI 会议纪要助手                          |
+------------------------------------------------------------+
|                                                            |
|  [输入] 会议录音 (mp3/wav/m4a)                             |
|     |                                                      |
|     +---> ElevenLabs STT (scribe_v1)                       |
|     |        |                                             |
|     |        v                                             |
|     |     转录文本                                         |
|     |        |                                             |
|     |        +---> TextSplitTransformer (分块)             |
|     |        |        |                                    |
|     |        |        +---> Vectorizer (向量化)            |
|     |        |        |        |                           |
|     |        |        |        v                           |
|     |        |        |     PostgreSQL pgvector (存储)     |
|     |        |        |        |                           |
|     |        |        |        v                           |
|     |        |        |     Retriever (检索)               |
|     |        |        |        |                           |
|     |        |        |        v                           |
|     |        |        |     Agent + EmbeddingProvider      |
|     |        |        |     + SimilaritySearch (回顾)      |
|     |        |        |        |                           |
|     |        |        |        v                           |
|     |        |        |     Chat + Doctrine DBAL           |
|     |        |        |     (多轮对话查询)                 |
|     |        |        |                                    |
|     |        v        v                                    |
|     |     Claude (结构化提取)                               |
|     |        |                                             |
|     |        +---> MeetingMinutes DTO                      |
|     |        +---> MeetingAnalysis DTO                     |
|     |        +---> TTS 语音播报                            |
|     |                                                      |
|     +---> Gemini (原生音频分析，替代方案)                   |
|                                                            |
+------------------------------------------------------------+
```

---

## 存储方案对比

本教程涉及两类存储需求：向量存储（用于语义搜索）和对话存储（用于 Chat 历史）。以下是各方案的适用场景：

### 向量存储

| 存储方案 | 适用场景 | 优势 | 限制 |
|----------|----------|------|------|
| **InMemory Store** | 开发、测试、演示 | 零配置，即开即用 | 数据不持久，重启丢失 |
| **PostgreSQL pgvector** | 生产环境（本教程） | 成熟稳定，SQL 生态兼容 | 需安装 pgvector 扩展 |
| **MongoDB Store** | 文档型数据偏好 | 灵活 Schema，水平扩展 | 向量搜索性能不如专用引擎 |
| **Qdrant / Milvus** | 大规模向量检索 | 专为向量设计，性能最优 | 额外运维成本 |

### 对话存储

| 存储方案 | 适用场景 | 优势 | 限制 |
|----------|----------|------|------|
| **InMemory Store** | 开发、测试 | 零配置 | 数据不持久 |
| **Doctrine DBAL** | 生产环境（本教程） | 复用已有数据库，支持事务 | 高并发下性能一般 |
| **Redis Store** | 高并发会话场景 | 极快的读写速度 | 数据持久性依赖 Redis 配置 |
| **MongoDB Store** | 文档型偏好 | 灵活存储，自然支持 JSON | 需额外部署 MongoDB |

> **提示：** 在实际项目中，向量存储和对话存储可以使用不同的后端。例如用 PostgreSQL pgvector 存会议向量，用 Redis 存对话会话——根据各存储层的访问模式选择最合适的方案。

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `ElevenLabs scribe_v1` | 高质量多语言 STT 模型，支持多种音频格式 |
| `Audio::fromFile()` | 加载本地音频文件，自动检测 MIME 类型 |
| `TextSplitTransformer` | 按固定大小分块长文本，支持重叠以保留上下文 |
| `PlatformSubscriber` | 启用 StructuredOutput，将 DTO 类映射为 JSON Schema |
| `response_format` | 在 `invoke()` 选项中指定 DTO 类，配合 `asObject()` 获取类型化结果 |
| `Vectorizer` | 将文本/文档转换为向量嵌入，封装了平台调用细节 |
| `DocumentIndexer` | 编排分块、向量化、存储的完整索引流程 |
| `DocumentProcessor` | `DocumentIndexer` 的核心，管理 filter -> transform -> vectorize -> store 流水线 |
| `PostgreSQL pgvector` | 生产级向量数据库，支持 Cosine / L2 / InnerProduct 距离度量 |
| `Retriever` | 高级检索接口，封装了查询向量化 + 相似度搜索的完整流程 |
| `Agent` | 智能助手框架，支持输入处理器、输出处理器和工具调用 |
| `EmbeddingProvider` | 动态语义记忆，根据用户输入检索相关历史上下文 |
| `StaticMemoryProvider` | 固定背景知识，每次调用注入相同信息 |
| `MemoryInputProcessor` | Agent 输入处理器，遍历所有记忆 Provider 并注入上下文 |
| `SimilaritySearch` | Agent 工具，封装了向量相似度搜索功能 |
| `FaultTolerantToolbox` | 容错工具箱，工具调用失败时返回错误信息而非抛异常 |
| `Chat` | 对话管理器，维护多轮对话状态 |
| `DoctrineDbalMessageStore` | 基于 Doctrine DBAL 的对话持久化，适合关系数据库 |
| 多平台混用 | Embedding 用 OpenAI、生成用 Claude、音频用 Gemini——各取所长 |

---

## 下一步

如果你想用 AI 构建知识图谱并进行智能查询，请看 [35-knowledge-graph-construction.md](./35-knowledge-graph-construction.md)。
