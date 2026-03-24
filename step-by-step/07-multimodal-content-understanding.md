# 多模态内容理解

## 业务场景

你在做一个内容审核/处理平台。用户上传的内容不仅有文本，还有图片、PDF 文档、音频和视频文件。你需要 AI 理解这些不同类型的内容并给出分析——图片识别、文档 OCR、语音转文字、内容合规审核，甚至把文本转为语音。

**典型应用：** 图片内容识别、文档智能处理、音频转文字、发票/票据 OCR、内容审核、语音合成播报

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接支持多模态的 AI 平台（Azure OpenAI、Gemini、ElevenLabs 等） |
| **Content 类** | 封装不同类型的输入：`Image`、`Audio`、`Video`、`Document`、`Text`、`ImageUrl` |
| **Capability 枚举** | 声明模型支持的输入/输出能力（`INPUT_IMAGE`、`INPUT_AUDIO` 等） |
| **BinaryResult** | 处理二进制输出结果（TTS 音频、生成的图片等） |

> **⚠️ 注意：** 不是所有平台都支持所有模态。选择平台时请确认其 `Capability` 支持：
> - **图片输入（`INPUT_IMAGE`）：** Azure OpenAI、OpenAI、Anthropic、Gemini、VertexAI
> - **音频输入（`INPUT_AUDIO`）：** Gemini、VertexAI、ElevenLabs（STT）、Azure Whisper
> - **视频输入（`INPUT_VIDEO`）：** Gemini、VertexAI
> - **PDF 输入（`INPUT_PDF`）：** Gemini、VertexAI、Anthropic
> - **语音合成（`TEXT_TO_SPEECH`）：** ElevenLabs、OpenAI、Azure OpenAI

## 项目流程图

```
┌──────────────┐      ┌─────────────────┐      ┌───────────────────┐
│  用户上传内容   │ ──▶ │  Content 类封装    │ ──▶ │  构建 MessageBag    │
│ (图片/PDF/音频) │      │ Image / Audio /  │      │  (System + User    │
│               │      │ Document / Video │      │   + 多个 Content)   │
└──────────────┘      └─────────────────┘      └────────┬──────────┘
                                                         │
                                                         ▼
┌──────────────┐      ┌─────────────────┐      ┌───────────────────┐
│  业务处理结果   │ ◀── │  解析结果          │ ◀── │ Platform.invoke()  │
│ (审核/OCR/转写) │      │ asText() 文本回复  │      │  (Azure OpenAI /   │
│               │      │ asBinary() 二进制  │      │   Gemini / 11Labs) │
│               │      │ asFile() 保存文件  │      │                    │
└──────────────┘      └─────────────────┘      └───────────────────┘
```

## 前置准备

### 环境要求

- PHP >= 8.2
- Composer

### 安装依赖

本教程以 **Azure OpenAI** 为主要平台：

```bash
composer require symfony/ai-platform symfony/ai-azure-open-ai-platform
```

如果需要 PDF 处理（Gemini）或语音合成（ElevenLabs），按需安装：

```bash
# Gemini（支持 PDF、音频、视频输入）
composer require symfony/ai-gemini-platform

# ElevenLabs（语音合成 TTS / 语音识别 STT）
composer require symfony/ai-eleven-labs-platform
```

> **💡 提示：** 核心包 `symfony/ai-platform` 提供统一的 Content 类（`Image`、`Audio`、`Document` 等），各 Bridge 包负责与具体平台通信。切换平台只需替换 Bridge 包和工厂方法，业务代码无需修改。

### 设置 API 密钥

```bash
# Azure OpenAI
export AZURE_OPENAI_BASEURL="https://your-resource.openai.azure.com/"
export AZURE_OPENAI_DEPLOYMENT="gpt-4o"
export AZURE_OPENAI_API_VERSION="2024-12-01-preview"
export AZURE_OPENAI_KEY="your-azure-api-key"

# Gemini（可选）
export GEMINI_API_KEY="your-gemini-api-key"

# ElevenLabs（可选）
export ELEVEN_LABS_API_KEY="your-elevenlabs-api-key"
```

> **🔒 安全建议：** 永远不要将 API 密钥硬编码在源代码中。在 Symfony 项目中，应使用 `.env.local` 文件或服务器环境变量来管理密钥；在 CI/CD 中使用 Secrets 管理。

---

## Step 1：创建 Azure OpenAI Platform 实例

