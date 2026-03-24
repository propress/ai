# 视频内容分析与总结

## 业务场景

你在做一个短视频/在线教育平台。需要 AI 自动分析上传的视频：提取关键画面、总结视频主题、生成文字摘要、标注关键时间点。用于自动审核视频内容，或为视频生成字幕和介绍文案。

**典型应用：** 视频内容审核、教学视频摘要、短视频标签生成、会议录像提取要点

> **💡 提示：** 本教程使用 **Gemini** 作为主要平台，因为它是目前对视频理解能力最强的 AI 平台，原生支持视频文件输入。同时提供 **VertexAI** 作为企业级替代，以及 **ffmpeg 关键帧提取**方案覆盖其他平台。

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台（Gemini 原生支持视频） |
| **Video Content** | `Video::fromFile()` 视频文件输入 |
| **Audio Content** | 视频音轨单独分析（非 Gemini 平台回退方案） |
| **StructuredOutput** | 结构化的视频分析报告（`VideoAnalysis` DTO） |

## 平台选择说明

| 平台 | 视频支持 | 推荐场景 |
|------|---------|---------|
| **Gemini** | ✅ 原生支持 `Video::fromFile()` | 首选，直接分析视频画面+音频 |
| **VertexAI** | ✅ 原生支持（同 Gemini 模型） | 企业级部署，需要 GCP 项目权限 |
| **OpenAI / Anthropic** | ❌ 不直接支持 | 用 ffmpeg 提取关键帧 + 音频间接处理 |

## 前置准备

```bash
# 推荐：Gemini（最佳视频支持）
composer require symfony/ai-platform symfony/ai-platform-google

# 企业替代：VertexAI（同样原生支持视频）
composer require symfony/ai-platform symfony/ai-platform-vertexai

# 非 Gemini 平台回退方案需要 ffmpeg
# apt install ffmpeg  (Ubuntu/Debian)
# brew install ffmpeg  (macOS)
```

---

## Step 1：Gemini 直接分析视频

Gemini 原生支持视频输入，可以理解视频中的画面和音频。使用 `Video::fromFile()` 直接传入视频文件。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Video;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create($_ENV['GOOGLE_API_KEY'], HttpClient::create());

// Video::fromFile() 原生视频输入
$messages = new MessageBag(
    Message::ofUser(
        '请详细分析这个视频的内容，包括：'
        . "\n1. 视频的主题是什么"
        . "\n2. 视频中有哪些关键画面"
        . "\n3. 视频中提到了哪些重要信息"
        . "\n4. 视频的整体风格和质量",
        Video::fromFile('/path/to/video.mp4'),
    ),
);

$result = $platform->invoke('gemini-2.5-flash', $messages);
echo "视频分析：\n" . $result->asText() . "\n";
```

> **💡 提示：** Gemini 支持的视频格式包括 MP4、AVI、MOV、MKV、WEBM、FLV、MPEG、MPG、WMV、3GP。推荐使用 MP4（H.264 编码），兼容性最好。

### VertexAI 替代方案

如果你的项目部署在 Google Cloud 上，可以使用 VertexAI 获得相同的视频分析能力，同时享受企业级 SLA 和数据驻留保障：

```php
<?php

use Symfony\AI\Platform\Bridge\VertexAi\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Video;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create(
    projectId: $_ENV['VERTEX_PROJECT_ID'],
    location: 'us-central1',
    httpClient: HttpClient::create(),
);

$messages = new MessageBag(
    Message::ofUser(
        '请详细分析这个视频内容：',
        Video::fromFile('/path/to/video.mp4'),
    ),
);

// VertexAI 使用相同的 Gemini 模型
$result = $platform->invoke('gemini-2.5-pro', $messages);
echo "视频分析：\n" . $result->asText() . "\n";
```

> **💡 提示：** `gemini-2.5-flash` 速度快、成本低，适合日常视频分析；`gemini-2.5-pro` 理解能力更强，适合复杂视频内容的深度分析。

---

## Step 2：结构化视频分析报告

使用 `StructuredOutput` 将分析结果映射为 `VideoAnalysis` DTO 对象，包含时间戳、关键时刻、情感分析等字段，方便入库和搜索。

```php
<?php

namespace App\Dto;

final class VideoKeyMoment
{
    /**
     * @param string $timestamp   时间点（如 "01:23"）
     * @param string $description 该时刻的内容描述
     * @param string $importance  重要程度（high/medium/low）
     */
    public function __construct(
        public readonly string $timestamp,
        public readonly string $description,
        public readonly string $importance,
    ) {
    }
}

