# 语音交互 AI 助手

## 业务场景

你在做一个智能客服电话系统或者语音助手 App。用户通过语音说出问题，系统自动把语音转成文字（STT），交给 AI 理解并处理（可以调用工具查询信息），然后把回答转成语音（TTS）播放给用户。整个流程：**语音输入 → AI 思考 → 语音输出**。

**典型应用：** 智能电话客服、语音助手、无障碍工具、语音导航、播客内容生成

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台，统一接口 |
| **Bridge（Anthropic）** | 主 LLM，Claude 处理用户意图 |
| **Bridge（ElevenLabs）** | STT（语音转文字）和 TTS（文字转语音） |
| **Bridge（Cartesia）** | 备选 TTS 供应商 |
| **Audio Content** | `Audio::fromFile()` 加载音频文件 |
| **Agent + #[AsTool]** | 智能体 + 自定义业务工具 |

## 项目流程图

```
🎤 用户语音
    │
    ▼
[ElevenLabs scribe_v1 STT] → 文字
    │
    ▼
[Anthropic Claude Agent] → 理解意图 + 调用 #[AsTool] 工具
    │
    ▼
[ElevenLabs eleven_multilingual_v2 TTS] → 语音回复（支持流式）
    │
    ▼
🔊 播放给用户
```

## 前置准备

### 环境要求

- PHP >= 8.2
- Composer

### 安装依赖

```bash
composer require symfony/ai-platform symfony/ai-anthropic-platform
composer require symfony/ai-eleven-labs-platform   # 语音能力（STT + TTS）
composer require symfony/ai-agent
```

> **💡 提示：** 本教程使用 Anthropic Claude 作为主 LLM。如果你更熟悉 OpenAI，可以替换为 `symfony/ai-open-ai-platform`，业务代码几乎不需要修改——这就是 Symfony AI Bridge 架构的优势。

### 设置 API 密钥

```bash
export ANTHROPIC_API_KEY="your-anthropic-api-key"
export ELEVENLABS_API_KEY="your-elevenlabs-api-key"
```

> **🔒 安全建议：** 永远不要将 API 密钥硬编码在源代码中。在 Symfony 项目中，应使用 `.env.local` 文件或服务器环境变量来管理密钥。

---

## Step 1：语音转文字（Speech-to-Text）

把用户说的话转成文字。ElevenLabs 的 `scribe_v1` 模型支持多语言自动识别。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\ElevenLabs\PlatformFactory as ElevenLabsFactory;
use Symfony\AI\Platform\Message\Content\Audio;

// 创建 ElevenLabs 平台实例
$elevenLabs = ElevenLabsFactory::create(apiKey: $_ENV['ELEVENLABS_API_KEY']);

// 通过 Audio::fromFile() 加载用户的语音文件
$audio = Audio::fromFile('/path/to/user-voice.mp3');

// 使用 scribe_v1 模型进行语音转文字
$result = $elevenLabs->invoke('scribe_v1', $audio);

$userText = $result->asText();
echo "用户说的话：{$userText}\n";
```

`Audio::fromFile()` 会自动检测文件的 MIME 类型，支持常见音频格式。

> **💡 提示：音频格式选择很重要。** 不同格式在文件大小和转录质量之间有取舍：
> - **mp3**：文件小，适合网络传输，大多数场景首选
> - **wav**：无损音质，转录准确率最高，适合对质量要求极高的场景
> - **ogg/webm**：浏览器录音默认格式，适合 Web 应用
> - **flac**：无损压缩，兼顾质量与文件大小
>
> 一般推荐使用 **mp3（128kbps 以上）** 或 **wav（16kHz, 16bit）**，在质量和传输效率之间取得平衡。

---

## Step 2：文字转语音（Text-to-Speech）

把 AI 的文字回复转成自然的语音。`eleven_multilingual_v2` 支持 29 种语言，包括中文。

```php
<?php

use Symfony\AI\Platform\Message\Content\Text;

$replyText = '好的，我已经帮您查询了订单状态。您的包裹目前在杭州转运中心，预计明天下午送达。';

// 使用 eleven_multilingual_v2 模型转为语音
$result = $elevenLabs->invoke(
    'eleven_multilingual_v2',
    new Text($replyText),
    [
        'voice' => 'pqHfZKP75CvOlQylNhV4',  // ElevenLabs 预置语音 ID
    ],
);