Azure OpenAI 的工厂方法需要提供资源 URL、部署名称、API 版本和密钥：

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\Azure\OpenAi\PlatformFactory;

$platform = PlatformFactory::create(
    baseUrl: $_ENV['AZURE_OPENAI_BASEURL'],
    deployment: $_ENV['AZURE_OPENAI_DEPLOYMENT'],
    apiVersion: $_ENV['AZURE_OPENAI_API_VERSION'],
    apiKey: $_ENV['AZURE_OPENAI_KEY'],
);
```

> **💡 提示：** Azure OpenAI 的 `PlatformFactory::create()` 也支持传入自定义 `HttpClientInterface`、`ModelCatalogInterface` 和 `EventDispatcherInterface`。在 Symfony 项目中推荐通过 AI Bundle 依赖注入自动配置。

> **🏭 生产建议：** Azure OpenAI 相比直接使用 OpenAI API，可在企业网络内部署，支持 VNet 私有端点、RBAC 权限控制和数据驻留策略，更适合有合规要求的生产环境。

---

## Step 2：图片理解——多种加载方式

让 AI 分析图片内容。Symfony AI 提供三种方式加载图片：

### 从本地文件加载：

```php
use Symfony\AI\Platform\Message\Content\Image;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// Image::fromFile() 会自动检测 MIME 类型，并延迟读取文件内容
$messages = new MessageBag(
    Message::forSystem('你是一个专业的图片分析师。请用中文描述图片内容。'),
    Message::ofUser(
        '请描述这张图片的内容。',
        Image::fromFile('/path/to/photo.jpg'),
    ),
);

$result = $platform->invoke('gpt-4o', $messages);
echo $result->asText() . "\n";
```

> **💡 提示：** `Image::fromFile()` 使用惰性加载——文件内容仅在实际发送请求时才读取，不会在构造时占用内存。`Audio::fromFile()`、`Video::fromFile()`、`Document::fromFile()` 同理。

### 从 URL 加载：

使用 `ImageUrl` 类引用远程图片，避免传输图片二进制数据：

```php
use Symfony\AI\Platform\Message\Content\ImageUrl;

$messages = new MessageBag(
    Message::ofUser(
        '这张图片里有什么？',
        new ImageUrl('https://example.com/product-photo.jpg'),
    ),
);

$result = $platform->invoke('gpt-4o', $messages);
echo $result->asText() . "\n";
```

> **🏭 生产建议：** 优先使用 `ImageUrl` 传递公网可访问的图片 URL，而非通过 `Image::fromFile()` 上传二进制数据。URL 方式减少请求体积、降低网络传输耗时，特别适合图片已托管在 CDN 或 OSS 的场景。

### 从 Data URL 加载：

适合处理前端上传的 Base64 图片数据或从数据库读取的图片：

```php
// 从 Data URL 格式加载（如前端 Canvas 导出的图片）
$dataUrl = 'data:image/jpeg;base64,' . base64_encode(file_get_contents('/path/to/photo.jpg'));

$messages = new MessageBag(
    Message::ofUser(
        '分析这张图片。',
        Image::fromDataUrl($dataUrl),
    ),
);
```

> **⚠️ 注意：** Base64 编码会使数据体积增大约 33%。大尺寸图片建议先压缩或缩放后再传输。OpenAI/Azure OpenAI 建议单张图片不超过 20MB，推荐分辨率不超过 2048×2048。

---

## Step 3：多内容混合消息

`Message::ofUser()` 支持同时传入文本和多个媒体内容，实现真正的多模态分析。

### 场景 A：商品图片质量审核

```php
$messages = new MessageBag(
    Message::forSystem(
        '你是电商平台的图片审核员。检查商品图片是否符合以下标准：'
        . '1) 图片清晰不模糊；2) 主体突出无遮挡；3) 背景干净；'
        . '4) 无水印或文字遮挡；5) 无不当内容。'
        . '返回审核结果：通过/不通过，以及原因。'
    ),
    Message::ofUser(
        '请审核这张商品图片。',
        Image::fromFile('/path/to/product-image.jpg'),
    ),
);

$result = $platform->invoke('gpt-4o', $messages);
echo "审核结果：" . $result->asText() . "\n";
```

### 场景 B：多张图片对比

```php
// 一条消息中传入多张图片 + 文字说明
$messages = new MessageBag(
    Message::ofUser(
        '请对比这两张产品图片，列出它们之间的区别。',
        Image::fromFile('/path/to/product-v1.jpg'),
        Image::fromFile('/path/to/product-v2.jpg'),
    ),
);

