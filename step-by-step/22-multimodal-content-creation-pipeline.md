# 多模态内容生产流水线

## 业务场景

你在做一个内容营销平台。运营人员只需要输入一个主题（如"新品发布"），AI 自动生成一整套内容物料：文案（中英双语）、配图、语音播报、短视频脚本。所有环节自动串联，最终输出可直接发布的多媒体内容包。

**典型应用：** 营销内容自动化、社交媒体批量发帖、多语言内容分发、播客/视频脚本生成

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform (OpenAI)** | 文案生成 + 图片生成 |
| **Platform (ElevenLabs)** | 文字转语音 |
| **Platform (Gemini)** | 视觉审核 |
| **Agent** | 编排完整的内容生产流水线 |
| **Agent Bridge Filesystem** | 保存生成的文件 |
| **StructuredOutput** | 结构化的内容计划 |

---

## Step 1：定义内容计划结构

AI 先规划要生成什么内容，再逐步执行。

```php
<?php

namespace App\Dto;

final class ContentPiece
{
    /**
     * @param string $type    内容类型（text/image/audio/video_script）
     * @param string $title   内容标题
     * @param string $prompt  生成提示
     * @param string $channel 发布渠道（wechat/weibo/xiaohongshu/twitter/linkedin）
     */
    public function __construct(
        public readonly string $type,
        public readonly string $title,
        public readonly string $prompt,
        public readonly string $channel,
    ) {
    }
}

final class ContentPlan
{
    /**
     * @param string         $theme       内容主题
     * @param string         $targetAudience 目标受众
     * @param string         $brandTone   品牌调性（professional/casual/playful/luxury）
     * @param ContentPiece[] $pieces      内容列表
     * @param string         $coreMessage 核心信息
     */
    public function __construct(
        public readonly string $theme,
        public readonly string $targetAudience,
        public readonly string $brandTone,
        public readonly array $pieces,
        public readonly string $coreMessage,
    ) {
    }
}
```

---

## Step 2：AI 生成内容计划

```php
<?php

require 'vendor/autoload.php';

use App\Dto\ContentPlan;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
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

$messages = new MessageBag(
    Message::forSystem(
        '你是内容营销策划专家。根据主题生成完整的内容分发计划。'
        . '每个渠道需要不同风格的内容：微信公众号偏长文、微博偏短文、小红书偏种草、Twitter 偏英文。'
        . '同时规划配图和语音版本。'
    ),
    Message::ofUser('主题：CloudFlow 2.0 新版本发布 — 全新 AI 协作功能。请制定内容计划。'),
);

$result = $platform->invoke('gpt-4o-mini', $messages, [
    'response_format' => ContentPlan::class,
]);

$plan = $result->asObject();

echo "=== 内容生产计划 ===\n";
echo "主题：{$plan->theme}\n";
echo "受众：{$plan->targetAudience}\n";
echo "调性：{$plan->brandTone}\n";
echo "核心信息：{$plan->coreMessage}\n\n";

echo "内容列表（共 " . count($plan->pieces) . " 项）：\n";
foreach ($plan->pieces as $i => $piece) {
    echo "  " . ($i + 1) . ". [{$piece->type}] {$piece->title} → {$piece->channel}\n";
}
```

---

## Step 3：按计划批量生成内容

