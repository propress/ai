# 语音交互 AI 助手

## 业务场景

你在做一个智能客服电话系统或者语音助手 App。用户通过语音说出问题，系统自动把语音转成文字（STT），交给 AI 理解并处理（可以调用工具查询信息），然后把回答转成语音（TTS）播放给用户。整个流程：**语音输入 → AI 思考 → 语音输出**。

**典型应用：** 智能电话客服、语音助手、无障碍工具、语音导航、播客内容生成

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台 |
| **Audio Content** | 音频文件输入输出 |
| **Platform (ElevenLabs)** | STT（语音转文字）和 TTS（文字转语音） |
| **Agent** | 智能体处理用户意图 |
| **Agent Bridge Clock** | 时间查询工具 |

## 前置准备

```bash
composer require symfony/ai-platform symfony/ai-platform-openai
composer require symfony/ai-platform-elevenlabs  # 语音能力
composer require symfony/ai-agent
composer require symfony/ai-agent-clock
```

需要 ElevenLabs API 密钥（免费额度可用）：
```bash
export ELEVENLABS_API_KEY="your-elevenlabs-api-key"
```

---

## Step 1：语音转文字（Speech-to-Text）

把用户说的话转成文字。ElevenLabs 的 Scribe 模型支持多语言。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\ElevenLabs\PlatformFactory as ElevenLabsFactory;
use Symfony\AI\Platform\Message\Content\Audio;
use Symfony\Component\HttpClient\HttpClient;

$elevenLabs = ElevenLabsFactory::create($_ENV['ELEVENLABS_API_KEY'], HttpClient::create());

// 将用户的语音文件转为文字
$result = $elevenLabs->invoke(
    'scribe_v1',                              // ElevenLabs STT 模型
    Audio::fromFile('/path/to/user-voice.mp3'), // 支持 mp3, wav, m4a 等
);

$userText = $result->asText();
echo "用户说的话：{$userText}\n";
```

**支持的音频格式：** mp3, wav, m4a, ogg, flac, webm

---

## Step 2：文字转语音（Text-to-Speech）

把 AI 的文字回复转成自然的语音。

```php
<?php

use Symfony\AI\Platform\Message\Content\Text;

// AI 生成的回复文字
$replyText = '好的，我已经帮您查询了订单状态。您的包裹目前在杭州转运中心，预计明天下午送达。';

// 转为语音
$result = $elevenLabs->invoke(
    'eleven_multilingual_v2',     // 多语言 TTS 模型
    new Text($replyText),
    [
        'voice' => 'pqHfZKP75CvOlQylNhV4',  // ElevenLabs 语音 ID
    ],
);

// 保存语音文件
$audioData = $result->asBinary();
file_put_contents('/tmp/reply.mp3', $audioData);
echo "✅ 语音回复已生成：/tmp/reply.mp3\n";
```

**可选语音：** ElevenLabs 提供多种预置语音，也支持自定义语音克隆。

---

## Step 3：语音流式输出

对于需要实时播放的场景，使用流式 TTS。

```php
<?php

