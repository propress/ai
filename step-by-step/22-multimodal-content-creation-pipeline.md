# 多模态内容生产流水线

## 业务场景

你在构建一个内容营销平台。运营人员输入一个主题（如"新品发布"），AI 自动生成一整套可发布的内容物料：文案（中英双语）、配图、语音播报、短视频脚本。多个 AI 平台各司其职 — Claude 负责策划与文案、DALL-E 生成配图、ElevenLabs 制作语音、Gemini 做视觉质量审核。生成的所有内容自动建立向量索引，支持后续搜索和去重检测。

**典型应用：** 营销内容自动化、社交媒体批量发帖、多语言内容分发、播客/视频脚本生成、品牌素材库管理

> **💡 提示：** 本教程展示了 Symfony AI 最核心的多平台协作能力 — 不同的 AI 平台擅长不同任务，通过统一的 `PlatformInterface` 无缝切换。选择 **Anthropic Claude** 做文案是因为它在中文创意写作上表现出色；选择 **Gemini** 做视觉审核是因为它的多模态理解能力强。你可以根据实际需要替换任何平台，代码改动极小。

## 涉及模块

| 模块 | 包名 | 用途 |
|------|------|------|
| **Platform (Anthropic)** | `symfony/ai-platform-anthropic` | Claude 策划内容计划 + 生成文案 |
| **Platform (OpenAI)** | `symfony/ai-platform-openai` | DALL-E 3 生成营销配图 |
| **Platform (ElevenLabs)** | `symfony/ai-platform-elevenlabs` | 多语言文字转语音 |
| **Platform (Gemini)** | `symfony/ai-platform-google` | 视觉质量审核（图片+文案一致性） |
| **Platform (Cache)** | `symfony/ai-platform` | 缓存重复的生成请求 |
| **Platform (Failover)** | `symfony/ai-platform` | 多平台故障转移 |
| **StructuredOutput** | `symfony/ai-platform` | 结构化内容计划 DTO |
| **Store** | `symfony/ai-store` | 向量化索引生成内容，用于搜索/去重 |

## 项目流程图

```
用户输入主题
    │
    ▼
┌─────────────────────────────┐
│  Step 1: 初始化多平台环境      │
│  Anthropic + OpenAI +        │
│  ElevenLabs + Gemini         │
│  （Capability 检查）           │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│  Step 2: Claude 规划内容计划   │
│  StructuredOutput → DTO      │
└─────────────┬───────────────┘
              │
    ┌─────────┼─────────┬─────────────┐
    ▼         ▼         ▼             ▼
 [文案生成]  [图片生成]  [语音生成]   [脚本生成]
  Claude     DALL-E 3   ElevenLabs    Claude
  Streaming  Binary     Binary        Streaming
    │         │         │             │
    └────┬────┘─────────┘─────────────┘
         │
         ▼
┌─────────────────────────────┐
│  Step 5: Gemini 视觉质量审核   │
│  Image::fromFile() 多模态     │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│  Step 6: 向量化索引 (Store)   │
│  Vectorizer → 内容搜索/去重   │
└─────────────┬───────────────┘
              │
              ▼
        📦 内容包输出
   （文案 + 图片 + 音频 + 脚本）
```

## 前置准备

### 环境要求

- PHP 8.2+
- Composer 2.x
- 各平台 API 密钥（Anthropic、OpenAI、ElevenLabs、Google）

### 安装依赖

```bash
# 核心平台 + 四个 AI Bridge
composer require symfony/ai-platform \
    symfony/ai-platform-anthropic \
    symfony/ai-platform-openai \
    symfony/ai-platform-elevenlabs \
    symfony/ai-platform-google

# Store 模块（向量化索引）
composer require symfony/ai-store symfony/ai-store-mongodb

# Symfony 组件
composer require symfony/event-dispatcher symfony/http-client symfony/cache symfony/rate-limiter
```

### 设置 API 密钥

```bash
# .env
ANTHROPIC_API_KEY=sk-ant-xxxxx       # Claude 文案生成
OPENAI_API_KEY=sk-xxxxx              # DALL-E 图片生成
ELEVENLABS_API_KEY=xxxxx             # ElevenLabs 语音
GOOGLE_API_KEY=xxxxx                 # Gemini 视觉审核
MONGODB_URI=mongodb://localhost:27017 # 向量存储（可选）
```

> **🔒 安全建议：** 生产环境中，所有 API 密钥必须通过环境变量或 Vault 注入，绝不要硬编码在源代码中。`PlatformFactory::create()` 的 `$apiKey` 参数标记了 `#[\SensitiveParameter]`，确保密钥不会出现在异常堆栈中。

---

## Step 1：初始化多平台环境

内容流水线的第一步是初始化所有需要的 AI 平台。每个平台擅长不同类型的任务：

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory as AnthropicFactory;
use Symfony\AI\Platform\Bridge\ElevenLabs\PlatformFactory as ElevenLabsFactory;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory as GeminiFactory;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory as OpenAiFactory;
use Symfony\AI\Platform\Capability;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

// StructuredOutput 需要 EventDispatcher
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

$httpClient = HttpClient::create();

// Anthropic Claude — 策划+文案（中文创意写作表现出色）
$anthropic = AnthropicFactory::create(
    $_ENV['ANTHROPIC_API_KEY'],
    $httpClient,
    eventDispatcher: $dispatcher,
);

