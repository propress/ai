# AI 图片生成与编辑流水线

## 业务场景

你在做一个电商平台的营销工具。设计师提出需求：输入一段商品描述，AI 自动生成商品宣传图；或者上传一张产品原图，AI 进行风格化编辑（换背景、加文字、调色调）。最终输出多个方案供设计师选择。

**典型应用：** 电商商品图自动生成、社交媒体配图、广告素材批量制作、产品照片美化

## 涉及模块

| 模块 | Composer 包 | 用途 |
|------|------------|------|
| **Platform** | `symfony/ai-platform` + `symfony/ai-openai-platform` | 连接 OpenAI DALL-E 图片生成 |
| **Image Content** | `symfony/ai-platform`（内含） | 图片输入（`Image`、`ImageUrl`）与输出处理 |
| **Agent** | `symfony/ai-agent` | 编排多步图片处理工作流 |
| **StructuredOutput** | `symfony/ai-platform`（内含） | 结构化的图片评审反馈 |

> **💡 提示：** 本教程使用 **OpenAI DALL-E 3** 作为主要图片生成平台——它在文字到图片的理解力和生成质量上表现优异。Step 6 会演示如何使用 **Gemini Imagen** 作为替代方案。图片分析部分使用 GPT-4o 的视觉能力。

## 项目流程图

```
┌──────────┐      ┌───────────────┐      ┌──────────────────┐      ┌──────────────┐
│  用户输入  │ ──▶ │ Agent 分析需求  │ ──▶ │  优化 Prompt 词   │ ──▶ │ DALL-E 生成   │
│ (商品描述) │      │ (理解意图)      │      │ (增强细节描述)     │      │ (图片输出)     │
└──────────┘      └───────────────┘      └──────────────────┘      └──────┬───────┘
                                                                          │
                                                              ┌───────────┼───────────┐
                                                              ▼           ▼           ▼
                                                          方案 A      方案 B      方案 C
                                                              │           │           │
                                                              └───────────┼───────────┘
                                                                          ▼
                                                                 ┌──────────────┐
                                                                 │ GPT-4o 视觉   │
                                                                 │ 评审（结构化）  │
                                                                 └──────┬───────┘
                                                                        ▼
                                                                ┌──────────────┐
                                                                │ A/B 对比选优  │
                                                                │ → 最佳推荐    │
                                                                └──────────────┘
```

---

## 前置准备

### 环境要求

- PHP 8.2+
- Composer
- OpenAI API 密钥（需开通 DALL-E 访问权限）

### 安装依赖

```bash
composer require symfony/ai-platform symfony/ai-openai-platform symfony/ai-agent
```

> **🔒 安全建议：** 不要把 API 密钥硬编码在源码中。使用环境变量或 Symfony Secrets 管理敏感信息。DALL-E 3 每次调用均产生费用，建议在开发环境设置用量告警。

---

## Step 1：文字生成图片（Text-to-Image）

用 DALL-E 3 根据文字描述生成商品图片。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);

// 场景：根据商品描述生成宣传图
$productDescription = '极简设计的白色无线蓝牙耳机，放在大理石桌面上，'
    . '旁边有一杯拿铁咖啡，自然光照射，高级感摄影风格';

$messages = new MessageBag(
    Message::ofUser($productDescription),
);

$response = $platform->invoke('dall-e-3', $messages, [
    'size' => '1024x1024',
    'quality' => 'hd',
    'style' => 'natural',
    'response_format' => 'b64_json',
]);

// ImageResult 包含 Base64Image 对象列表
$imageResult = $response->getResult();

// 获取 DALL-E 3 优化后的提示词（仅 DALL-E 3 返回）
if (null !== $imageResult->revisedPrompt) {
    echo "DALL-E 优化后的提示词：{$imageResult->revisedPrompt}\n\n";
}