$result = $platform->invoke('gpt-4o', $messages);
echo $result->asText() . "\n";
```

### 场景 C：图文 + 文档混合分析

```php
use Symfony\AI\Platform\Message\Content\Document;

// 同时传入文字描述、产品图片和规格文档
$messages = new MessageBag(
    Message::forSystem('你是一个产品合规审查专家。对比产品实物与规格文档，指出任何不一致之处。'),
    Message::ofUser(
        '请对比实物照片与产品规格书，检查是否一致。',
        Image::fromFile('/path/to/product-photo.jpg'),
        Document::fromFile('/path/to/spec-sheet.pdf'),
    ),
);

// PDF 输入需使用支持 INPUT_PDF 的平台（如 Gemini）
$geminiPlatform = \Symfony\AI\Platform\Bridge\Gemini\PlatformFactory::create(
    $_ENV['GEMINI_API_KEY'],
);
$result = $geminiPlatform->invoke('gemini-2.5-flash', $messages);
echo $result->asText() . "\n";
```

> **💡 提示：** `Message::ofUser()` 可接收任意数量的 `ContentInterface` 参数，你可以自由组合文本、图片、文档、音频等内容。AI 会综合理解所有输入。

---

## Step 4：PDF 文档处理与 OCR

### 文档总结

让 AI 直接阅读和理解 PDF 文档：

```php
<?php

use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Document;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['GEMINI_API_KEY']);

$messages = new MessageBag(
    Message::forSystem('你是一个文档分析助手。帮用户理解和总结文档内容。'),
    Message::ofUser(
        Document::fromFile('/path/to/contract.pdf'),
        '请总结这份合同的要点：1) 甲乙双方 2) 合同金额 3) 关键条款 4) 有效期',
    ),
);

$result = $platform->invoke('gemini-2.5-flash', $messages);
echo $result->asText() . "\n";
```

> **⚠️ 注意：** PDF 输入目前仅部分平台支持。Gemini / VertexAI 支持 `INPUT_PDF` 能力，Anthropic 也支持。Azure OpenAI 和标准 OpenAI 暂不支持直接 PDF 输入——需先将 PDF 转为图片或提取文本后再传入。

### 发票 OCR 提取（结合结构化输出）

结合结构化输出，可以从发票 PDF 中提取关键信息：

```php
// 定义发票数据结构
class Invoice
{
    /**
     * @param string  $invoiceNumber 发票号码
     * @param string  $date          开票日期
     * @param string  $seller        销售方名称
     * @param string  $buyer         购买方名称
     * @param float   $totalAmount   总金额
     * @param float   $taxAmount     税额
     * @param string  $currency      币种
     */
    public function __construct(
        public readonly string $invoiceNumber,
        public readonly string $date,
        public readonly string $seller,
        public readonly string $buyer,
        public readonly float $totalAmount,
        public readonly float $taxAmount,
        public readonly string $currency = 'CNY',
    ) {
    }
}

// 需要注册 PlatformSubscriber（参见 06-structured-data-extraction.md）
$messages = new MessageBag(
    Message::forSystem('从发票中提取关键信息。'),
    Message::ofUser(
        Document::fromFile('/path/to/invoice.pdf'),
    ),
);

$result = $platform->invoke('gemini-2.5-flash', $messages, [
    'response_format' => Invoice::class,
]);

$invoice = $result->asObject();
echo "发票号：{$invoice->invoiceNumber}\n";
echo "金额：{$invoice->totalAmount} {$invoice->currency}\n";
```

> **🏭 生产建议：** 大批量 PDF 处理时，建议控制并发数量并实现队列化。单个 PDF 的 Token 消耗与页数成正比——一份 50 页的文档可能消耗数万 Token。考虑先拆分页面，仅处理需要的部分。

---

## Step 5：音频处理——语音转文字（STT）与文字转语音（TTS）

### 语音转文字（STT）

将音频文件转为文字。适用于会议录音转文字、客服通话分析等。

**使用 Azure Whisper：**

```php
<?php

use Symfony\AI\Platform\Bridge\Azure\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Audio;

// 创建专门指向 Whisper 部署的 Platform 实例
$whisperPlatform = PlatformFactory::create(
    baseUrl: $_ENV['AZURE_OPENAI_BASEURL'],
    deployment: $_ENV['AZURE_OPENAI_WHISPER_DEPLOYMENT'],
    apiVersion: $_ENV['AZURE_OPENAI_WHISPER_API_VERSION'],
    apiKey: $_ENV['AZURE_OPENAI_KEY'],
);