// OpenAI — DALL-E 3 图片生成
$openai = OpenAiFactory::create(
    $_ENV['OPENAI_API_KEY'],
    $httpClient,
);

// ElevenLabs — 高质量多语言语音合成
$elevenLabs = ElevenLabsFactory::create(
    apiKey: $_ENV['ELEVENLABS_API_KEY'],
    httpClient: $httpClient,
);

// Gemini — 视觉审核（多模态理解能力强）
$gemini = GeminiFactory::create(
    $_ENV['GOOGLE_API_KEY'],
    $httpClient,
);
```

### 使用 Capability 检查模型能力

在调用模型之前，可以通过 `Model::supports()` 验证模型是否具备所需能力。这在动态选择模型时非常有用：

```php
<?php

use Symfony\AI\Platform\Capability;

$catalog = $anthropic->getModelCatalog();
$claudeModel = $catalog->getModel('claude-sonnet-4-5-20250929');

// 验证 Claude 支持文本生成和结构化输出
if ($claudeModel->supports(Capability::OUTPUT_TEXT)
    && $claudeModel->supports(Capability::OUTPUT_STRUCTURED)) {
    echo "✅ Claude 支持文案生成 + 结构化输出\n";
}

$openaiCatalog = $openai->getModelCatalog();
$dalleModel = $openaiCatalog->getModel('dall-e-3');

if ($dalleModel->supports(Capability::OUTPUT_IMAGE)) {
    echo "✅ DALL-E 3 支持图片生成\n";
}

// 检查 ElevenLabs 是否支持 TTS
$elevenCatalog = $elevenLabs->getModelCatalog();
$ttsModel = $elevenCatalog->getModel('eleven_multilingual_v2');

if ($ttsModel->supports(Capability::TEXT_TO_SPEECH)) {
    echo "✅ ElevenLabs 支持文字转语音\n";
}
```

> **💡 提示：** `Capability` 枚举涵盖了所有 AI 能力类型：`INPUT_AUDIO`、`INPUT_IMAGE`、`INPUT_VIDEO`、`OUTPUT_TEXT`、`OUTPUT_IMAGE`、`OUTPUT_STREAMING`、`OUTPUT_STRUCTURED`、`TEXT_TO_SPEECH`、`SPEECH_TO_TEXT`、`EMBEDDINGS` 等。在运行时做能力检查可以避免调用不支持的功能时出现意外错误。

---

## Step 2：定义结构化内容计划

使用 `StructuredOutput` 让 AI 直接返回 PHP DTO 对象，而不是需要手动解析的自由文本。这是整个流水线的核心 — AI 输出的计划结构决定了后续每个步骤的执行方式。

### 定义 DTO

```php
<?php

namespace App\Dto;

final class ContentPiece
{
    /**
     * @param string $type       内容类型：text|image|audio|video_script
     * @param string $title      内容标题
     * @param string $prompt     生成提示词（给下游模型使用）
     * @param string $channel    发布渠道：wechat|weibo|xiaohongshu|twitter|linkedin
     * @param string $language   语言：zh|en
     */
    public function __construct(
        public readonly string $type,
        public readonly string $title,
        public readonly string $prompt,
        public readonly string $channel,
        public readonly string $language = 'zh',
    ) {
    }
}

final class ContentPlan
{
    /**
     * @param string         $theme          内容主题
     * @param string         $targetAudience 目标受众
     * @param string         $brandTone      品牌调性：professional|casual|playful|luxury
     * @param ContentPiece[] $pieces         内容列表
     * @param string         $coreMessage    核心信息（一句话总结）
     * @param string         $englishSummary 英文摘要（用于海外渠道）
     */
    public function __construct(
        public readonly string $theme,
        public readonly string $targetAudience,
        public readonly string $brandTone,
        public readonly array $pieces,
        public readonly string $coreMessage,
        public readonly string $englishSummary,
    ) {
    }
}
```

### 用 Claude 生成内容计划

选择 Anthropic Claude 做策划，因为它在长文本创意规划上表现出色，并且原生支持结构化输出：

```php
<?php

use App\Dto\ContentPlan;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$messages = new MessageBag(
    Message::forSystem(
        '你是资深内容营销策划总监，擅长制定跨渠道内容分发策略。'
        . '根据用户输入的主题，生成完整的内容生产计划。'
        . "\n\n要求："
        . "\n- 微信公众号：深度长文，专业调性"
        . "\n- 微博：短文 + 话题标签，适合传播"
        . "\n- 小红书：种草风格，图文并茂"
        . "\n- Twitter/LinkedIn：英文版本，面向海外受众"
        . "\n- 每个渠道至少 1 条文案 + 1 张配图"
        . "\n- 另外规划 1 条语音播报和 1 个短视频脚本"
    ),
    Message::ofUser('主题：CloudFlow 2.0 正式发布 — 全新 AI 协作功能，让团队效率提升 300%'),
);

$result = $anthropic->invoke('claude-sonnet-4-5-20250929', $messages, [
    'response_format' => ContentPlan::class,
]);

/** @var ContentPlan $plan */
$plan = $result->asObject();

