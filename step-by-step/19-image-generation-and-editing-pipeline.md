# AI 图片生成与编辑流水线

## 业务场景

你在做一个电商平台的营销工具。设计师提出需求：输入一段商品描述，AI 自动生成商品宣传图；或者上传一张产品原图，AI 进行风格化编辑（换背景、加文字、调色调）。最终输出多个方案供设计师选择。

**典型应用：** 电商商品图自动生成、社交媒体配图、广告素材批量制作、产品照片美化

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台（OpenAI DALL-E / Gemini） |
| **Image Content** | 图片输入与输出处理 |
| **Agent** | 编排多步图片处理工作流 |
| **StructuredOutput** | 结构化的图片评审反馈 |

## 前置准备

```bash
composer require symfony/ai-platform symfony/ai-platform-openai
composer require symfony/ai-agent
```

需要 OpenAI API 密钥（支持 DALL-E 模型）。

---

## Step 1：文字生成图片（Text-to-Image）

用 DALL-E 3 根据文字描述生成商品图片。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());

// 场景：根据商品描述生成宣传图
$productDescription = '极简设计的白色无线蓝牙耳机，放在大理石桌面上，旁边有一杯拿铁咖啡，自然光照射，高级感摄影风格';

$messages = new MessageBag(
    Message::ofUser($productDescription),
);

$result = $platform->invoke('dall-e-3', $messages, [
    'size' => '1024x1024',
    'quality' => 'hd',      // 高清模式
    'style' => 'natural',   // natural（自然）或 vivid（生动）
]);

// 保存生成的图片
$imageData = $result->asBinary();
file_put_contents('/tmp/product-hero.png', $imageData);
echo "✅ 商品图已生成：/tmp/product-hero.png\n";
```

---

## Step 2：图片理解与分析

在编辑图片之前，先让 AI 分析原图内容。

```php
<?php

use Symfony\AI\Platform\Message\Content\Image;

// 上传一张产品原图，AI 分析内容
$messages = new MessageBag(
    Message::ofUser(
        '请详细分析这张产品图片的内容，包括：主体物品、背景环境、光线、色调、构图、适合的使用场景。',
        Image::fromFile('/path/to/product-photo.jpg'),
    ),
);

$result = $platform->invoke('gpt-4o', $messages);
echo "图片分析：\n" . $result->asText() . "\n";
```

---

## Step 3：结构化图片评审

设计团队审核 AI 生成的图片时，需要结构化的评分和反馈。

```php
<?php

namespace App\Dto;

final class ImageReview
{
    /**
     * @param int      $compositionScore  构图评分（1-10）
     * @param int      $colorScore        色彩评分（1-10）
     * @param int      $relevanceScore    与描述相关度（1-10）
     * @param int      $qualityScore      整体质量（1-10）
     * @param string   $strengths         优点
     * @param string   $weaknesses        不足
     * @param string[] $improvementSuggestions 改进建议
     * @param bool     $readyForUse       是否可直接使用
     */
    public function __construct(
        public readonly int $compositionScore,
        public readonly int $colorScore,
        public readonly int $relevanceScore,
        public readonly int $qualityScore,
        public readonly string $strengths,
        public readonly string $weaknesses,
        public readonly array $improvementSuggestions,
        public readonly bool $readyForUse,
    ) {
    }
}
```

### 使用结构化评审：

```php
<?php

use App\Dto\ImageReview;
use Symfony\AI\Platform\Message\Content\Image;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$platform = PlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);

// AI 审核生成的图片
$messages = new MessageBag(
    Message::forSystem(
        '你是专业的视觉设计评审。评估商品宣传图的质量。'
        . '原始需求：极简白色蓝牙耳机，大理石桌面，拿铁咖啡，自然光，高级感。'
    ),
    Message::ofUser(
        '请评审这张 AI 生成的商品图：',
        Image::fromFile('/tmp/product-hero.png'),
    ),
);

$result = $platform->invoke('gpt-4o', $messages, [
    'response_format' => ImageReview::class,
]);

$review = $result->asObject();

echo "=== 图片评审报告 ===\n";
echo "构图：{$review->compositionScore}/10\n";
echo "色彩：{$review->colorScore}/10\n";
echo "相关度：{$review->relevanceScore}/10\n";
echo "质量：{$review->qualityScore}/10\n";
echo "总分：" . ($review->compositionScore + $review->colorScore + $review->relevanceScore + $review->qualityScore) . "/40\n\n";
echo "优点：{$review->strengths}\n";
echo "不足：{$review->weaknesses}\n";
echo "可直接使用：" . ($review->readyForUse ? '✅ 是' : '❌ 需修改') . "\n\n";

