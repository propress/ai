# YouTube 视频知识库

## 业务场景

你的团队通过 YouTube 频道做技术教育。频道有上百个视频教程，每个 20-60 分钟。学员经常问："哪个视频讲了 XXX？""第几分钟讲到了这个知识点？"现在你要做一个智能问答系统：自动获取视频字幕，建立知识库索引，学员可以用自然语言提问，AI 精准定位到具体视频和时间段。

**典型应用：** 视频课程智能搜索、会议录像知识库、播客内容检索、培训资料问答

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台 |
| **Agent Bridge YouTube** | 获取 YouTube 视频字幕 |
| **Store** | 字幕内容向量化存储 |
| **Agent** | 编排知识库构建和问答 |
| **Chat** | 对话持久化，支持多轮追问 |

## 前置准备

```bash
composer require symfony/ai-platform symfony/ai-platform-openai
composer require symfony/ai-agent
composer require symfony/ai-agent-youtube
composer require symfony/ai-store symfony/ai-store-pinecone
composer require symfony/ai-chat
```

---

## Step 1：获取视频字幕

使用 YouTube Bridge 自动获取视频字幕（转录文本）。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Bridge\Youtube\YoutubeTranscriber;
use Symfony\Component\HttpClient\HttpClient;

$transcriber = new YoutubeTranscriber(HttpClient::create());

// 获取单个视频的字幕
$transcript = $transcriber->transcribe('https://www.youtube.com/watch?v=dQw4w9WgXcQ');

echo "字幕内容（前 500 字）：\n";
echo mb_substr($transcript, 0, 500) . "...\n";
```

---

## Step 2：字幕分段与向量化索引

将长字幕按段落切分，每段关联视频 ID 和时间信息，存入向量数据库。

```php
<?php

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Store\Bridge\Pinecone\Store;
use Symfony\AI\Store\Document\Document;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\Transformer\ChunkTransformer;
use Symfony\AI\Store\Indexer;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());
$store = new Store(/* Pinecone 配置 */);

// 文档分块器：每 500 字一段，重叠 50 字
$chunker = new ChunkTransformer(maxLength: 500, overlap: 50);

$indexer = new Indexer($platform, 'text-embedding-3-small', $store);

// 视频列表（实际可从频道 API 获取）
$videos = [
    ['id' => 'abc123', 'title' => 'PHP 8.4 新特性详解', 'url' => 'https://youtube.com/watch?v=abc123'],
    ['id' => 'def456', 'title' => 'Symfony 7 快速上手', 'url' => 'https://youtube.com/watch?v=def456'],
    ['id' => 'ghi789', 'title' => 'AI Agent 开发实战', 'url' => 'https://youtube.com/watch?v=ghi789'],
];

foreach ($videos as $video) {
    echo "📹 处理视频：{$video['title']}...\n";

    // 获取字幕
    $transcript = $transcriber->transcribe($video['url']);

    // 分段
    $fullDoc = new Document(
        id: $video['id'],
        content: $transcript,
        metadata: new Metadata([
            'video_id' => $video['id'],
            'title' => $video['title'],
            'url' => $video['url'],
        ]),
    );

    $chunks = $chunker->transform([$fullDoc]);

    // 为每个分段添加视频元数据
    $indexDocuments = [];
    foreach ($chunks as $i => $chunk) {
        $indexDocuments[] = new Document(
            id: "{$video['id']}-chunk-{$i}",
            content: $chunk->content,
            metadata: new Metadata([
                'video_id' => $video['id'],
                'title' => $video['title'],
                'url' => $video['url'],
                'chunk_index' => $i,
            ]),
        );
    }

    $indexer->index($indexDocuments);
    echo "  ✅ 已索引 " . count($indexDocuments) . " 个段落\n";
}
```

---

## Step 3：智能视频问答

学员用自然语言提问，AI 从知识库中找到相关内容并回答。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Store\Retriever;

// 创建检索工具
$retriever = new Retriever($platform, 'text-embedding-3-small', $store);
$searchTool = new SimilaritySearch($retriever);

$toolbox = new Toolbox([$searchTool]);
$processor = new AgentProcessor($toolbox);

$agent = new Agent(
    $platform, 'gpt-4o-mini',
    [
        new SystemPromptInputProcessor(
            '你是视频课程助手。用户会问关于视频教程内容的问题。'
            . "\n使用 similarity_search 工具从视频字幕知识库中搜索相关内容。"
            . "\n回答时要：1) 指出来自哪个视频 2) 引用具体内容 3) 提供视频链接"
            . "\n如果知识库中找不到答案，诚实告知。"
        ),
        $processor,
    ],
    [$processor],
);

// 学员提问
$result = $agent->call(new MessageBag(
    Message::ofUser('PHP 8.4 有哪些新的数组函数？'),
));

echo "🎓 回答：\n" . $result->getContent() . "\n";
```