echo "=== 内容生产计划 ===\n";
echo "主题：{$plan->theme}\n";
echo "受众：{$plan->targetAudience}\n";
echo "调性：{$plan->brandTone}\n";
echo "核心信息：{$plan->coreMessage}\n";
echo "English：{$plan->englishSummary}\n\n";

echo "内容列表（共 " . count($plan->pieces) . " 项）：\n";
foreach ($plan->pieces as $i => $piece) {
    echo sprintf(
        "  %d. [%s] %s → %s (%s)\n",
        $i + 1,
        $piece->type,
        $piece->title,
        $piece->channel,
        $piece->language,
    );
}
```

> **💡 提示：** `StructuredOutput` 的工作原理是通过 `PlatformSubscriber` 在请求前把 DTO 类的属性结构转为 JSON Schema，注入到请求参数中。AI 返回 JSON 后，自动反序列化为 PHP 对象。`response_format` 选项接受任何 PHP 类名，只要构造函数参数带有 `@param` 类型注解。

---

## Step 3：流式生成文案内容

文案是内容包的核心。对于长篇内容（如公众号文章），使用 **Streaming** 可以边生成边显示进度，提升用户体验：

```php
<?php

use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$outputDir = '/tmp/content-pack-' . date('Ymd-His');
if (!is_dir($outputDir)) {
    mkdir($outputDir, 0755, true);
}

$textPieces = array_filter($plan->pieces, fn ($p) => 'text' === $p->type);

foreach ($textPieces as $piece) {
    echo "\n⏳ 生成文案：{$piece->title} ({$piece->channel})...\n";

    $messages = new MessageBag(
        Message::forSystem(
            "你是资深内容文案。品牌：CloudFlow。渠道：{$piece->channel}。"
            . "调性：{$plan->brandTone}。语言：" . ('en' === $piece->language ? '英文' : '中文')
            . "\n核心信息：{$plan->coreMessage}"
        ),
        Message::ofUser($piece->prompt),
    );

    // 使用 stream 选项启用流式输出
    $result = $anthropic->invoke('claude-sonnet-4-5-20250929', $messages, [
        'stream' => true,
    ]);

    // 流式接收并写入文件
    $content = '';
    $charCount = 0;
    foreach ($result->asStream() as $chunk) {
        $content .= $chunk;
        $charCount += mb_strlen($chunk);

        // 每 100 字符显示一次进度
        if (0 === $charCount % 100) {
            echo "  📝 已生成 {$charCount} 字...\r";
        }
    }

    $filename = sprintf('%s/%s-%s.md', $outputDir, $piece->channel, $piece->language);
    file_put_contents($filename, $content);
    echo "  ✅ 文案已保存：{$filename}（共 {$charCount} 字）\n";
}
```

> **🏭 生产建议：** 流式输出（Streaming）适合用户界面实时展示生成进度的场景。对于批量后台任务，非流式模式更简单高效。`'stream' => true` 选项让 `asStream()` 返回一个 `\Generator`，逐块产出文本。

### 视频脚本生成

视频脚本需要特殊格式，包含分镜描述、旁白文字和时长标注：

```php
<?php

$scriptPieces = array_filter($plan->pieces, fn ($p) => 'video_script' === $p->type);

foreach ($scriptPieces as $piece) {
    echo "\n⏳ 生成视频脚本：{$piece->title}...\n";

    $messages = new MessageBag(
        Message::forSystem(
            '你是短视频脚本编剧。输出严格按以下格式：'
            . "\n[镜头1] 画面描述 | 旁白文字 | 时长（秒）"
            . "\n[镜头2] 画面描述 | 旁白文字 | 时长（秒）"
            . "\n..."
            . "\n\n总时长控制在 60-90 秒。"
        ),
        Message::ofUser($piece->prompt),
    );

    $result = $anthropic->invoke('claude-sonnet-4-5-20250929', $messages);

    $filename = "{$outputDir}/video-script.md";
    file_put_contents($filename, $result->asText());
    echo "  ✅ 脚本已保存：{$filename}\n";
}
```

---

## Step 4：生成配图与语音

### DALL-E 3 图片生成

DALL-E 3 接受纯文本提示词作为输入，返回 `ImageResult` 对象。注意它和 Chat 模型的调用方式不同 — 输入是字符串而非 `MessageBag`：

```php
<?php

use Symfony\AI\Platform\Bridge\OpenAi\DallE\ImageResult;

$imagePieces = array_filter($plan->pieces, fn ($p) => 'image' === $p->type);

foreach ($imagePieces as $piece) {
    echo "\n⏳ 生成配图：{$piece->title}...\n";

    $deferredResult = $openai->invoke('dall-e-3', $piece->prompt, [
        'size' => '1024x1024',
        'quality' => 'hd',
        'response_format' => 'b64_json',
    ]);

    /** @var ImageResult $imageResult */
    $imageResult = $deferredResult->getResult();
    $images = $imageResult->getContent();

    // 保存图片二进制数据
    $imageData = base64_decode($images[0]->encodedImage);
    $filename = sprintf('%s/%s-image.png', $outputDir, $piece->channel);
    file_put_contents($filename, $imageData);

    echo "  ✅ 图片已保存：{$filename}\n";

    if (null !== $imageResult->revisedPrompt) {
        echo "  📋 DALL-E 优化后的提示词：{$imageResult->revisedPrompt}\n";
    }
}
```

> **💡 提示：** DALL-E 3 会自动优化你的提示词（`revisedPrompt`），生成更精确的图片。`response_format` 支持 `'b64_json'`（返回 Base64 编码图片数据）和 `'url'`（返回临时下载链接，1 小时后过期）。生产环境建议用 `'b64_json'` 直接获取数据，避免链接过期问题。

### ElevenLabs 语音合成

ElevenLabs 的调用方式也与 Chat 模型不同 — 输入是 `Text` 内容对象，返回二进制音频数据：

```php
<?php