if ([] !== $review->improvementSuggestions) {
    echo "改进建议：\n";
    foreach ($review->improvementSuggestions as $suggestion) {
        echo "  💡 {$suggestion}\n";
    }
}
```

---

## Step 4：Agent 编排图片处理工作流

使用 Agent 自动化整个流程：分析需求 → 生成 prompt → 生成图片 → 自动评审。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Attribute\AsTool;
use Symfony\AI\Agent\Toolbox\Toolbox;

// 自定义工具：生成图片
#[AsTool('generate_image', description: '根据提示词生成商品图片', method: 'generate')]
final class ImageGenerator
{
    public function __construct(private object $platform)
    {
    }

    public function generate(string $prompt, string $style = 'natural'): string
    {
        $result = $this->platform->invoke('dall-e-3', new MessageBag(
            Message::ofUser($prompt),
        ), [
            'size' => '1024x1024',
            'quality' => 'hd',
            'style' => $style,
        ]);

        $path = '/tmp/generated-' . time() . '.png';
        file_put_contents($path, $result->asBinary());

        return "图片已生成并保存到：{$path}";
    }
}

// 自定义工具：分析图片
#[AsTool('analyze_image', description: '分析图片内容和质量', method: 'analyze')]
final class ImageAnalyzer
{
    public function __construct(private object $platform)
    {
    }

    public function analyze(string $imagePath): string
    {
        $messages = new MessageBag(
            Message::ofUser(
                '请从构图、色彩、质量、商业价值四个维度分析这张图片：',
                Image::fromFile($imagePath),
            ),
        );

        $result = $this->platform->invoke('gpt-4o', $messages);

        return $result->asText();
    }
}

// 组装 Agent
$toolbox = new Toolbox([
    new ImageGenerator($platform),
    new ImageAnalyzer($platform),
]);
$processor = new AgentProcessor($toolbox);

$agent = new Agent(
    $platform, 'gpt-4o',
    [
        new SystemPromptInputProcessor(
            '你是电商视觉设计助手。你可以生成和分析商品图片。'
            . "\n流程：1) 理解用户需求 2) 优化提示词 3) 调用 generate_image 生成图片"
            . ' 4) 调用 analyze_image 评审图片 5) 给出最终建议。'
        ),
        $processor,
    ],
    [$processor],
);

// 一句话生成并评审
$result = $agent->call(new MessageBag(
    Message::ofUser('帮我为一款智能手表生成一张电商首页 Banner 图，要有科技感和高级感。'),
));

echo $result->getContent() . "\n";
```

---

## Step 5：图片对比与 A/B 测试

对比多个方案，选出最佳。

```php
<?php

// 同时展示两张图让 AI 对比
$messages = new MessageBag(
    Message::forSystem('你是视觉设计总监。对比两张商品图，推荐更适合电商首页的方案。'),
    Message::ofUser(
        '请对比以下两个方案，从吸引力、品牌调性、点击转化潜力三个角度分析：',
        Image::fromFile('/tmp/option-a.png'),
        Image::fromFile('/tmp/option-b.png'),
    ),
);

$result = $platform->invoke('gpt-4o', $messages);
echo "A/B 对比结果：\n" . $result->asText() . "\n";
```

---

## 完整流程

```
用户需求 → [Agent 分析需求]
              │
              ▼
     [优化 Prompt 词]
              │
              ▼
     [DALL-E 生成图片] × N 个方案
              │
              ▼
     [GPT-4o 视觉评审] → 结构化评分
              │
              ▼
     ┌────────┼─────────┐
     ▼        ▼         ▼
   方案A    方案B     方案C
     │        │         │
     └────────┼─────────┘
              ▼
     [A/B 对比] → 最佳推荐
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `dall-e-3` 模型 | OpenAI 图片生成，支持 size/quality/style 参数 |
| `Image::fromFile()` | 从文件加载图片用于视觉分析 |
| `Image::fromUrl()` | 从 URL 加载图片 |
| `$result->asBinary()` | 获取生成图片的二进制数据 |
| 自定义 `#[AsTool]` | 将图片生成/分析封装为 Agent 工具 |
| 多图输入 | 一条消息中传入多张图片做对比 |
| 结构化评审 | `ImageReview` 对象提供量化评分 |

## 下一步

如果你需要一个语音交互的 AI 助手（语音输入 → AI 处理 → 语音输出），请看 [20-voice-interactive-assistant.md](./20-voice-interactive-assistant.md)。