```php
<?php

use Symfony\AI\Platform\Bridge\ElevenLabs\PlatformFactory as ElevenLabsFactory;
use Symfony\AI\Platform\Message\Content\Text;

$elevenLabs = ElevenLabsFactory::create($_ENV['ELEVENLABS_API_KEY'], HttpClient::create());

$outputDir = '/tmp/content-pack-' . date('Ymd');
if (!is_dir($outputDir)) {
    mkdir($outputDir, 0755, true);
}

foreach ($plan->pieces as $piece) {
    echo "\n⏳ 生成：{$piece->title}...\n";

    match ($piece->type) {
        // 文案生成
        'text' => (function () use ($platform, $piece, $outputDir) {
            $result = $platform->invoke('gpt-4o-mini', new MessageBag(
                Message::forSystem("你是资深文案。品牌：CloudFlow。渠道：{$piece->channel}。"),
                Message::ofUser($piece->prompt),
            ));

            $filename = "{$outputDir}/{$piece->channel}-text.md";
            file_put_contents($filename, $result->asText());
            echo "  ✅ 文案已保存：{$filename}\n";
        })(),

        // 图片生成
        'image' => (function () use ($platform, $piece, $outputDir) {
            $result = $platform->invoke('dall-e-3', new MessageBag(
                Message::ofUser($piece->prompt),
            ), [
                'size' => '1024x1024',
                'quality' => 'hd',
            ]);

            $filename = "{$outputDir}/{$piece->channel}-image.png";
            file_put_contents($filename, $result->asBinary());
            echo "  ✅ 图片已保存：{$filename}\n";
        })(),

        // 语音生成
        'audio' => (function () use ($platform, $elevenLabs, $piece, $outputDir) {
            // 先生成文案
            $textResult = $platform->invoke('gpt-4o-mini', new MessageBag(
                Message::forSystem('你是播客主持人。写一段适合朗读的产品介绍，1-2 分钟。'),
                Message::ofUser($piece->prompt),
            ));

            $script = $textResult->asText();
            file_put_contents("{$outputDir}/audio-script.md", $script);

            // 再转语音
            $audioResult = $elevenLabs->invoke(
                'eleven_multilingual_v2',
                new Text($script),
                ['voice' => 'pqHfZKP75CvOlQylNhV4'],
            );

            $filename = "{$outputDir}/audio-broadcast.mp3";
            file_put_contents($filename, $audioResult->asBinary());
            echo "  ✅ 语音已保存：{$filename}\n";
        })(),

        // 视频脚本
        'video_script' => (function () use ($platform, $piece, $outputDir) {
            $result = $platform->invoke('gpt-4o-mini', new MessageBag(
                Message::forSystem(
                    '你是短视频脚本编剧。输出格式：'
                    . '[镜头N] 画面描述 | 旁白文字 | 时长'
                ),
                Message::ofUser($piece->prompt),
            ));

            $filename = "{$outputDir}/video-script.md";
            file_put_contents($filename, $result->asText());
            echo "  ✅ 脚本已保存：{$filename}\n";
        })(),

        default => echo "  ⚠️ 未知类型：{$piece->type}\n",
    };
}

echo "\n📦 内容包已生成在：{$outputDir}/\n";
```

---

## Step 4：AI 自动质量审核

生成完所有内容后，AI 对整体质量进行审核。

```php
<?php

use Symfony\AI\Platform\Message\Content\Image;

// 审核生成的图片
$imageFiles = glob("{$outputDir}/*.png");
$images = array_map(fn ($path) => Image::fromFile($path), $imageFiles);

$messages = new MessageBag(
    Message::forSystem(
        '你是品牌审核经理。检查所有生成的营销素材是否：'
        . "\n1. 符合品牌调性（专业、现代、科技感）"
        . "\n2. 各渠道风格是否适配"
        . "\n3. 有无不当内容"
        . "\n4. 图文是否一致"
    ),
    Message::ofUser(
        "品牌：CloudFlow\n核心信息：{$plan->coreMessage}\n\n请审核以下生成的配图：",
        ...$images,
    ),
);

$result = $platform->invoke('gpt-4o', $messages);
echo "\n=== 内容审核 ===\n" . $result->asText() . "\n";
```

---

## Step 5：使用 Agent 编排完整流水线

把所有步骤用 Agent + 自定义工具自动化。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\Filesystem\Filesystem;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Attribute\AsTool;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\Component\Filesystem\Filesystem as SymfonyFilesystem;

// 文案生成工具
#[AsTool('generate_text', description: '生成指定渠道的营销文案', method: '__invoke')]
final class TextGeneratorTool
{
    public function __construct(private object $platform)
    {
    }