use Symfony\AI\Platform\Message\Content\Text;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$audioPieces = array_filter($plan->pieces, fn ($p) => 'audio' === $p->type);

foreach ($audioPieces as $piece) {
    echo "\n⏳ 生成语音：{$piece->title}...\n";

    // 第一步：用 Claude 生成适合朗读的播报稿
    $scriptResult = $anthropic->invoke('claude-sonnet-4-5-20250929', new MessageBag(
        Message::forSystem(
            '你是专业播客主持人。将营销内容改写为适合朗读的播报稿。'
            . '语气自然、节奏流畅、有停顿标记。控制在 1-2 分钟朗读时长。'
        ),
        Message::ofUser($piece->prompt),
    ));

    $script = $scriptResult->asText();
    file_put_contents("{$outputDir}/audio-script.md", $script);
    echo "  📝 播报稿已保存\n";

    // 第二步：用 ElevenLabs 转为语音
    $audioResult = $elevenLabs->invoke(
        'eleven_multilingual_v2',
        new Text($script),
        ['voice' => 'pqHfZKP75CvOlQylNhV4'],  // Bill 语音
    );

    $filename = "{$outputDir}/audio-broadcast.mp3";
    file_put_contents($filename, $audioResult->asBinary());
    echo "  ✅ 语音已保存：{$filename}\n";
}
```

> **⚠️ 注意：** ElevenLabs 的 `invoke()` 输入是 `new Text($text)` 对象，不是 `MessageBag`。`voice` 参数是 ElevenLabs 平台的语音 ID，可以在 ElevenLabs 控制台查看可用语音列表。`eleven_multilingual_v2` 模型支持中英文等多语言。

---

## Step 5：Gemini 视觉质量审核

生成完所有素材后，利用 Gemini 的多模态能力对配图进行质量审核。Gemini 可以同时分析多张图片，判断是否符合品牌调性：

```php
<?php

use Symfony\AI\Platform\Message\Content\Image;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 收集所有生成的图片
$imageFiles = glob("{$outputDir}/*-image.png");

if ([] === $imageFiles) {
    echo "⚠️ 没有找到需要审核的图片\n";
} else {
    $images = array_map(fn (string $path) => Image::fromFile($path), $imageFiles);

    // 同时读取所有文案用于对比审核
    $textFiles = glob("{$outputDir}/*.md");
    $textContents = '';
    foreach ($textFiles as $file) {
        $textContents .= basename($file) . ":\n" . file_get_contents($file) . "\n\n";
    }

    $messages = new MessageBag(
        Message::forSystem(
            '你是品牌视觉审核经理。请严格检查以下标准：'
            . "\n1. 图片质量：清晰度、构图、色彩是否专业"
            . "\n2. 品牌一致性：是否符合科技产品的专业形象"
            . "\n3. 图文匹配：图片与对应文案的内容是否一致"
            . "\n4. 合规检查：有无不当、敏感、侵权内容"
            . "\n\n对每张图片给出 1-10 分评分和具体改进建议。"
        ),
        Message::ofUser(
            "品牌：CloudFlow\n核心信息：{$plan->coreMessage}\n\n"
            . "以下是各渠道文案内容：\n{$textContents}\n"
            . '请审核以下生成的配图：',
            ...$images,
        ),
    );

    $result = $gemini->invoke('gemini-2.5-flash', $messages);
    $review = $result->asText();

    file_put_contents("{$outputDir}/quality-review.md", $review);
    echo "\n=== 质量审核报告 ===\n" . $review . "\n";
}
```

> **💡 提示：** `Image::fromFile()` 使用延迟加载（lazy loading）— 文件内容在实际发送请求时才读取，不会提前占用内存。Gemini 支持在单条消息中传入多张图片，使用展开运算符 `...$images` 将图片数组合并到消息内容中。

---

## Step 6：向量化索引生成的内容

将所有生成的文案内容建立向量索引，可以实现两个关键功能：（1）后续通过自然语言搜索查找历史内容；（2）在生成新内容前检测相似度，避免重复。

### 配置 Vectorizer 和 Store

```php
<?php

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory as OpenAiFactory;
use Symfony\AI\Store\Bridge\MongoDb\Store;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\Component\HttpClient\HttpClient;

// 使用 OpenAI 嵌入模型进行向量化
$embeddingPlatform = OpenAiFactory::create(
    $_ENV['OPENAI_API_KEY'],
    HttpClient::create(),
);

$vectorizer = new Vectorizer(
    $embeddingPlatform,
    'text-embedding-3-small',  // 1536 维度，性价比最高
);

// 使用 MongoDB 作为向量存储
$mongoClient = new \MongoDB\Client($_ENV['MONGODB_URI']);
$store = new Store(
    client: $mongoClient,
    databaseName: 'content_platform',
    collectionName: 'generated_content',
    indexName: 'content_vector_index',
    embeddingsDimension: 1536,
);