final class VideoAnalysis
{
    /**
     * @param string           $title          视频标题建议
     * @param string           $summary        内容摘要（100 字以内）
     * @param string           $category       视频分类（education/entertainment/news/tutorial/review）
     * @param string[]         $tags           标签列表
     * @param VideoKeyMoment[] $keyMoments     关键时间点
     * @param string           $targetAudience 目标受众
     * @param string           $sentiment      整体情感（positive/neutral/negative）
     * @param int              $qualityScore   内容质量评分（1-10）
     * @param bool             $hasSensitiveContent 是否包含敏感内容
     */
    public function __construct(
        public readonly string $title,
        public readonly string $summary,
        public readonly string $category,
        public readonly array $tags,
        public readonly array $keyMoments,
        public readonly string $targetAudience,
        public readonly string $sentiment,
        public readonly int $qualityScore,
        public readonly bool $hasSensitiveContent,
    ) {
    }
}
```

### 使用结构化分析：

```php
<?php

use App\Dto\VideoAnalysis;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Video;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$platform = PlatformFactory::create(
    $_ENV['GOOGLE_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);

$messages = new MessageBag(
    Message::forSystem(
        '你是视频内容分析系统。对视频进行全面分析，包括画面、音频、文字等所有信息。'
        . '标记关键时间点（格式 MM:SS），为视频自动分类和打标签。'
    ),
    Message::ofUser(
        '请分析以下视频并生成结构化报告：',
        Video::fromFile('/path/to/tutorial-video.mp4'),
    ),
);

$result = $platform->invoke('gemini-2.5-flash', $messages, [
    'response_format' => VideoAnalysis::class,
]);

$analysis = $result->asObject();

echo "=== 视频分析报告 ===\n";
echo "标题：{$analysis->title}\n";
echo "分类：{$analysis->category}\n";
echo "质量：{$analysis->qualityScore}/10\n";
echo "摘要：{$analysis->summary}\n";
echo "标签：" . implode('、', $analysis->tags) . "\n";
echo "目标受众：{$analysis->targetAudience}\n";
echo "情感倾向：{$analysis->sentiment}\n";
echo "敏感内容：" . ($analysis->hasSensitiveContent ? '⚠️ 是' : '✅ 否') . "\n\n";

echo "关键时间点：\n";
foreach ($analysis->keyMoments as $moment) {
    $icon = match ($moment->importance) {
        'high' => '🔴',
        'medium' => '🟡',
        'low' => '🟢',
        default => '⚪',
    };
    echo "  {$icon} [{$moment->timestamp}] {$moment->description}\n";
}
```

> **💡 提示：** `StructuredOutput` 让 AI 返回的结果直接映射为 PHP 对象。`VideoKeyMoment[]` 数组中的每个元素都会自动反序列化为 `VideoKeyMoment` 实例，无需手动解析 JSON。

---

## Step 3：视频内容审核

使用 `VideoModerationResult` DTO 检查视频是否违规，是否适合发布。审核系统会标注具体违规时间点，方便人工复审。

```php
<?php

namespace App\Dto;

final class VideoModerationResult
{
    /**
     * @param string   $verdict         判定（approved/rejected/review）
     * @param bool     $hasViolence     是否包含暴力内容
     * @param bool     $hasNudity       是否包含裸露内容
     * @param bool     $hasHateSpeech   是否包含仇恨言论
     * @param bool     $hasMisinformation 是否包含虚假信息
     * @param string   $reason          审核理由
     * @param string[] $flaggedMoments  违规时间点列表
     */
    public function __construct(
        public readonly string $verdict,
        public readonly bool $hasViolence,
        public readonly bool $hasNudity,
        public readonly bool $hasHateSpeech,
        public readonly bool $hasMisinformation,
        public readonly string $reason,
        public readonly array $flaggedMoments = [],
    ) {
    }
}
```

```php
<?php

use App\Dto\VideoModerationResult;
use Symfony\AI\Platform\Message\Content\Video;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$messages = new MessageBag(
    Message::forSystem(
        '你是视频内容审核系统。检查视频是否包含违规内容：暴力、裸露、仇恨言论、虚假信息。'
        . '如果发现问题，标注具体出现的时间点。'
    ),
    Message::ofUser(
        '审核以下用户上传的视频：',
        Video::fromFile('/path/to/user-upload.mp4'),
    ),
);

$result = $platform->invoke('gemini-2.5-flash', $messages, [
    'response_format' => VideoModerationResult::class,
]);

$moderation = $result->asObject();

$icon = match ($moderation->verdict) {
    'approved' => '✅',
    'rejected' => '❌',
    'review' => '⚠️',
    default => '❓',
};

echo "{$icon} 审核结果：{$moderation->verdict}\n";
echo "原因：{$moderation->reason}\n";

if ([] !== $moderation->flaggedMoments) {
    echo "违规时间点：\n";
    foreach ($moderation->flaggedMoments as $moment) {
        echo "  ⚡ {$moment}\n";
    }
}
```

> **💡 提示：** AI 审核适合作为第一道防线快速过滤明显违规内容，但不应完全替代人工审核。建议将 `verdict` 为 `review` 的视频推入人工复审队列。

---

## Step 4：非 Gemini 平台的 ffmpeg 回退方案

如果使用 OpenAI/Anthropic 等不直接支持视频的平台，可以用 ffmpeg 提取关键帧，再通过 `Image::fromFile()` 逐帧分析。

```php
<?php

use Symfony\AI\Platform\Message\Content\Audio;
use Symfony\AI\Platform\Message\Content\Image;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

/**
 * ffmpeg 回退方案：拆分视频为关键帧 + 音频
 * 适用于 OpenAI、Anthropic 等不支持原生视频输入的平台
 */

$videoPath = '/path/to/video.mp4';
$framesDir = '/tmp/frames_' . uniqid();
mkdir($framesDir, 0755, true);

// 1. 提取关键帧（每 10 秒一帧）
exec(sprintf(
    'ffmpeg -i %s -vf "fps=1/10" %s/frame_%%04d.jpg 2>&1',
    escapeshellarg($videoPath),
    escapeshellarg($framesDir),
));

// 2. 提取音频
$audioPath = '/tmp/video-audio-' . uniqid() . '.mp3';
exec(sprintf(
    'ffmpeg -i %s -vn -acodec mp3 %s 2>&1',
    escapeshellarg($videoPath),
    escapeshellarg($audioPath),
));

// 3. 用 Image 分析关键帧（GPT-4o / Claude 支持多图）
$frameImages = [];
foreach (glob($framesDir . '/*.jpg') as $framePath) {
    $frameImages[] = Image::fromFile($framePath);
}

$messages = new MessageBag(
    Message::forSystem('以下是视频每 10 秒的关键帧截图。请分析视频内容。'),
    Message::ofUser(
        '这些是视频的关键帧截图，请描述视频的主要内容：',
        ...$frameImages,
    ),
);

$visualAnalysis = $platform->invoke('gpt-4o', $messages);
echo "画面分析：\n" . $visualAnalysis->asText() . "\n\n";

// 4. 分析音频（转录 + 理解）
$audioMessages = new MessageBag(
    Message::ofUser(
        '请转录并总结这段音频的内容：',
        Audio::fromFile($audioPath),
    ),
);
$audioAnalysis = $platform->invoke('gpt-4o-audio-preview', $audioMessages);
echo "音频转录：\n" . $audioAnalysis->asText() . "\n\n";

// 5. 综合分析
$messages = new MessageBag(
    Message::forSystem('根据视频的画面分析和音频转录，生成综合分析报告。'),
    Message::ofUser(
        "画面分析：\n{$visualAnalysis->asText()}\n\n"
        . "音频转录：\n{$audioAnalysis->asText()}\n\n"
        . '请综合以上信息，给出视频的完整分析。'
    ),
);

$combined = $platform->invoke('gpt-4o-mini', $messages);
echo "综合分析：\n" . $combined->asText() . "\n";

// 6. 清理临时文件
array_map('unlink', glob($framesDir . '/*.jpg'));
rmdir($framesDir);
unlink($audioPath);
```

> **💡 提示：** ffmpeg 关键帧提取间隔可根据视频类型调整：教学视频建议 5-10 秒/帧，短视频建议 2-3 秒/帧。帧数越多分析越精确，但 API 调用成本也越高。注意大多数平台对单次请求的图片数量有上限（如 OpenAI 最多 20 张）。

---

## Step 5：生产环境最佳实践

### 文件大小与处理时间

> **💡 提示：** Gemini API 对视频文件大小有限制（通常 2GB 以内）。超过限制时，建议先用 ffmpeg 压缩：`ffmpeg -i input.mp4 -vcodec libx264 -crf 28 -preset fast output.mp4`。视频越长，API 处理时间越长（10 分钟视频约需 30-60 秒分析）。

### 异步处理架构

视频分析通常耗时较长，生产环境中应使用异步消息队列处理：

```php
<?php

namespace App\MessageHandler;

use App\Dto\VideoAnalysis;
use App\Message\AnalyzeVideoMessage;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Video;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
final class AnalyzeVideoHandler
{
    public function __construct(
        private PlatformFactory $platformFactory,
    ) {
    }

    public function __invoke(AnalyzeVideoMessage $message)
    {
        $messages = new MessageBag(
            Message::forSystem('你是视频内容分析系统。'),
            Message::ofUser(
                '请分析视频：',
                Video::fromFile($message->videoPath),
            ),
        );

        $result = $this->platformFactory->invoke('gemini-2.5-flash', $messages, [
            'response_format' => VideoAnalysis::class,
        ]);

        $analysis = $result->asObject();

        // 存储分析结果、生成缩略图、更新数据库...
    }
}
```

### 缩略图选择与存储优化

```php
<?php

/**
 * 根据 AI 分析结果自动选择最佳缩略图
 *
 * @param VideoAnalysis $analysis  AI 分析结果
 * @param string        $videoPath 视频文件路径
 * @param string        $outputDir 输出目录
 *
 * @return string 缩略图路径
 */
function selectThumbnail(
    VideoAnalysis $analysis,
    string $videoPath,
    string $outputDir,
): string {
    // 找到重要性最高的关键时刻作为缩略图
    $bestMoment = null;
    foreach ($analysis->keyMoments as $moment) {
        if ('high' === $moment->importance) {
            $bestMoment = $moment;
            break;
        }
    }

    // 默认取第一个关键时刻，或视频 10% 处
    $timestamp = null !== $bestMoment ? $bestMoment->timestamp : '00:05';

    $thumbnailPath = $outputDir . '/thumbnail_' . uniqid() . '.jpg';
    exec(sprintf(
        'ffmpeg -i %s -ss %s -vframes 1 -q:v 2 %s 2>&1',
        escapeshellarg($videoPath),
        escapeshellarg($timestamp),
        escapeshellarg($thumbnailPath),
    ));

    return $thumbnailPath;
}
```

> **💡 提示：** 生产环境中建议将原始视频存储到对象存储（如 S3），分析结果和缩略图存储到数据库和 CDN。避免在 PHP 进程中长时间持有大视频文件的内存引用。对于高并发场景，使用 Symfony Messenger 的 `delay` 和 `retry` 策略控制 API 调用频率。

---

## 关于视频处理的性能考虑

> **Q：视频处理会不会太耗时，不适合 PHP？**

**A：需要区分两种情况：**

| 操作 | 耗时 | PHP 适合？ |
|------|------|-----------|
| 视频上传到 AI API | 取决于文件大小和网络 | ✅ 适合（异步上传） |
| AI 分析视频内容 | 10-60 秒 | ✅ 适合（等待 API 响应） |
| 提取关键帧（ffmpeg） | 几秒 | ✅ 适合（调用外部命令） |
| 实时视频流处理 | 持续运行 | ❌ 不适合（用 Python/Go） |
| 视频编码/转码 | CPU 密集 | ❌ 不适合（用专门服务） |

**结论：** PHP 作为"编排层"非常合适 — 调用 AI API 分析视频、提取信息、生成报告。但实时视频流处理和视频编码应该交给专门的服务。

**推荐架构：**
```
用户上传视频 → [消息队列] → PHP Worker 调用 AI API 分析 → 存储结果
```

---

## 完整流程

```
上传视频
    │
    ├──── Gemini / VertexAI ─┐
    │     Video::fromFile()   │
    │     直接分析视频         │
    │                         │
    ├──── 其他平台（回退）────┤
    │     ffmpeg 拆分          │
    │     ├─ 关键帧 → Image   │
    │     └─ 音频 → Audio     │
    │     分别分析后综合       │
    │                         │
    └───────────┬─────────────┘
                │
                ▼
        StructuredOutput
        ├─ VideoAnalysis DTO
        │  ├─ 标题 & 摘要
        │  ├─ 标签 & 分类
        │  ├─ 关键时间点（timestamps）
        │  └─ 情感分析（sentiment）
        └─ VideoModerationResult DTO
           ├─ 审核判定
           └─ 违规时间点
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Video::fromFile()` | 加载视频文件（Gemini/VertexAI 原生支持） |
| Gemini 视频理解 | 直接分析视频画面+音频，推荐 `gemini-2.5-flash` |
| VertexAI 替代 | 企业级部署，同样的 Gemini 模型，GCP 数据驻留 |
| ffmpeg 回退 | 非 Gemini 平台提取关键帧 + 音频间接分析 |
| `StructuredOutput` | `VideoAnalysis` DTO 含时间戳、关键时刻、情感分析 |
| `VideoModerationResult` | 内容审核 DTO，标注违规类型和时间点 |
| 异步处理 | Symfony Messenger 队列处理耗时视频分析 |
| 缩略图选择 | 根据 AI 分析结果自动选择最佳封面帧 |
| 文件大小限制 | Gemini 最大约 2GB，超限先用 ffmpeg 压缩 |
| 性能边界 | API 调用适合 PHP，实时流处理不适合 |

## 下一步

如果你想把图片、语音、文字等多模态能力组合成一个完整的内容生产流水线，请看 [22-multimodal-content-creation-pipeline.md](./22-multimodal-content-creation-pipeline.md)。
