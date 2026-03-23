# 多模态内容理解

## 业务场景

你在做一个内容审核/处理平台。用户上传的内容不仅有文本，还有图片、PDF 文档、音频文件。你需要 AI 理解这些不同类型的内容并给出分析。

**典型应用：** 图片内容识别、文档智能处理、音频转文字、发票/票据 OCR、内容审核

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接支持多模态的 AI 平台 |
| **Message Content 类** | 封装不同类型的输入（Image、Audio、Document） |

> **注意：** 不是所有平台都支持所有模态。
> - **图片输入：** OpenAI、Anthropic、Gemini、VertexAI 等
> - **音频输入：** Gemini、VertexAI、ElevenLabs、Azure Whisper
> - **PDF 输入：** Gemini、VertexAI、Anthropic

---

## Step 1：图片理解

让 AI 分析图片内容。图片可以从本地文件或 URL 加载。

### 从本地文件加载图片：

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Image;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());

// 从本地文件加载图片
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

### 从 URL 加载图片：

```php
$messages = new MessageBag(
    Message::ofUser(
        '这张图片里有什么？',
        Image::fromUrl('https://example.com/product-photo.jpg'),
    ),
);

$result = $platform->invoke('gpt-4o-mini', $messages);
echo $result->asText() . "\n";
```

### 从 Base64 数据加载：

```php
// 适合从数据库或 API 获取的图片数据
$imageData = base64_encode(file_get_contents('/path/to/photo.jpg'));

$messages = new MessageBag(
    Message::ofUser(
        '分析这张图片。',
        Image::fromBase64($imageData, 'image/jpeg'),
    ),
);
```

---

## Step 2：图片分析的业务场景

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

$result = $platform->invoke('gpt-4o-mini', $messages);
echo "审核结果：" . $result->asText() . "\n";
```

### 场景 B：多张图片对比

```php
// 可以在一条消息中传入多张图片
$messages = new MessageBag(
    Message::ofUser(
        '请对比这两张产品图片，列出它们之间的区别。',
        Image::fromFile('/path/to/product-v1.jpg'),
        Image::fromFile('/path/to/product-v2.jpg'),
    ),
);

$result = $platform->invoke('gpt-4o-mini', $messages);
echo $result->asText() . "\n";
```

---

## Step 3：PDF 文档处理

让 AI 直接阅读和理解 PDF 文档（需要 Gemini 或 Anthropic 等支持 PDF 输入的平台）。

```php
<?php

use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Document;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['GEMINI_API_KEY'], HttpClient::create());

// 加载 PDF 文档
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

### 发票 OCR 提取

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

---

## Step 4：音频处理

### 语音转文字（STT）

将音频文件转为文字。适用于会议录音转文字、客服通话分析等。

```php
<?php

// 方式一：使用 ElevenLabs
use Symfony\AI\Platform\Bridge\ElevenLabs\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Audio;

$platform = PlatformFactory::create(
    apiKey: $_ENV['ELEVEN_LABS_API_KEY'],
    httpClient: HttpClient::create(),
);

$result = $platform->invoke('scribe_v1', Audio::fromFile('/path/to/meeting-recording.mp3'));
echo "会议内容：\n" . $result->asText() . "\n";
```

```php
// 方式二：使用 Azure Whisper
use Symfony\AI\Platform\Bridge\Azure\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Audio;

$platform = PlatformFactory::create(
    baseUrl: $_ENV['AZURE_OPENAI_BASEURL'],
    deployment: $_ENV['AZURE_OPENAI_WHISPER_DEPLOYMENT'],
    apiVersion: $_ENV['AZURE_OPENAI_WHISPER_API_VERSION'],
    apiKey: $_ENV['AZURE_OPENAI_KEY'],
    httpClient: HttpClient::create(),
);

$result = $platform->invoke('whisper-1', Audio::fromFile('/path/to/recording.mp3'));
echo $result->asText() . "\n";
```

```php
// 方式三：使用 Gemini（音频理解，不仅是转文字）
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Audio;

$platform = PlatformFactory::create($_ENV['GEMINI_API_KEY'], HttpClient::create());

$messages = new MessageBag(
    Message::ofUser(
        '这段录音在说什么？请总结主要内容。',
        Audio::fromFile('/path/to/podcast.mp3'),
    ),
);

$result = $platform->invoke('gemini-2.5-flash', $messages);
echo $result->asText() . "\n";
```

### 文字转语音（TTS）

将文本转为语音，适用于内容播报、语音通知等。

```php
<?php

use Symfony\AI\Platform\Bridge\ElevenLabs\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Text;

$platform = PlatformFactory::create(
    apiKey: $_ENV['ELEVEN_LABS_API_KEY'],
    httpClient: HttpClient::create(),
);

$text = new Text('欢迎使用我们的产品。如果您有任何问题，请随时联系我们的客服团队。');

$result = $platform->invoke('eleven_multilingual_v2', $text, [
    'voice' => 'pqHfZKP75CvOlQylNhV4', // 选择声音
]);

// 获取音频二进制数据
$audioData = $result->asBinary();

// 保存为文件
file_put_contents('/path/to/output.mp3', $audioData);
echo "音频已生成\n";
```

---

## 完整示例：智能内容审核流水线

模拟一个内容平台，对用户上传的不同类型内容进行自动审核。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Image;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());

$reviewPrompt = '你是内容审核员。对用户上传的内容进行审核。'
    . '判断内容是否合规（不含暴力、色情、仇恨言论、虚假信息等）。'
    . '返回格式：[通过/不通过] - 原因简述';

// === 审核场景 1：纯文本内容 ===
echo "=== 文本审核 ===\n";
$messages = new MessageBag(
    Message::forSystem($reviewPrompt),
    Message::ofUser('用户发布的文章：今天天气真好，和家人一起去公园野餐了，非常开心。'),
);
$result = $platform->invoke('gpt-4o-mini', $messages);
echo $result->asText() . "\n\n";

// === 审核场景 2：图片内容 ===
echo "=== 图片审核 ===\n";
$messages = new MessageBag(
    Message::forSystem($reviewPrompt),
    Message::ofUser(
        '用户上传了一张头像图片，请审核。',
        Image::fromFile('/path/to/avatar.jpg'),
    ),
);
$result = $platform->invoke('gpt-4o-mini', $messages);
echo $result->asText() . "\n\n";

// === 审核场景 3：文本 + 图片组合 ===
echo "=== 图文组合审核 ===\n";
$messages = new MessageBag(
    Message::forSystem($reviewPrompt),
    Message::ofUser(
        '用户发布的内容如下，请同时审核文字和配图：\n文字：今天去了一家新开的餐厅，推荐！',
        Image::fromFile('/path/to/restaurant-photo.jpg'),
    ),
);
$result = $platform->invoke('gpt-4o-mini', $messages);
echo $result->asText() . "\n";
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Image::fromFile()` | 从本地文件加载图片 |
| `Image::fromUrl()` | 从 URL 加载图片 |
| `Image::fromBase64()` | 从 Base64 数据加载图片 |
| `Audio::fromFile()` | 从本地文件加载音频 |
| `Document::fromFile()` | 从本地文件加载 PDF 文档 |
| `Text` | 纯文本内容（用于 TTS 等） |
| 混合消息 | `Message::ofUser()` 可同时传入文本和多个媒体内容 |
| `asBinary()` | 获取二进制输出（音频、图片生成等） |

## 下一步

如果你的 AI 助手需要记住用户的偏好和历史信息（不仅是当前对话），需要长期记忆能力。请看 [08-agent-with-memory.md](./08-agent-with-memory.md)。