// 首次运行时创建索引
$store->setup();
```

### 索引生成的内容

```php
<?php

use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\VectorDocument;

$textFiles = glob("{$outputDir}/*.md");
$documents = [];

foreach ($textFiles as $file) {
    $content = file_get_contents($file);
    $basename = basename($file, '.md');

    // 将文案内容向量化
    $vector = $vectorizer->vectorize($content);

    $documents[] = new VectorDocument(
        id: $basename . '-' . date('Ymd'),
        vector: $vector,
        metadata: new Metadata([
            'theme' => $plan->theme,
            'channel' => $basename,
            'content' => mb_substr($content, 0, 500),  // 存储内容摘要
            'created_at' => date('c'),
            'brand_tone' => $plan->brandTone,
        ]),
    );
}

// 批量写入向量存储
$store->add($documents);
echo "\n📊 已索引 " . count($documents) . " 篇内容到向量数据库\n";
```

### 相似内容检测（去重）

在生成新内容之前，可以先检查是否已有高度相似的历史内容：

```php
<?php

use Symfony\AI\Store\Query\TextQuery;

/**
 * 检查是否存在相似内容，用于去重。
 *
 * @param string $newContent  待检测的新内容
 * @param float  $threshold   相似度阈值（0-1，越高越严格）
 *
 * @return array{is_duplicate: bool, similar_content: list<array{id: string|int, score: float|null}>}
 */
function checkDuplicate(
    Store $store,
    Vectorizer $vectorizer,
    string $newContent,
    float $threshold = 0.92,
): array {
    $vector = $vectorizer->vectorize($newContent);

    $query = new TextQuery($newContent);
    $results = $store->query($query, ['limit' => 3]);

    $similar = [];
    $isDuplicate = false;

    foreach ($results as $doc) {
        if (null !== $doc->getScore() && $doc->getScore() >= $threshold) {
            $isDuplicate = true;
        }
        $similar[] = [
            'id' => $doc->getId(),
            'score' => $doc->getScore(),
        ];
    }

    return [
        'is_duplicate' => $isDuplicate,
        'similar_content' => $similar,
    ];
}

// 使用示例
$duplicateCheck = checkDuplicate(
    $store,
    $vectorizer,
    '关于 CloudFlow 2.0 AI 协作功能的介绍文案...',
);

if ($duplicateCheck['is_duplicate']) {
    echo "⚠️ 检测到高度相似的历史内容，建议调整主题角度\n";
} else {
    echo "✅ 未发现重复内容，可以继续生成\n";
}
```

> **🏭 生产建议：** 向量索引不仅用于去重，还可以构建内部素材搜索引擎 — 运营团队通过自然语言查询历史内容，比如"找一篇关于 AI 协作的微信文章"。Store 模块支持 MongoDB、PostgreSQL、Elasticsearch 等 24+ 后端，按需切换。

---

## Step 7：异常处理与生产加固

### 细粒度异常处理

Symfony AI 提供了丰富的异常类型，让你可以针对不同错误采取不同策略：

```php
<?php

use Symfony\AI\Platform\Exception\AuthenticationException;
use Symfony\AI\Platform\Exception\BadRequestException;
use Symfony\AI\Platform\Exception\ContentFilterException;
use Symfony\AI\Platform\Exception\RateLimitExceededException;
use Symfony\AI\Platform\Exception\RuntimeException;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

/**
 * 带重试和异常处理的内容生成。
 *
 * @param array{max_retries?: int, retry_delay?: int} $options
 */
function safeGenerate(
    object $platform,
    string $model,
    MessageBag $messages,
    array $options = [],
): string {
    $maxRetries = $options['max_retries'] ?? 3;
    $retryDelay = $options['retry_delay'] ?? 2;
    unset($options['max_retries'], $options['retry_delay']);

    for ($attempt = 1; $attempt <= $maxRetries; ++$attempt) {
        try {
            $result = $platform->invoke($model, $messages, $options);

            return $result->asText();
        } catch (RateLimitExceededException $e) {
            // 触发限流：指数退避重试
            $waitSeconds = $retryDelay * (2 ** ($attempt - 1));
            echo "⏳ 触发限流，等待 {$waitSeconds} 秒后重试（{$attempt}/{$maxRetries}）\n";
            sleep($waitSeconds);
        } catch (ContentFilterException $e) {
            // 内容被过滤：调整提示词后重试
            echo "⚠️ 内容被安全过滤器拦截，尝试调整提示词\n";
            $messages = new MessageBag(
                Message::forSystem('请确保内容积极、专业、符合商业规范。'),
                ...$messages->getMessages(),
            );
        } catch (AuthenticationException $e) {
            // 认证失败：无法恢复，立即退出
            throw $e;
        } catch (BadRequestException $e) {
            echo "❌ 请求参数错误：{$e->getMessage()}\n";
            throw $e;
        }
    }

    throw new RuntimeException("内容生成失败：已重试 {$maxRetries} 次");
}
```

### Cache Bridge — 缓存重复请求

相同的内容计划可能多次生成（如预览、修改后重新生成），使用 `CachePlatform` 避免重复调用 API：

```php
<?php