$result = $whisperPlatform->invoke('whisper-1', Audio::fromFile('/path/to/meeting-recording.mp3'));
echo "会议内容：\n" . $result->asText() . "\n";
```

> **⚠️ 注意：** Azure Whisper 的音频文件大小限制为 25MB。支持格式：mp3、mp4、mpeg、mpga、m4a、wav、webm。超过限制时需先分割音频文件。

**使用 ElevenLabs：**

```php
use Symfony\AI\Platform\Bridge\ElevenLabs\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Audio;

$platform = PlatformFactory::create(
    apiKey: $_ENV['ELEVEN_LABS_API_KEY'],
);

$result = $platform->invoke('scribe_v1', Audio::fromFile('/path/to/meeting-recording.mp3'));
echo "会议内容：\n" . $result->asText() . "\n";
```

### 文字转语音（TTS）与二进制结果处理

将文本转为语音，适用于内容播报、语音通知等。TTS 输出为二进制音频数据，通过 `asBinary()` 或 `asFile()` 获取。

```php
<?php

use Symfony\AI\Platform\Bridge\ElevenLabs\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Text;

$platform = PlatformFactory::create(
    apiKey: $_ENV['ELEVEN_LABS_API_KEY'],
);

$text = new Text('欢迎使用我们的产品。如果您有任何问题，请随时联系我们的客服团队。');

$result = $platform->invoke('eleven_multilingual_v2', $text, [
    'voice' => 'pqHfZKP75CvOlQylNhV4',
]);

// 方式一：获取二进制数据后自行处理
$audioData = $result->asBinary();
file_put_contents('/path/to/output.mp3', $audioData);

// 方式二：直接保存到文件（推荐，内置目录和权限检查）
$result->asFile('/path/to/output.mp3');

echo "音频已生成\n";
```

> **💡 提示：** `asBinary()` 返回原始二进制字符串，适合进一步处理（如流式传输给客户端）。`asFile()` 封装了目录存在性和可写权限检查，生产环境中推荐使用。

> **🏭 生产建议：** TTS 结果适合缓存——相同文本 + 相同声音的输出是确定性的。建议用文本内容的哈希值作为缓存键，避免重复调用 API。对于高频播报场景（如固定欢迎语），可预生成并存入 CDN。

---

## Step 6：视频内容理解

Gemini / VertexAI 支持视频输入（`INPUT_VIDEO`），可以分析视频内容：

```php
<?php

use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Video;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['GEMINI_API_KEY']);

$messages = new MessageBag(
    Message::forSystem('你是一个视频内容审核员。分析视频内容并判断是否合规。'),
    Message::ofUser(
        '请审核这段用户上传的短视频，描述视频内容并判断是否合规。',
        Video::fromFile('/path/to/user-upload.mp4'),
    ),
);

$result = $platform->invoke('gemini-2.5-flash', $messages);
echo $result->asText() . "\n";
```

> **⚠️ 注意：** 视频输入目前仅 Gemini 和 VertexAI 支持。视频文件会消耗大量 Token（每秒视频约等于多帧图片的 Token 消耗）。建议限制视频时长，或先截取关键帧后用图片方式分析。

---

## Step 7：模型能力检查与错误处理

在处理用户上传的混合内容时，并非所有模型都支持所有模态。Symfony AI 通过 `Capability` 枚举和 `MissingModelSupportException` 提供了能力检查机制。

### Capability 枚举

```php
use Symfony\AI\Platform\Capability;

// 多模态相关的 Capability 枚举值：
// 输入能力
Capability::INPUT_IMAGE       // 图片输入
Capability::INPUT_AUDIO       // 音频输入
Capability::INPUT_VIDEO       // 视频输入
Capability::INPUT_PDF         // PDF 文档输入
Capability::INPUT_TEXT        // 文本输入
Capability::INPUT_MULTIMODAL  // 多模态输入
Capability::INPUT_MULTIPLE    // 多内容输入

// 输出能力
Capability::OUTPUT_TEXT       // 文本输出
Capability::OUTPUT_IMAGE      // 图片输出
Capability::OUTPUT_AUDIO      // 音频输出