---

## Step 4：多轮对话支持

学员可以基于上一个问题继续追问。

```php
<?php

use Symfony\AI\Chat\ChatInterface;

// 持久化对话
/** @var ChatInterface $chat */
$conversationId = 'student-session-001';

// 第一轮
$messages = $chat->loadOrCreate($conversationId, new MessageBag(
    Message::ofUser('Symfony 7 的路由系统有什么变化？'),
));
$result = $agent->call($messages);
$chat->save($conversationId, $result->getMessages());
echo "Q1 回答：\n" . $result->getContent() . "\n\n";

// 第二轮追问
$messages = $chat->load($conversationId);
$messages->add(Message::ofUser('那控制器注解的写法呢？有代码示例吗？'));
$result = $agent->call($messages);
$chat->save($conversationId, $result->getMessages());
echo "Q2 回答：\n" . $result->getContent() . "\n";
```

---

## Step 5：视频推荐与学习路径

根据学员的问题历史，推荐相关视频。

```php
<?php

namespace App\Dto;

final class VideoRecommendation
{
    /**
     * @param string $videoTitle  视频标题
     * @param string $videoUrl    视频链接
     * @param string $reason      推荐理由
     * @param string $relevance   相关度（high/medium/low）
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

```php
<?php

use App\Dto\LearningPath;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;

// 根据学员的提问历史推荐
$messages = new MessageBag(
    Message::forSystem(
        '根据学员的学习历史和提问记录，推荐相关视频并制定学习路径。'
    ),
    Message::ofUser(
        "学员最近的提问：\n"
        . "1. PHP 8.4 新特性\n"
        . "2. Symfony 路由系统\n"
        . "3. 依赖注入怎么用\n\n"
        . "可用视频：\n"
        . "- PHP 8.4 新特性详解\n"
        . "- Symfony 7 快速上手\n"
        . "- AI Agent 开发实战\n"
        . "- PHP 设计模式\n"
        . "- 数据库优化实战\n\n"
        . '请推荐学习路径。'
    ),
);

$result = $platform->invoke('gpt-4o-mini', $messages, [
    'response_format' => LearningPath::class,
]);

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

## 完整流程

```
YouTube 视频库
    │
    ▼
[YoutubeTranscriber] → 获取字幕文本
    │
    ▼
[ChunkTransformer] → 分段（500 字/段）
    │
    ▼
[Indexer] → 嵌入 + 存入向量数据库
    │
    ▼
构建知识库 ✅
    │
    ├──► [学员提问] → Agent + SimilaritySearch → 精准回答
    │
    ├──► [多轮追问] → Chat 持久化上下文
    │
    └──► [学习推荐] → LearningPath 个性化路径
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `YoutubeTranscriber` | 自动获取 YouTube 视频字幕 |
| `ChunkTransformer` | 将长文本分段，控制每段大小和重叠 |
| `Indexer` | 文档嵌入 + 向量存储一步完成 |
| `SimilaritySearch` | Agent 工具，从向量库中语义检索 |
| `Retriever` | 向量检索器 |
| `Chat` 持久化 | 支持多轮追问，保持上下文 |
| 学习路径 | 结合提问历史，推荐视频和知识盲区 |

## 下一步

如果你需要自动抓取和分析竞品网站信息，请看 [27-competitor-intelligence-monitoring.md](./27-competitor-intelligence-monitoring.md)。