use Symfony\AI\Platform\Bridge\Cache\CachePlatform;
use Symfony\Component\Cache\Adapter\FilesystemTagAwareAdapter;

$cache = new FilesystemTagAwareAdapter(
    namespace: 'content_generation',
    defaultLifetime: 3600,  // 缓存 1 小时
);

$cachedAnthropic = new CachePlatform(
    platform: $anthropic,
    cache: $cache,
    cacheTtl: 3600,
);

// 相同输入会命中缓存，不重复调用 API
$result = $cachedAnthropic->invoke('claude-sonnet-4-5-20250929', $messages, [
    'response_format' => ContentPlan::class,
]);
```

> **💡 提示：** `CachePlatform` 自动根据模型、输入内容和选项参数生成缓存 Key。它对 `StructuredOutput` 和普通文本请求都生效。适合缓存内容计划、翻译结果等幂等性操作，不适合缓存需要多样性的创意生成。

### Failover Bridge — 平台故障转移

在生产环境中，单一 AI 平台可能出现临时故障。`FailoverPlatform` 可以在多个平台之间自动切换：

```php
<?php

use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory as AnthropicFactory;
use Symfony\AI\Platform\Bridge\Failover\FailoverPlatformFactory;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory as OpenAiFactory;
use Symfony\Component\HttpClient\HttpClient;
use Symfony\Component\RateLimiter\RateLimiterFactory;
use Symfony\Component\RateLimiter\Storage\InMemoryStorage;

$httpClient = HttpClient::create();

// 主力：Anthropic Claude
$primaryPlatform = AnthropicFactory::create(
    $_ENV['ANTHROPIC_API_KEY'],
    $httpClient,
    eventDispatcher: $dispatcher,
);

// 备用：OpenAI GPT-4o
$fallbackPlatform = OpenAiFactory::create(
    $_ENV['OPENAI_API_KEY'],
    $httpClient,
    eventDispatcher: $dispatcher,
);

$rateLimiterFactory = new RateLimiterFactory(
    ['id' => 'content_failover', 'policy' => 'sliding_window', 'limit' => 10, 'interval' => '1 minute'],
    new InMemoryStorage(),
);

$resilientPlatform = FailoverPlatformFactory::create(
    platforms: [$primaryPlatform, $fallbackPlatform],
    rateLimiterFactory: $rateLimiterFactory,
);

// 自动故障转移：Claude 失败会自动切换到 GPT-4o
$result = $resilientPlatform->invoke('claude-sonnet-4-5-20250929', $messages);
```

> **⚠️ 注意：** Failover 在平台之间切换时，需要确保备用平台也支持相同的模型功能。如果主力用的是 Claude 独有模型，备用平台调用时会自动使用其默认模型。建议主备平台使用相同能力等级的模型。

---

## 完整示例

以下是完整的可运行示例，整合了前面所有步骤：

```php
<?php

require 'vendor/autoload.php';

use App\Dto\ContentPlan;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory as AnthropicFactory;
use Symfony\AI\Platform\Bridge\ElevenLabs\PlatformFactory as ElevenLabsFactory;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory as GeminiFactory;
use Symfony\AI\Platform\Bridge\OpenAi\DallE\ImageResult;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory as OpenAiFactory;
use Symfony\AI\Platform\Exception\ContentFilterException;
use Symfony\AI\Platform\Exception\RateLimitExceededException;
use Symfony\AI\Platform\Message\Content\Image;
use Symfony\AI\Platform\Message\Content\Text;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

// === 1. 初始化平台 ===

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$httpClient = HttpClient::create();

$anthropic = AnthropicFactory::create(
    $_ENV['ANTHROPIC_API_KEY'], $httpClient, eventDispatcher: $dispatcher,
);
$openai = OpenAiFactory::create($_ENV['OPENAI_API_KEY'], $httpClient);
$elevenLabs = ElevenLabsFactory::create(
    apiKey: $_ENV['ELEVENLABS_API_KEY'], httpClient: $httpClient,
);
$gemini = GeminiFactory::create($_ENV['GOOGLE_API_KEY'], $httpClient);

// === 2. 生成内容计划 ===

$topic = $argv[1] ?? 'CloudFlow 2.0 发布 — AI 协作功能让团队效率提升 300%';
echo "🚀 开始为主题生成内容包：{$topic}\n\n";

$planMessages = new MessageBag(
    Message::forSystem(
        '你是资深内容营销策划总监。根据主题制定跨渠道内容计划。'
        . "\n包含：微信长文、微博短文、小红书种草、英文 Twitter"
        . "\n配图 2 张、语音播报 1 条、短视频脚本 1 个"
    ),
    Message::ofUser("主题：{$topic}"),
);

$plan = $anthropic->invoke('claude-sonnet-4-5-20250929', $planMessages, [
    'response_format' => ContentPlan::class,
])->asObject();

echo "📋 内容计划：{$plan->theme}（{$plan->targetAudience}）\n";
echo "   共 " . count($plan->pieces) . " 项内容任务\n\n";

// === 3. 执行内容生成 ===

$outputDir = '/tmp/content-pack-' . date('Ymd-His');
mkdir($outputDir, 0755, true);

$generatedFiles = [];