// 获取生成的音频二进制数据并保存
$audioData = $result->asBinary();
file_put_contents('/tmp/reply.mp3', $audioData);
echo "✅ 语音回复已生成：/tmp/reply.mp3\n";
```

> **💡 提示：** ElevenLabs 提供多种预置语音，也支持自定义语音克隆。`voice` 参数填写语音 ID，可在 ElevenLabs 控制台的 Voice Library 中查看。如果需要更低延迟的 TTS，可以使用 `eleven_flash_v2_5` 模型——速度更快，适合实时对话场景。

---

## Step 3：流式语音输出（Streaming TTS）

对于需要实时播放的场景（如电话客服），使用流式 TTS 可以大幅降低首字节延迟。

```php
<?php

// 开启流式 TTS — 边生成边传输
$result = $elevenLabs->invoke(
    'eleven_multilingual_v2',
    new Text('欢迎致电智能客服系统。请问有什么可以帮到您？'),
    [
        'voice' => 'pqHfZKP75CvOlQylNhV4',
        'stream' => true,  // 开启流式输出
    ],
);

// 实时处理音频流
$outputFile = fopen('/tmp/stream-reply.mp3', 'wb');
foreach ($result->asStream() as $chunk) {
    fwrite($outputFile, $chunk);
    // 在实际应用中，这里可以实时推送给 WebSocket 客户端或播放器
}
fclose($outputFile);
echo "✅ 流式语音已生成\n";
```

> **⚡ 延迟优化：** 流式 TTS 的核心优势是**首字节延迟低**——用户不需要等整段语音生成完毕就能开始听到回复。在电话系统中，这可以让响应时间从 2-3 秒降低到 300-500 毫秒。搭配 `eleven_flash_v2_5` 模型效果更好。

---

## Step 4：自定义业务工具（#[AsTool]）

语音助手的核心价值在于能执行实际业务操作。通过 `#[AsTool]` 属性定义自定义工具，让 AI Agent 能够查询订单、预约服务等。

```php
<?php

use Symfony\AI\Agent\Toolbox\Attribute\AsTool;

// 工具 1：查询订单物流
#[AsTool('query_order', description: '根据订单号查询物流状态')]
final class OrderQueryTool
{
    /**
     * @param string $orderNumber 用户提供的订单号
     */
    public function __invoke(string $orderNumber): string
    {
        // 实际项目中这里调用数据库或物流 API
        $orders = [
            'ORD-2025-001' => '已发货 - 杭州转运中心 - 预计明天到达',
            'ORD-2025-002' => '已签收 - 2025-01-18 14:30',
            'ORD-2025-003' => '待发货 - 预计今天发出',
        ];

        return $orders[$orderNumber] ?? '未找到该订单，请确认订单号是否正确';
    }
}

// 工具 2：预约上门服务
#[AsTool('book_service', description: '为客户预约上门维修或安装服务')]
final class ServiceBookingTool
{
    /**
     * @param string $serviceType 服务类型（维修/安装/保养）
     * @param string $preferredDate 期望日期（如：明天、下周一）
     * @param string $address 上门地址
     */
    public function __invoke(string $serviceType, string $preferredDate, string $address): string
    {
        // 实际项目中这里调用预约系统 API
        $bookingId = 'BK-' . date('Ymd') . '-' . random_int(1000, 9999);

        return sprintf(
            '预约成功！预约号：%s，服务类型：%s，时间：%s，地址：%s',
            $bookingId,
            $serviceType,
            $preferredDate,
            $address,
        );
    }
}
```

> **💡 提示：** `#[AsTool]` 的 `description` 参数非常重要——AI 模型根据描述来判断何时调用哪个工具。描述要清晰具体，参数的 PHPDoc `@param` 注释会被自动提取为工具参数的 schema。

---

## Step 5：完整语音对话回路

把 STT → Anthropic Claude Agent → TTS 串联成完整的语音对话系统。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Bridge\ElevenLabs\PlatformFactory as ElevenLabsFactory;
use Symfony\AI\Platform\Message\Content\Audio;
use Symfony\AI\Platform\Message\Content\Text;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 创建平台实例
$aiPlatform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY']);
$elevenLabs = ElevenLabsFactory::create(apiKey: $_ENV['ELEVENLABS_API_KEY']);

// 注册业务工具
$toolbox = new Toolbox([
    new OrderQueryTool(),
    new ServiceBookingTool(),
]);
$processor = new AgentProcessor($toolbox);