    public function __invoke(string $prompt, string $channel): string
    {
        $result = $this->platform->invoke('gpt-4o-mini', new MessageBag(
            Message::forSystem("渠道：{$channel}。品牌：CloudFlow。"),
            Message::ofUser($prompt),
        ));

        return $result->asText();
    }
}

// 图片生成工具
#[AsTool('generate_image', description: '根据描述生成营销配图', method: '__invoke')]
final class ImageGeneratorTool
{
    public function __construct(private object $platform)
    {
    }

    public function __invoke(string $prompt): string
    {
        $result = $this->platform->invoke('dall-e-3', new MessageBag(
            Message::ofUser($prompt),
        ), ['size' => '1024x1024', 'quality' => 'hd']);

        $path = '/tmp/generated-' . uniqid() . '.png';
        file_put_contents($path, $result->asBinary());

        return "图片已保存到：{$path}";
    }
}

// 语音生成工具
#[AsTool('generate_audio', description: '将文字转为语音播报', method: '__invoke')]
final class AudioGeneratorTool
{
    public function __construct(private object $elevenLabs)
    {
    }

    public function __invoke(string $text): string
    {
        $result = $this->elevenLabs->invoke(
            'eleven_multilingual_v2',
            new Text($text),
            ['voice' => 'pqHfZKP75CvOlQylNhV4'],
        );

        $path = '/tmp/audio-' . uniqid() . '.mp3';
        file_put_contents($path, $result->asBinary());

        return "语音已保存到：{$path}";
    }
}

// Filesystem 工具用于保存文件
$filesystem = new Filesystem(
    new SymfonyFilesystem(),
    '/tmp/content-output',
    allowWrite: true,
);

$toolbox = new Toolbox([
    new TextGeneratorTool($platform),
    new ImageGeneratorTool($platform),
    new AudioGeneratorTool($elevenLabs),
    $filesystem,
]);

$processor = new AgentProcessor($toolbox);

$agent = new Agent(
    $platform, 'gpt-4o',
    [
        new SystemPromptInputProcessor(
            '你是内容生产总监。根据用户的主题需求，自动规划和生成一整套内容物料：'
            . "\n1. 先确定内容计划（渠道、类型、风格）"
            . "\n2. 用 generate_text 生成各渠道文案"
            . "\n3. 用 generate_image 生成配图"
            . "\n4. 用 generate_audio 生成语音播报"
            . "\n5. 用 filesystem_write 保存所有文件"
            . "\n6. 最后汇报生成结果"
        ),
        $processor,
    ],
    [$processor],
);

// 一句话启动整个流水线
$result = $agent->call(new MessageBag(
    Message::ofUser('主题：CloudFlow 2.0 发布。生成微信公众号文案+配图+语音播报。'),
));

echo $result->getContent() . "\n";
```

---

## 完整流程

```
用户输入主题
    │
    ▼
[AI 规划内容计划] → ContentPlan 对象
    │
    ├─── 文案生成 ───► 微信/微博/小红书/Twitter 文案
    │
    ├─── 图片生成 ───► DALL-E 配图（多尺寸）
    │
    ├─── 语音生成 ───► ElevenLabs TTS 播报音频
    │
    ├─── 脚本生成 ───► 短视频分镜脚本
    │
    └─── 质量审核 ───► GPT-4o 视觉+文案审核
              │
              ▼
        📦 内容包（文案+图片+音频+脚本）
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| 内容计划 DTO | `ContentPlan` 结构化规划内容矩阵 |
| 多平台协作 | OpenAI 文案+图片、ElevenLabs 语音、Gemini 审核 |
| Agent 工具化 | 每个生成能力封装为 `#[AsTool]` |
| Filesystem 保存 | 生成的文件自动保存到指定目录 |
| 批量生成 | 一个主题 → 多渠道多形式内容 |
| 自动审核 | GPT-4o 视觉能力审核生成的图片 |

## 下一步

如果你想用 PHP 搭建一个类似 CrewAI 的协作智能体团队系统，请看 [23-crewai-style-agent-team.md](./23-crewai-style-agent-team.md)。