foreach ($plan->pieces as $piece) {
    echo "⏳ [{$piece->type}] {$piece->title}...\n";

    try {
        switch ($piece->type) {
            case 'text':
            case 'video_script':
                $result = $anthropic->invoke(
                    'claude-sonnet-4-5-20250929',
                    new MessageBag(
                        Message::forSystem(
                            "渠道：{$piece->channel}。品牌：CloudFlow。调性：{$plan->brandTone}。"
                        ),
                        Message::ofUser($piece->prompt),
                    ),
                );
                $filename = sprintf('%s/%s-%s.md', $outputDir, $piece->channel, $piece->type);
                file_put_contents($filename, $result->asText());
                $generatedFiles[] = $filename;
                echo "   ✅ 已保存：{$filename}\n";
                break;

            case 'image':
                $deferredResult = $openai->invoke('dall-e-3', $piece->prompt, [
                    'size' => '1024x1024',
                    'quality' => 'hd',
                    'response_format' => 'b64_json',
                ]);
                /** @var ImageResult $imageResult */
                $imageResult = $deferredResult->getResult();
                $imageData = base64_decode($imageResult->getContent()[0]->encodedImage);
                $filename = sprintf('%s/%s-image.png', $outputDir, $piece->channel);
                file_put_contents($filename, $imageData);
                $generatedFiles[] = $filename;
                echo "   ✅ 已保存：{$filename}\n";
                break;

            case 'audio':
                // 先生成播报稿
                $script = $anthropic->invoke(
                    'claude-sonnet-4-5-20250929',
                    new MessageBag(
                        Message::forSystem('将营销信息改写为适合朗读的播报稿，控制在 1-2 分钟。'),
                        Message::ofUser($piece->prompt),
                    ),
                )->asText();

                $scriptFile = "{$outputDir}/audio-script.md";
                file_put_contents($scriptFile, $script);
                $generatedFiles[] = $scriptFile;

                // 再转语音
                $audioResult = $elevenLabs->invoke(
                    'eleven_multilingual_v2',
                    new Text($script),
                    ['voice' => 'pqHfZKP75CvOlQylNhV4'],
                );
                $filename = "{$outputDir}/audio-broadcast.mp3";
                file_put_contents($filename, $audioResult->asBinary());
                $generatedFiles[] = $filename;
                echo "   ✅ 已保存：{$filename}\n";
                break;

            default:
                echo "   ⚠️ 未知类型：{$piece->type}\n";
        }
    } catch (RateLimitExceededException $e) {
        echo "   ⏳ 触发限流，等待 5 秒...\n";
        sleep(5);
    } catch (ContentFilterException $e) {
        echo "   ⚠️ 内容被安全过滤，跳过：{$piece->title}\n";
    }
}

// === 4. 质量审核 ===

$imageFiles = glob("{$outputDir}/*-image.png");
if ([] !== $imageFiles) {
    echo "\n🔍 正在进行质量审核...\n";

    $images = array_map(fn (string $path) => Image::fromFile($path), $imageFiles);
    $reviewResult = $gemini->invoke('gemini-2.5-flash', new MessageBag(
        Message::forSystem(
            '审核营销配图质量：品牌一致性、专业度、合规性。每张图给出 1-10 分。'
        ),
        Message::ofUser(
            "品牌：CloudFlow\n核心信息：{$plan->coreMessage}\n审核以下配图：",
            ...$images,
        ),
    ));

    file_put_contents("{$outputDir}/quality-review.md", $reviewResult->asText());
    echo "   ✅ 审核报告已保存\n";
}

// === 5. 输出汇总 ===

echo "\n" . str_repeat('=', 50) . "\n";
echo "📦 内容包已生成：{$outputDir}/\n";
echo "   共 " . count($generatedFiles) . " 个文件\n";
foreach ($generatedFiles as $file) {
    $size = filesize($file);
    echo sprintf("   - %s (%s)\n", basename($file), $size > 1024 ? round($size / 1024) . ' KB' : $size . ' B');
}
```

---

## 其他实现方案

### 方案一：使用 Agent 自动编排

如果想让 AI 自主决定生成顺序和策略，可以用 Agent + Tool 模式替代手动编排：

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Attribute\AsTool;
use Symfony\AI\Agent\Toolbox\Toolbox;

#[AsTool('generate_text', description: '生成指定渠道的营销文案', method: '__invoke')]
final class TextGeneratorTool
{
    public function __construct(private readonly object $platform)
    {
    }

    /**
     * @param string $prompt  文案生成提示
     * @param string $channel 发布渠道
     */
    public function __invoke(string $prompt, string $channel): string
    {
        return $this->platform->invoke(
            'claude-sonnet-4-5-20250929',
            new MessageBag(
                Message::forSystem("渠道：{$channel}。品牌：CloudFlow。"),
                Message::ofUser($prompt),
            ),
        )->asText();
    }
}

#[AsTool('generate_image', description: '根据描述生成营销配图，返回文件路径', method: '__invoke')]
final class ImageGeneratorTool
{
    public function __construct(private readonly object $platform)
    {
    }

    public function __invoke(string $prompt): string
    {
        $deferredResult = $this->platform->invoke('dall-e-3', $prompt, [
            'size' => '1024x1024',
            'response_format' => 'b64_json',
        ]);

        $imageResult = $deferredResult->getResult();
        $imageData = base64_decode($imageResult->getContent()[0]->encodedImage);

        $path = '/tmp/generated-' . uniqid() . '.png';
        file_put_contents($path, $imageData);

        return "图片已保存到：{$path}";
    }
}

$toolbox = new Toolbox([
    new TextGeneratorTool($anthropic),
    new ImageGeneratorTool($openai),
]);

$processor = new AgentProcessor($toolbox);

$agent = new Agent(
    $anthropic,
    'claude-sonnet-4-5-20250929',
    [$processor, new SystemPromptInputProcessor(
        '你是内容生产总监。根据用户的主题需求，使用工具生成内容物料。'
    )],
    [$processor],
);

$result = $agent->call(new MessageBag(
    Message::ofUser('生成一篇微信公众号文章 + 配图，主题：CloudFlow 2.0 发布'),
));

echo $result->getContent() . "\n";
```