// 遍历生成的图片并保存
foreach ($imageResult->getContent() as $index => $image) {
    $path = "/tmp/product-hero-{$index}.png";
    file_put_contents($path, base64_decode($image->encodedImage));
    echo "✅ 图片已保存：{$path}\n";
}
```

> **💡 提示：** DALL-E 3 的 `style` 参数影响显著——`vivid` 会产生超现实、戏剧化的效果，适合创意广告；`natural` 更接近真实摄影，适合商品展示图。建议先用 `natural` 生成，再用 `vivid` 探索创意方向。

> **⚠️ 注意：** DALL-E 3 的 `size` 支持 `1024x1024`、`1024x1792`（竖版）和 `1792x1024`（横版）。`quality` 设为 `hd` 会提升细节但费用翻倍。生产环境中建议根据用途选择：商品主图用 `hd`，缩略图用标准质量即可。

---

## Step 2：图片理解与分析

在编辑图片之前，先让 AI 分析原图内容。框架提供了三种加载图片的方式。

```php
<?php

use Symfony\AI\Platform\Message\Content\Image;
use Symfony\AI\Platform\Message\Content\ImageUrl;

// 方式 1：从本地文件加载（最常用）
$localImage = Image::fromFile('/path/to/product-photo.jpg');

// 方式 2：从 URL 加载（适合分析线上图片）
$urlImage = new ImageUrl('https://example.com/product.jpg');

// 方式 3：从 Data URL 加载（适合前端直传的 Base64 数据）
$dataUrlImage = Image::fromDataUrl('data:image/jpeg;base64,/9j/4AAQ...');

// 使用 Image::fromFile() 进行产品图分析
$messages = new MessageBag(
    Message::ofUser(
        '请详细分析这张产品图片的内容，包括：'
        . '主体物品、背景环境、光线、色调、构图、适合的使用场景。',
        Image::fromFile('/path/to/product-photo.jpg'),
    ),
);

$result = $platform->invoke('gpt-4o', $messages);
echo "图片分析：\n" . $result->getContent() . "\n";
```

> **💡 提示：** `Image` 继承自 `File` 类，提供了丰富的格式转换方法：`asBinary()` 获取原始二进制、`asBase64()` 获取 Base64 编码、`asDataUrl()` 获取完整 Data URL、`asResource()` 获取文件句柄。这些方法在图片处理管线中非常实用——例如将 AI 生成的图片直接传入下一步分析。

---

## Step 3：结构化图片评审（StructuredOutput）

设计团队审核 AI 生成的图片时，需要结构化的评分和反馈。通过 StructuredOutput 让 AI 直接返回 PHP 对象。

### 定义评审 DTO：

```php
<?php

namespace App\Dto;