// 创建 Agent — 使用 Anthropic Claude
$agent = new Agent(
    $aiPlatform, 'claude-sonnet-4-20250514',
    [
        new SystemPromptInputProcessor(
            '你是智能电话客服。用简洁友好的口语回答问题。'
            . '注意：你的回复会被转成语音播放给用户，所以不要使用 markdown 格式、'
            . '不要用符号列表，用自然的说话方式。回答控制在 3-5 句话内。'
        ),
        $processor,
    ],
    [$processor],
);

/**
 * 语音对话核心函数
 *
 * @return array{text: string, audioPath: string}
 */
function voiceConversation(
    Agent $agent,
    object $elevenLabs,
    string $audioInputPath,
): array {
    // 1️⃣ STT：语音转文字
    $sttResult = $elevenLabs->invoke('scribe_v1', Audio::fromFile($audioInputPath));
    $userText = $sttResult->asText();
    echo "🎤 用户说：{$userText}\n";

    // 2️⃣ AI 处理：Claude 理解意图，必要时调用工具
    $aiResult = $agent->call(new MessageBag(
        Message::ofUser($userText),
    ));
    $replyText = $aiResult->getContent();
    echo "🤖 AI 回复：{$replyText}\n";

    // 3️⃣ TTS：文字转语音（流式输出降低延迟）
    $ttsResult = $elevenLabs->invoke(
        'eleven_multilingual_v2',
        new Text($replyText),
        [
            'voice' => 'pqHfZKP75CvOlQylNhV4',
            'stream' => true,
        ],
    );

    $outputPath = '/tmp/reply-' . time() . '.mp3';
    $file = fopen($outputPath, 'wb');
    foreach ($ttsResult->asStream() as $chunk) {
        fwrite($file, $chunk);
    }
    fclose($file);
    echo "🔊 语音回复：{$outputPath}\n";

    return ['text' => $replyText, 'audioPath' => $outputPath];
}

// 模拟一通电话
echo "=== 智能电话客服 ===\n\n";
echo "📞 来电...\n\n";

$response = voiceConversation($agent, $elevenLabs, '/path/to/call-round1.mp3');
echo "\n";

$response = voiceConversation($agent, $elevenLabs, '/path/to/call-round2.mp3');
echo "\n";

echo "📞 通话结束\n";
```

> **💡 提示：切换 LLM 非常简单。** 如果你想用 OpenAI 替代 Anthropic，只需改两行：
> ```php
> use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
> $aiPlatform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);
> ```
> 然后把模型名换成 `'gpt-4o-mini'`。Agent 和工具代码完全不变。

---

## Step 6：多语言语音助手

ElevenLabs 的 `scribe_v1` 支持自动语言检测——用户说任何语言，都能正确转录。搭配 `eleven_multilingual_v2` 的多语言 TTS，可以轻松实现跨语言语音交互。

```php
<?php

// 用户用中文说 → AI 用英文回答 → 英文语音输出
$sttResult = $elevenLabs->invoke('scribe_v1', Audio::fromFile('/path/to/chinese-audio.mp3'));
$userText = $sttResult->asText();
echo "识别到的文字：{$userText}\n";

// Claude 翻译并回答（通过 system prompt 控制输出语言）
$result = $aiPlatform->invoke('claude-sonnet-4-20250514', new MessageBag(
    Message::forSystem('用户说中文，请用英文回答他的问题。'),
    Message::ofUser($userText),
));

$englishReply = $result->asText();

// TTS 输出英文语音 — eleven_multilingual_v2 自动适配语言
$ttsResult = $elevenLabs->invoke(
    'eleven_multilingual_v2',
    new Text($englishReply),
    ['voice' => 'pqHfZKP75CvOlQylNhV4'],
);

file_put_contents('/tmp/english-reply.mp3', $ttsResult->asBinary());
echo "中文输入 → 英文语音输出 ✅\n";
```

> **🌐 多语言检测提示：** `scribe_v1` 能自动识别 90+ 种语言，无需手动指定语言代码。在多语种客服场景中，你可以根据 STT 识别结果动态切换 Agent 的 system prompt 语言，实现"说什么语言就用什么语言回答"的体验。对于混合语言场景（如中英夹杂），`eleven_multilingual_v2` 也能正确处理。

---

## Step 7：替换 TTS 供应商（Cartesia）

Symfony AI 的 Bridge 架构让你可以轻松替换语音供应商。Cartesia 的 `sonic-3` 模型是另一个优秀的 TTS 选择，在某些语言和场景下有独特优势。

```bash
composer require symfony/ai-cartesia-platform
```

```php
<?php

