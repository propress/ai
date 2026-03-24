# 视频内容分析与总结

## 业务场景

你在做一个短视频/在线教育平台。需要 AI 自动分析上传的视频：提取关键画面、总结视频主题、生成文字摘要、标注关键时间点。用于自动审核视频内容，或为视频生成字幕和介绍文案。

**典型应用：** 视频内容审核、教学视频摘要、短视频标签生成、会议录像提取要点

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台（Gemini 原生支持视频） |
| **Video Content** | 视频文件输入 |
| **Audio Content** | 视频音轨单独分析 |
| **StructuredOutput** | 结构化的视频分析报告 |

## 平台选择说明

目前原生支持视频输入的 AI 平台是 **Gemini / VertexAI**。其他平台可以用"提取关键帧 + 音频"的方式间接处理视频。

## 前置准备

```bash
composer require symfony/ai-platform symfony/ai-platform-google  # Gemini
# 或者
composer require symfony/ai-platform symfony/ai-platform-vertexai  # VertexAI
```

---

## Step 1：Gemini 直接分析视频

Gemini 原生支持视频输入，可以理解视频中的画面和音频。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\Google\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Video;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create($_ENV['GOOGLE_API_KEY'], HttpClient::create());

// 直接输入视频文件
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

$result = $platform->invoke('gemini-2.0-flash', $messages);
echo "视频分析：\n" . $result->asText() . "\n";
```

---

## Step 2：结构化视频分析报告

将分析结果映射为结构化对象，方便入库和搜索。

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
use Symfony\AI\Platform\Message\Content\Video;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

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

$result = $platform->invoke('gemini-2.0-flash', $messages, [
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

---

## Step 3：视频内容审核

检查视频是否违规，是否适合发布。

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

$result = $platform->invoke('gemini-2.0-flash', $messages, [
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

---

## Step 4：非 Gemini 平台的替代方案

如果使用 OpenAI/Anthropic（不直接支持视频），可以用"关键帧 + 音频"拆分策略。

```php
<?php

use Symfony\AI\Platform\Message\Content\Audio;
use Symfony\AI\Platform\Message\Content\Image;

/**
 * 方案 B：拆分视频为关键帧 + 音频
 * 使用 ffmpeg 命令行工具提取
 */

// 1. 提取关键帧（每 10 秒一帧）
exec('ffmpeg -i /path/to/video.mp4 -vf "fps=1/10" /tmp/frames/frame_%04d.jpg');

// 2. 提取音频
exec('ffmpeg -i /path/to/video.mp4 -vn -acodec mp3 /tmp/video-audio.mp3');

// 3. 分析关键帧（GPT-4o 支持多图）
$frameImages = [];
foreach (glob('/tmp/frames/*.jpg') as $framePath) {
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
$audioAnalysis = $elevenLabs->invoke('scribe_v1', Audio::fromFile('/tmp/video-audio.mp3'));
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
```

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
    ├──── Gemini 平台 ────┐
    │     Video::fromFile()  │
    │     直接分析视频       │
    │                        │
    ├──── 其他平台 ─────────┤
    │     ffmpeg 拆分        │
    │     ├─ 关键帧 → Image  │
    │     └─ 音频 → Audio    │
    │     分别分析后综合      │
    │                        │
    └────────────┬───────────┘
                 │
                 ▼
         结构化分析报告
         ├─ 标题 & 摘要
         ├─ 标签 & 分类
         ├─ 关键时间点
         └─ 内容审核结果
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Video::fromFile()` | 加载视频文件（Gemini 原生支持） |
| Gemini 视频理解 | 直接分析视频画面+音频 |
| 关键帧提取 | 非 Gemini 平台的替代方案 |
| 结构化报告 | `VideoAnalysis` 对象含标签、关键点、评分 |
| 视频审核 | 检查暴力/裸露/仇恨/虚假信息 |
| ffmpeg 配合 | PHP 调用 ffmpeg 提取帧和音频 |
| 性能边界 | API 调用适合 PHP，实时流处理不适合 |

## 下一步

如果你想把图片、语音、文字等多模态能力组合成一个完整的内容生产流水线，请看 [22-multimodal-content-creation-pipeline.md](./22-multimodal-content-creation-pipeline.md)。