final class ImageReview
{
    /**
     * @param int      $compositionScore      构图评分（1-10，10 为最佳）
     * @param int      $colorScore            色彩和谐度评分（1-10）
     * @param int      $relevanceScore        与原始描述的相关度（1-10）
     * @param int      $qualityScore          整体画面质量（1-10）
     * @param string   $strengths             图片的主要优点（2-3 句话）
     * @param string   $weaknesses            图片的主要不足（2-3 句话）
     * @param string[] $improvementSuggestions 具体可操作的改进建议列表
     * @param bool     $readyForUse           是否达到可直接商用的标准
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

> **💡 提示：** PHPDoc 中的 `@param` 描述会被 `ResponseFormatFactory` 自动提取为 JSON Schema 的 `description` 字段。描述写得越具体（如"构图评分（1-10，10 为最佳）"），AI 返回的结果越精准。这就是 StructuredOutput 的 Prompt 工程。

### 使用结构化评审：

```php
<?php

use App\Dto\ImageReview;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Image;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

// StructuredOutput 需要通过事件订阅器注册
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

$platform = PlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
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
        Image::fromFile('/tmp/product-hero-0.png'),
    ),
);

$result = $platform->invoke('gpt-4o', $messages, [
    'response_format' => ImageReview::class,
]);

/** @var ImageReview $review */
$review = $result->getResult()->getContent();

echo "=== 图片评审报告 ===\n";
echo "构图：{$review->compositionScore}/10\n";
echo "色彩：{$review->colorScore}/10\n";
echo "相关度：{$review->relevanceScore}/10\n";
echo "质量：{$review->qualityScore}/10\n";

$total = $review->compositionScore + $review->colorScore
    + $review->relevanceScore + $review->qualityScore;
echo "总分：{$total}/40\n\n";

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

## Step 4：自定义 `#[AsTool]` 工具

将图片生成和分析封装为 Agent 可调用的工具。`#[AsTool]` 属性支持 `name`（工具标识）、`description`（LLM 用来判断何时调用）和 `method`（实际执行的方法）。

```php
<?php

use Symfony\AI\Agent\Toolbox\Attribute\AsTool;
use Symfony\AI\Platform\Message\Content\Image;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\PlatformInterface;

// 自定义工具：生成图片
#[AsTool('generate_product_image', description: '根据提示词使用 DALL-E 3 生成商品宣传图片', method: 'generate')]
final class ImageGenerator
{
    public function __construct(
        private readonly PlatformInterface $platform,
    ) {
    }

    /**
     * @param string $prompt 详细的图片描述提示词
     * @param string $style  生成风格：natural（自然摄影）或 vivid（创意生动）
     * @param string $size   图片尺寸：1024x1024、1024x1792 或 1792x1024
     */
    public function generate(
        string $prompt,
        string $style = 'natural',
        string $size = '1024x1024',
    ): string {
        $response = $this->platform->invoke('dall-e-3', new MessageBag(
            Message::ofUser($prompt),
        ), [
            'size' => $size,
            'quality' => 'hd',
            'style' => $style,
            'response_format' => 'b64_json',
        ]);

        $imageResult = $response->getResult();
        $path = '/tmp/generated-' . time() . '.png';
        file_put_contents($path, base64_decode($imageResult->getContent()[0]->encodedImage));

        $info = "图片已生成：{$path}";
        if (null !== $imageResult->revisedPrompt) {
            $info .= "\nDALL-E 优化后的提示词：{$imageResult->revisedPrompt}";
        }

        return $info;
    }
}

// 自定义工具：分析图片
#[AsTool('analyze_product_image', description: '使用视觉 AI 从构图、色彩、质量、商业价值四个维度分析图片', method: 'analyze')]
final class ImageAnalyzer
{
    public function __construct(
        private readonly PlatformInterface $platform,
    ) {
    }

    /**
     * @param string $imagePath 待分析图片的文件路径
     */
    public function analyze(string $imagePath): string
    {
        $messages = new MessageBag(
            Message::ofUser(
                '请从构图、色彩、质量、商业价值四个维度详细分析这张商品图片，'
                . '并给出 1-10 的评分和改进建议：',
                Image::fromFile($imagePath),
            ),
        );

        $result = $this->platform->invoke('gpt-4o', $messages);

        return $result->getResult()->getContent();
    }
}
```

> **⚠️ 注意：** `#[AsTool]` 的 `description` 参数非常重要——LLM 依赖它来判断何时调用该工具。描述要清晰、具体，说明工具的功能和适用场景。工具方法的参数通过 PHP 类型声明和 PHPDoc 注释自动映射为 JSON Schema，LLM 会据此传递正确的参数。

---

## Step 5：Agent 编排图片处理工作流

使用 Agent 自动化整个流程：分析需求 → 优化 prompt → 生成图片 → 自动评审 → 给出建议。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;

// 注册工具到 Toolbox
$toolbox = new Toolbox([
    new ImageGenerator($platform),
    new ImageAnalyzer($platform),
]);

// AgentProcessor 同时处理输入（注入工具定义）和输出（拦截并执行工具调用）
$processor = new AgentProcessor($toolbox);

$agent = new Agent(
    $platform,
    'gpt-4o',
    [
        new SystemPromptInputProcessor(
            "你是电商视觉设计助手。你可以生成和分析商品图片。\n"
            . "工作流程：\n"
            . "1) 理解用户的商品需求和风格偏好\n"
            . "2) 优化提示词——加入专业的摄影术语、光线描述、构图指导\n"
            . "3) 调用 generate_product_image 生成图片\n"
            . "4) 调用 analyze_product_image 评审图片质量\n"
            . "5) 根据评审结果给出最终建议，必要时建议重新生成"
        ),
        $processor,
    ],
    [$processor],
);

// 一句话触发完整工作流
$result = $agent->call(new MessageBag(
    Message::ofUser(
        '帮我为一款智能手表生成一张电商首页 Banner 图，'
        . '要有科技感和高级感，适合放在天猫首页。'
    ),
));

echo $result->getContent() . "\n";
```

> **💡 提示：** `AgentProcessor` 同时实现了 `InputProcessorInterface` 和 `OutputProcessorInterface`。输入阶段注入工具定义让 LLM 知道有哪些工具可用，输出阶段拦截 LLM 的工具调用请求并执行。这就是为什么它同时出现在 Agent 构造函数的第三和第四个参数中。

---

## Step 6：图片对比与 A/B 测试

对比多个方案，利用多图输入能力选出最佳。

```php
<?php

use Symfony\AI\Platform\Message\Content\Image;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$messages = new MessageBag(
    Message::forSystem(
        '你是视觉设计总监。对比两张商品图，推荐更适合电商首页的方案。'
    ),
    Message::ofUser(
        '请对比以下两个方案，从吸引力、品牌调性、点击转化潜力三个角度分析：',
        Image::fromFile('/tmp/option-a.png'),
        Image::fromFile('/tmp/option-b.png'),
    ),
);

$result = $platform->invoke('gpt-4o', $messages);
echo "A/B 对比结果：\n" . $result->getResult()->getContent() . "\n";
```

> **💡 提示：** 一条消息可以传入多张图片，GPT-4o 会同时看到所有图片进行对比。这比分别分析再手动对比更可靠，因为 AI 能在相同上下文中直接比较差异。对比维度可以根据业务需求自定义，例如电商场景关注"点击转化"，品牌场景关注"调性一致性"。

---

## Step 7：使用 Gemini Imagen 生成图片（替代方案）

Google Gemini 的 `gemini-2.5-flash-image` 和 `gemini-3-pro-image-preview` 模型同时支持文本理解和图片生成（`OUTPUT_IMAGE` 能力），可以作为 DALL-E 的替代方案。

```php
<?php

use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory as GeminiPlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$geminiPlatform = GeminiPlatformFactory::create($_ENV['GEMINI_API_KEY']);

$messages = new MessageBag(
    Message::ofUser(
        '生成一张：极简设计的白色无线蓝牙耳机，放在大理石桌面上，'
        . '旁边有一杯拿铁咖啡，自然光照射，高级感摄影风格'
    ),
);

// Gemini 图片模型返回 BinaryResult
$response = $geminiPlatform->invoke('gemini-2.5-flash-image', $messages);

/** @var \Symfony\AI\Platform\Result\BinaryResult $binaryResult */
$binaryResult = $response->getResult();

// BinaryResult 提供便捷的文件保存方法
$binaryResult->asFile('/tmp/gemini-product.png');
echo "✅ Gemini 图片已保存：/tmp/gemini-product.png\n";
echo "MIME 类型：{$binaryResult->getMimeType()}\n";

// 也可以获取其他格式
$base64 = $binaryResult->toBase64();       // Base64 编码
$dataUri = $binaryResult->toDataUri();     // Data URI（可直接嵌入 HTML）
$binary = $binaryResult->getContent();     // 原始二进制数据
```

> **💡 提示：** Gemini 的图片生成模型同时具备 `INPUT_IMAGE` 和 `OUTPUT_IMAGE` 能力，意味着你可以传入一张参考图加上文字描述来生成新图——实现"以图生图"的编辑效果。`gemini-3-pro-image-preview` 还支持 `THINKING` 能力，在复杂创意场景下表现更好。

> **💡 提示：** 选择平台的参考因素——**DALL-E 3** 在遵循复杂提示词、文字渲染、商业摄影风格上更强，适合电商场景；**Gemini Imagen** 在以图生图编辑、多模态理解上有优势，且支持 `TOOL_CALLING`（`gemini-3-pro-image-preview`），可以更自然地集成到 Agent 工作流中。

---

## Step 8：生产环境最佳实践

### 速率限制与错误处理

```php
<?php

use Symfony\AI\Platform\Exception\RateLimitExceededException;

/**
 * @param int $maxRetries 最大重试次数
 * @param int $baseDelay  基础延迟（秒）
 */
function generateWithRetry(
    object $platform,
    MessageBag $messages,
    array $options,
    int $maxRetries = 3,
    int $baseDelay = 2,
): object {
    for ($attempt = 0; $attempt <= $maxRetries; ++$attempt) {
        try {
            return $platform->invoke('dall-e-3', $messages, $options);
        } catch (RateLimitExceededException) {
            if ($attempt === $maxRetries) {
                throw new \RuntimeException("已达最大重试次数（{$maxRetries}），图片生成失败。");
            }
            $delay = $baseDelay * (2 ** $attempt);
            sleep($delay);
        }
    }
}
```

### 图片存储与 CDN 分发

```php
<?php

// 生产环境：不要存到 /tmp，使用持久化存储
function saveGeneratedImage(string $imageData, string $productId): string
{
    $hash = substr(md5($imageData), 0, 12);
    $filename = "{$productId}/{$hash}.png";

    // 存储到对象存储（如 S3、OSS）
    // $filesystem->write($filename, $imageData);

    // 本地存储示例
    $dir = "/var/www/storage/generated/{$productId}";
    if (!is_dir($dir)) {
        mkdir($dir, 0o755, true);
    }
    file_put_contents("{$dir}/{$hash}.png", $imageData);

    // 返回 CDN URL
    return "https://cdn.example.com/generated/{$filename}";
}
```

> **🏭 生产建议：** 图片生成的成本管理要点：① DALL-E 3 `hd` 质量费用约为标准质量的 2 倍，批量生成时按需选择；② 每次 API 调用只生成 1 张图（DALL-E 3 的 `n` 参数仅支持 1），需要多方案时循环调用；③ 生成的图片建议立即持久化到对象存储并通过 CDN 分发，避免重复生成相同内容；④ 实现请求去重机制——对相同的 prompt 缓存生成结果。

> **🔒 安全建议：** ① 用户提交的图片描述需做内容审核，防止生成不当内容；② AI 生成的图片用于商业用途前，确认符合平台的使用条款；③ 图片存储路径要做安全校验，防止路径遍历攻击；④ 生产环境的 API 密钥要限制 IP 白名单和用量上限。

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `dall-e-3` 模型 | OpenAI 图片生成，支持 `size`、`quality`（`standard`/`hd`）、`style`（`natural`/`vivid`）参数 |
| `gemini-2.5-flash-image` | Gemini 图片生成模型，返回 `BinaryResult`，支持以图生图 |
| `Image::fromFile()` | 从本地文件加载图片，用于视觉分析或以图生图 |
| `ImageUrl` | 从 URL 引用图片，适合分析线上资源 |
| `Image::fromDataUrl()` | 从 Base64 Data URL 加载图片，适合前端直传场景 |
| `ImageResult` | DALL-E 返回结果，包含 `Base64Image`/`UrlImage` 列表和 `revisedPrompt` |
| `BinaryResult` | Gemini 图片返回结果，提供 `asFile()`、`toBase64()`、`toDataUri()` 方法 |
| `#[AsTool]` 属性 | 将类方法注册为 Agent 工具，`description` 决定 LLM 何时调用 |
| `StructuredOutput` | 通过 `PlatformSubscriber` + DTO 类让 AI 返回结构化 PHP 对象 |
| 多图输入 | 一条消息中传入多张 `Image`，AI 可同时分析对比 |
| `AgentProcessor` | 同时作为输入/输出处理器，注入工具定义并执行工具调用 |

## 下一步

如果你需要一个语音交互的 AI 助手（语音输入 → AI 处理 → 语音输出），请看 [20-voice-interactive-assistant.md](./20-voice-interactive-assistant.md)。