### 方案二：全部使用 OpenAI 生态

如果想减少 API 密钥管理的复杂度，可以全部使用 OpenAI 生态：

```php
<?php

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Text;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

$platform = PlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);

// 文案：GPT-4o
$text = $platform->invoke('gpt-4o', $messages)->asText();

// 图片：DALL-E 3（同一平台实例）
$image = $platform->invoke('dall-e-3', 'prompt text', [
    'response_format' => 'b64_json',
]);

// 语音：OpenAI TTS（同一平台实例）
$audio = $platform->invoke('tts-1-hd', new Text('播报内容'), [
    'voice' => 'alloy',
]);
```

> **💡 提示：** 全 OpenAI 方案更简单，但 ElevenLabs 的语音质量通常优于 OpenAI TTS，Claude 在长文本中文创作上也有优势。生产环境建议根据实际效果选择最合适的平台组合。

---

## 生产环境建议

### 性能优化

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| **CachePlatform** | 缓存内容计划等幂等请求 | 预览、重复生成 |
| **Streaming** | 长文案边生成边展示 | 用户界面实时反馈 |
| **并行生成** | 文案、图片、语音可并行执行 | 批量内容生产 |
| **异步队列** | Symfony Messenger 异步处理 | 高并发场景 |

### 成本控制

| 模型 | 用途 | 成本等级 | 优化策略 |
|------|------|---------|---------|
| Claude Sonnet 4.5 | 文案生成 | 中 | 短文案可用 Claude Haiku 4.5 降本 |
| DALL-E 3 HD | 配图生成 | 高 | 非核心图片用 standard 质量 |
| ElevenLabs v2 | 语音合成 | 中 | 控制播报稿长度 |
| Gemini 2.5 Flash | 视觉审核 | 低 | Flash 模型成本极低 |
| text-embedding-3-small | 向量化 | 极低 | 批量调用降低单次开销 |

### 安全与合规

> **🔒 安全建议：**
> - 所有生成的内容在发布前必须经过人工审核终审
> - 使用 `ContentFilterException` 捕获被安全策略拦截的内容
> - DALL-E 生成的图片可能包含水印，需确认商用授权
> - 存储用户主题输入时注意 PII（个人身份信息）脱敏
> - 向量数据库中存储的内容摘要不应包含敏感商业信息

### 架构建议

```
用户提交主题
       │
       ▼
[API Gateway] → [消息队列 (Messenger)]
                       │
           ┌───────────┼───────────┐
           ▼           ▼           ▼
      [文案 Worker]  [图片 Worker] [语音 Worker]
       Anthropic      OpenAI       ElevenLabs
           │           │           │
           └───────────┼───────────┘
                       ▼
              [审核 Worker] → Gemini
                       │
                       ▼
              [向量索引 Worker]
                       │
                       ▼
              [通知用户完成]
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `PlatformFactory::create()` | 每个 Bridge 都有工厂方法，统一的创建接口 |
| `Capability` 枚举 | `Model::supports(Capability::OUTPUT_STRUCTURED)` 运行时检查模型能力 |
| `StructuredOutput` | `PlatformSubscriber` + `response_format => DTO::class`，AI 直接返回 PHP 对象 |
| `StreamResult` | `'stream' => true` + `$result->asStream()` 逐块接收长文本 |
| `ImageResult` | DALL-E 返回 `Base64Image` 或 `UrlImage`，需通过 `getResult()` 获取 |
| `Text` 内容对象 | ElevenLabs TTS 的输入是 `new Text($script)`，不是 `MessageBag` |
| `Image::fromFile()` | 延迟加载图片，Gemini 审核时可传入多张图片 |
| `CachePlatform` | 包装任意平台，缓存幂等请求，减少 API 调用 |
| `FailoverPlatformFactory` | 多平台故障转移，配合 RateLimiter 实现高可用 |
| `Vectorizer` | `PlatformInterface` + 嵌入模型 → 文本向量化 |
| `VectorDocument` | 向量 + 元数据，存入 Store 后支持自然语言搜索 |
| `Store::query()` | 基于向量相似度查询，可用于内容去重检测 |
| 异常层次 | `RateLimitExceededException` 重试、`ContentFilterException` 调整内容、`AuthenticationException` 立即失败 |

## 下一步

如果你想用 PHP 搭建一个类似 CrewAI 的协作智能体团队系统，请看 [23-crewai-style-agent-team.md](./23-crewai-style-agent-team.md)。