// 专项能力
Capability::TEXT_TO_SPEECH    // 文字转语音
Capability::SPEECH_TO_TEXT    // 语音转文字
Capability::TEXT_TO_IMAGE     // 文字生成图片
```

### 模型能力检查

```php
use Symfony\AI\Platform\Model;
use Symfony\AI\Platform\Capability;

// Model 实例提供 supports() 方法检查能力
$model = new Model('gpt-4o', [
    Capability::INPUT_IMAGE,
    Capability::INPUT_TEXT,
    Capability::OUTPUT_TEXT,
]);

if ($model->supports(Capability::INPUT_IMAGE)) {
    echo "该模型支持图片输入\n";
}

if (!$model->supports(Capability::INPUT_VIDEO)) {
    echo "该模型不支持视频输入\n";
}
```

### 异常处理

当模型不支持某种输入时，平台会抛出 `MissingModelSupportException`：

```php
use Symfony\AI\Platform\Exception\MissingModelSupportException;

try {
    $result = $platform->invoke('gpt-4o', $messages);
    echo $result->asText() . "\n";
} catch (MissingModelSupportException $e) {
    // 例如："Model "gpt-4o" does not support "audio input"."
    echo "模型能力不匹配：" . $e->getMessage() . "\n";
    // 可以在这里切换到支持该能力的平台/模型
}
```

> **🏭 生产建议：** 在内容审核流水线中，建议根据上传内容的 MIME 类型动态选择模型。例如：图片/文本用 Azure OpenAI，PDF 用 Gemini，音频用 ElevenLabs 或 Azure Whisper。避免将不支持的内容类型发送给错误的模型。

---

## 完整示例：智能内容审核流水线

模拟一个内容平台，对用户上传的不同类型内容进行自动审核，根据内容类型选择合适的平台。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\Azure\OpenAi\PlatformFactory as AzurePlatformFactory;
use Symfony\AI\Platform\Exception\MissingModelSupportException;
use Symfony\AI\Platform\Message\Content\Image;
use Symfony\AI\Platform\Message\Content\ImageUrl;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// === 创建 Azure OpenAI Platform ===
$platform = AzurePlatformFactory::create(
    baseUrl: $_ENV['AZURE_OPENAI_BASEURL'],
    deployment: $_ENV['AZURE_OPENAI_DEPLOYMENT'],
    apiVersion: $_ENV['AZURE_OPENAI_API_VERSION'],
    apiKey: $_ENV['AZURE_OPENAI_KEY'],
);

$reviewPrompt = '你是内容审核员。对用户上传的内容进行审核。'
    . '判断内容是否合规（不含暴力、色情、仇恨言论、虚假信息等）。'
    . '返回 JSON 格式：{"status": "通过|不通过", "reason": "原因简述"}';

// === 审核场景 1：纯文本内容 ===
echo "=== 文本审核 ===\n";
$messages = new MessageBag(
    Message::forSystem($reviewPrompt),
    Message::ofUser('用户发布的文章：今天天气真好，和家人一起去公园野餐了，非常开心。'),
);
$result = $platform->invoke('gpt-4o', $messages);
echo $result->asText() . "\n\n";

// === 审核场景 2：图片内容（本地文件）===
echo "=== 图片审核（本地文件）===\n";
$messages = new MessageBag(
    Message::forSystem($reviewPrompt),
    Message::ofUser(
        '用户上传了一张头像图片，请审核。',
        Image::fromFile('/path/to/avatar.jpg'),
    ),
);
$result = $platform->invoke('gpt-4o', $messages);
echo $result->asText() . "\n\n";

// === 审核场景 3：图片内容（URL 引用）===
echo "=== 图片审核（URL）===\n";
$messages = new MessageBag(
    Message::forSystem($reviewPrompt),
    Message::ofUser(
        '用户引用了一张网络图片，请审核。',
        new ImageUrl('https://example.com/user-uploaded-image.jpg'),
    ),
);
$result = $platform->invoke('gpt-4o', $messages);
echo $result->asText() . "\n\n";

// === 审核场景 4：文本 + 多张图片组合 ===
echo "=== 图文组合审核 ===\n";
$messages = new MessageBag(
    Message::forSystem($reviewPrompt),
    Message::ofUser(
        '用户发布的内容如下，请同时审核文字和所有配图：'
        . "\n文字：今天去了一家新开的餐厅，强烈推荐！",
        Image::fromFile('/path/to/restaurant-photo-1.jpg'),
        Image::fromFile('/path/to/restaurant-photo-2.jpg'),
    ),
);

try {
    $result = $platform->invoke('gpt-4o', $messages);
    echo $result->asText() . "\n";
} catch (MissingModelSupportException $e) {
    echo "审核失败：" . $e->getMessage() . "\n";
}
```