use Symfony\AI\Platform\Bridge\Cartesia\PlatformFactory as CartesiaFactory;
use Symfony\AI\Platform\Message\Content\Text;

$cartesia = CartesiaFactory::create(
    $_ENV['CARTESIA_API_KEY'],
    '2024-12-06',  // Cartesia API 版本
);

// 使用 Cartesia sonic-3 模型生成语音
$result = $cartesia->invoke(
    'sonic-3',
    new Text('您好，欢迎来电咨询。我是您的智能助手。'),
    [
        'voice' => 'a0e99841-438c-4a64-b679-ae501e7d6091',  // Cartesia 语音 ID
        'output_format' => [
            'container' => 'mp3',
            'bit_rate' => 128000,
            'sample_rate' => 44100,
        ],
    ],
);

file_put_contents('/tmp/cartesia-reply.mp3', $result->asBinary());
echo "✅ Cartesia TTS 语音已生成\n";
```

> **💡 提示：如何选择 TTS 供应商？**
> - **ElevenLabs**：多语言支持广泛，语音克隆能力强，流式输出成熟，适合国际化场景
> - **Cartesia**：低延迟，音频输出格式控制更精细，适合对音质参数有特殊要求的场景
>
> 你甚至可以同时接入两个供应商，根据语言或场景动态切换。

---

## 生产环境建议

### 延迟优化

> **⚡ 性能提示：并发处理降低整体延迟。** 在完整对话回路中，STT、AI 推理、TTS 三个步骤是串行的。但你可以在架构层面优化：
> - **流式 TTS**（`stream: true`）是最有效的优化——用户不需要等整段语音生成完
> - 使用 `eleven_flash_v2_5` 代替 `eleven_multilingual_v2` 可降低 TTS 延迟约 50%
> - 在 Symfony 项目中，结合 Messenger 组件可以实现 STT 和 AI 推理的异步处理
> - 对于电话系统，考虑预生成常用回复（如问候语、等待提示）的语音缓存

### 音频质量

> **🎵 音频质量建议：**
> - 录音输入：建议 16kHz 采样率、16bit 深度、单声道，这是语音识别的最佳格式
> - TTS 输出：mp3 128kbps 适合大多数场景；对音质要求高的场景（如播客生成）使用 wav 或 flac
> - 注意降噪：如果用户环境嘈杂，在发送 STT 之前做前端降噪可以显著提高识别准确率

### 通话录音存储

> **🏭 生产建议：通话记录对运营至关重要。**
> - 保存每轮对话的原始音频（用户输入）、STT 转录文字、AI 回复文字、TTS 音频
> - 使用结构化目录存储：`/recordings/{date}/{session_id}/{round}-{type}.mp3`
> - 录音数据要遵守当地隐私法规（如 GDPR），注意保留期限和用户知情同意
> - 定期清理过期录音，或存储到对象存储（S3/OSS）以降低成本

---

## 关于 PHP 处理音频的性能

> **Q：语音处理会不会太耗时，不适合 PHP？**

A：不会！因为重点在于：
- **语音识别（STT）和生成（TTS）都是 API 调用**，不是本地计算
- PHP 只负责发送 HTTP 请求和接收结果
- 一次 STT 调用通常 1-3 秒，TTS 通常 0.5-2 秒
- **流式 TTS** 可以边生成边播放，用户体感延迟更低
- PHP 完全胜任作为"编排层"来协调各个 AI 服务

真正的"重计算"发生在 AI 平台服务端，PHP 只是胶水和调度器。

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Audio::fromFile()` | 加载音频文件，自动检测格式 |
| `scribe_v1` | ElevenLabs STT 模型，支持 90+ 语言自动检测 |
| `eleven_multilingual_v2` | ElevenLabs 多语言 TTS 模型 |
| `eleven_flash_v2_5` | ElevenLabs 低延迟 TTS 模型 |
| `sonic-3` | Cartesia TTS 模型，备选供应商 |
| `new Text(...)` | 文字内容用于 TTS 输入 |
| `$result->asBinary()` | 获取生成的音频二进制数据 |
| `$result->asStream()` | 获取流式音频数据迭代器 |
| `stream: true` | 开启流式 TTS，降低首字节延迟 |
| `#[AsTool]` | 定义自定义业务工具供 Agent 调用 |
| `voice` 参数 | 选择不同的语音风格 |

## 下一步

如果你需要处理视频内容（分析、总结、提取关键帧），请看 [21-video-content-analysis.md](./21-video-content-analysis.md)。