// 流式 TTS — 边生成边播放
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
    // 在实际应用中，这里可以实时推送给播放器
}
fclose($outputFile);
echo "✅ 流式语音已生成\n";
```

---

## Step 4：完整语音对话回路

把 STT → AI Agent → TTS 串联成完整的语音对话。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\Clock\Clock;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Attribute\AsTool;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\ElevenLabs\PlatformFactory as ElevenLabsFactory;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Audio;
use Symfony\AI\Platform\Message\Content\Text;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\Clock\Clock as SymfonyClock;
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create();
$aiPlatform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], $httpClient);
$elevenLabs = ElevenLabsFactory::create($_ENV['ELEVENLABS_API_KEY'], $httpClient);

// 自定义工具：查询订单
#[AsTool('query_order', description: '根据订单号查询物流状态', method: '__invoke')]
final class OrderQueryTool
{
    public function __invoke(string $orderNumber): string
    {
        // 模拟订单查询
        $orders = [
            'ORD-2025-001' => '已发货 - 杭州转运中心 - 预计明天到达',
            'ORD-2025-002' => '已签收 - 2025-01-18 14:30',
            'ORD-2025-003' => '待发货 - 预计今天发出',
        ];

        return $orders[$orderNumber] ?? '未找到该订单';
    }
}

// 创建带工具的 Agent
$toolbox = new Toolbox([
    new OrderQueryTool(),
    new Clock(new SymfonyClock()),
]);
$processor = new AgentProcessor($toolbox);

$agent = new Agent(
    $aiPlatform, 'gpt-4o-mini',
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
    object $agent,
    object $elevenLabs,
    string $audioInputPath,
): array {
    // 1️⃣ STT：语音转文字
    $sttResult = $elevenLabs->invoke('scribe_v1', Audio::fromFile($audioInputPath));
    $userText = $sttResult->asText();
    echo "🎤 用户说：{$userText}\n";

    // 2️⃣ AI 处理：理解意图并回复
    $aiResult = $agent->call(new MessageBag(
        Message::ofUser($userText),
    ));
    $replyText = $aiResult->getContent();
    echo "🤖 AI 回复：{$replyText}\n";

    // 3️⃣ TTS：文字转语音
    $ttsResult = $elevenLabs->invoke(
        'eleven_multilingual_v2',
        new Text($replyText),
        ['voice' => 'pqHfZKP75CvOlQylNhV4'],
    );

    $outputPath = '/tmp/reply-' . time() . '.mp3';
    file_put_contents($outputPath, $ttsResult->asBinary());
    echo "🔊 语音回复：{$outputPath}\n";

    return ['text' => $replyText, 'audioPath' => $outputPath];
}

// 模拟一通电话
echo "=== 智能电话客服 ===\n\n";

echo "📞 来电...\n\n";

// 第一轮对话
$response = voiceConversation($agent, $elevenLabs, '/path/to/call-round1.mp3');
echo "\n";

// 第二轮对话（用户追问）
$response = voiceConversation($agent, $elevenLabs, '/path/to/call-round2.mp3');
echo "\n";

echo "📞 通话结束\n";
```

---

## Step 5：多语言语音助手

利用 ElevenLabs 的多语言能力，做跨语言语音交互。

```php
<?php

// 用户用中文说 → AI 用英文回答 → 英文语音输出
// 先 STT
$sttResult = $elevenLabs->invoke('scribe_v1', Audio::fromFile('/path/to/chinese-audio.mp3'));
$userText = $sttResult->asText();

// AI 翻译并回答（通过 system prompt 控制语言）
$result = $aiPlatform->invoke('gpt-4o-mini', new MessageBag(
    Message::forSystem('用户说中文，请用英文回答。'),
    Message::ofUser($userText),
));

$englishReply = $result->asText();

// TTS 英文输出
$ttsResult = $elevenLabs->invoke(
    'eleven_multilingual_v2',
    new Text($englishReply),
    ['voice' => 'pqHfZKP75CvOlQylNhV4'],  // 英文语音
);

file_put_contents('/tmp/english-reply.mp3', $ttsResult->asBinary());
echo "中文输入 → 英文语音输出 ✅\n";
```

---

## Step 6：音频内容分析（Gemini）

Gemini 不只做转录，还能理解音频内容的情感、背景声音等。

```php
<?php

use Symfony\AI\Platform\Bridge\Google\PlatformFactory as GeminiFactory;

$gemini = GeminiFactory::create($_ENV['GOOGLE_API_KEY'], HttpClient::create());

// Gemini 可以理解音频内容（不只是转录）
$messages = new MessageBag(
    Message::ofUser(
        '请分析这段客服通话录音，判断：1) 客户的情绪变化 2) 客服的应对质量 3) 问题是否解决',
        Audio::fromFile('/path/to/customer-call.mp3'),
    ),
);

$result = $gemini->invoke('gemini-2.0-flash', $messages);
echo "通话质量分析：\n" . $result->asText() . "\n";
```

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

## 完整流程

```
🎤 用户语音
    │
    ▼
[ElevenLabs STT] → 文字
    │
    ▼
[AI Agent] → 理解意图 + 调用工具
    │
    ▼
[ElevenLabs TTS] → 语音回复
    │
    ▼
🔊 播放给用户
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Audio::fromFile()` | 加载音频文件用于 STT |
| `scribe_v1` | ElevenLabs 语音转文字模型 |
| `eleven_multilingual_v2` | ElevenLabs 多语言 TTS 模型 |
| `new Text(...)` | 文字内容用于 TTS 输入 |
| `$result->asBinary()` | 获取生成的音频二进制数据 |
| `stream: true` | 流式 TTS，边生成边播放 |
| `voice` 参数 | 选择不同的语音风格 |
| Gemini 音频理解 | 不只转录，还能分析情感和内容 |

## 下一步

如果你需要处理视频内容（分析、总结、提取关键帧），请看 [21-video-content-analysis.md](./21-video-content-analysis.md)。