---

## 其他实现方案

Symfony AI 的 Bridge 架构让切换平台非常简单——业务代码（`MessageBag`、`invoke()`、`asText()`）完全一致，只需替换 `PlatformFactory` 和模型名称。

### 方案 A：使用标准 OpenAI

```bash
composer require symfony/ai-open-ai-platform
```

```php
<?php

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Image;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 仅需更换工厂类和 API 密钥
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);

$messages = new MessageBag(
    Message::forSystem('你是一个专业的图片分析师。请用中文描述图片内容。'),
    Message::ofUser(
        '请描述这张图片的内容。',
        Image::fromFile('/path/to/photo.jpg'),
    ),
);

$result = $platform->invoke('gpt-4o-mini', $messages);
echo $result->asText() . "\n";
```

### 方案 B：使用 Google Gemini（多模态最全面）

Gemini 是目前支持模态最全面的平台——图片、音频、视频、PDF 均支持：

```bash
composer require symfony/ai-gemini-platform
```

```php
<?php

use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Audio;
use Symfony\AI\Platform\Message\Content\Document;
use Symfony\AI\Platform\Message\Content\Image;
use Symfony\AI\Platform\Message\Content\Video;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['GEMINI_API_KEY']);

// Gemini 可以在一条消息中同时处理图片 + PDF + 音频
$messages = new MessageBag(
    Message::forSystem('你是一个多模态内容分析师。综合分析所有输入内容并给出报告。'),
    Message::ofUser(
        '请综合分析以下内容：一张产品图片、一份规格文档和一段客户反馈录音。',
        Image::fromFile('/path/to/product.jpg'),
        Document::fromFile('/path/to/spec.pdf'),
        Audio::fromFile('/path/to/feedback.mp3'),
    ),
);

$result = $platform->invoke('gemini-2.5-flash', $messages);
echo $result->asText() . "\n";
```

> **💡 提示：** 对比三种方案——除了 `use` 语句、API 密钥和模型名称不同，其余代码（`MessageBag`、`Message::ofUser()`、`invoke()`、`asText()`）完全一致。这就是 Symfony AI Platform 抽象层的价值。

### 各平台多模态能力对比

| 能力 | Azure OpenAI | OpenAI | Gemini | Anthropic | ElevenLabs |
|------|:---:|:---:|:---:|:---:|:---:|
| 图片输入 | ✅ | ✅ | ✅ | ✅ | ❌ |
| 音频输入 | ✅（Whisper） | ✅（Whisper） | ✅ | ❌ | ✅（STT） |
| 视频输入 | ❌ | ❌ | ✅ | ❌ | ❌ |
| PDF 输入 | ❌ | ❌ | ✅ | ✅ | ❌ |
| 语音合成 | ✅ | ✅ | ❌ | ❌ | ✅ |

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Image::fromFile()` | 从本地文件加载图片（惰性读取，自动检测 MIME 类型） |
| `ImageUrl` | 从 URL 引用远程图片（`new ImageUrl($url)`） |
| `Image::fromDataUrl()` | 从 Data URL 格式加载图片（Base64 编码数据） |
| `Audio::fromFile()` | 从本地文件加载音频 |
| `Video::fromFile()` | 从本地文件加载视频 |
| `Document::fromFile()` | 从本地文件加载 PDF 文档 |
| `Text` | 纯文本内容封装（用于 TTS 等） |
| 混合消息 | `Message::ofUser()` 可同时传入文本和多个 `ContentInterface` |
| `$result->asBinary()` | 获取二进制输出数据（音频、图片等） |
| `$result->asFile()` | 将二进制输出直接保存为文件（带权限检查） |
| `Capability` 枚举 | 声明模型能力：`INPUT_IMAGE`、`INPUT_AUDIO`、`TEXT_TO_SPEECH` 等 |
| `Model::supports()` | 检查模型是否支持某种能力 |
| `MissingModelSupportException` | 模型不支持所请求的输入/输出时抛出的异常 |

## 下一步

如果你的 AI 助手需要记住用户的偏好和历史信息（不仅是当前对话），需要长期记忆能力。请看 [08-agent-with-memory.md](./08-agent-with-memory.md)。
