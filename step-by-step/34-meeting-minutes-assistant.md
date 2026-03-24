# AI 会议纪要助手

## 业务场景

你的公司每天有大量会议：产品评审、技术方案讨论、客户沟通、周会。会后需要整理会议纪要：谁说了什么、做了什么决定、有哪些行动项。这件事通常由轮值的同事花 30 分钟手动整理。现在用 AI：上传会议录音，自动转录为文字，然后提取结构化的会议纪要、行动项和决议。

**典型应用：** 会议纪要自动生成、客户沟通记录、培训课程笔记、播客内容提取

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台 |
| **Platform (ElevenLabs)** | 语音转文字（STT） |
| **Audio Content** | 音频文件输入 |
| **StructuredOutput** | 会议纪要结构化输出 |
| **Agent** | 编排转录→分析→总结流程 |

## 前置准备

```bash
composer require symfony/ai-platform symfony/ai-platform-openai
composer require symfony/ai-platform-elevenlabs  # 或 openai whisper
composer require symfony/ai-agent
```

---

## Step 1：音频转录

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\ElevenLabs\PlatformFactory as ElevenLabsFactory;
use Symfony\AI\Platform\Message\Content\Audio;
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create();
$elevenLabs = ElevenLabsFactory::create($_ENV['ELEVENLABS_API_KEY'], $httpClient);

// 上传会议录音
$result = $elevenLabs->invoke(
    'scribe_v1',
    Audio::fromFile('/path/to/meeting-recording.mp3'),
);

$transcript = $result->asText();
echo "📝 转录完成（" . mb_strlen($transcript) . " 字）\n";
echo "前 300 字预览：\n" . mb_substr($transcript, 0, 300) . "...\n";
```

---

## Step 2：定义会议纪要结构

```php
<?php

namespace App\Dto;

final class ActionItem
{
    /**
     * @param string $owner    负责人
     * @param string $task     任务描述
     * @param string $deadline 截止时间（如有）
     * @param string $priority 优先级（high/medium/low）
     */
    public function __construct(
        public readonly string $owner,
        public readonly string $task,
        public readonly string $deadline,
        public readonly string $priority,
    ) {
    }
}

final class Decision
{
    /**
     * @param string $topic    决策主题
     * @param string $outcome  决策结果
     * @param string $rationale 决策依据
     */
    public function __construct(
        public readonly string $topic,
        public readonly string $outcome,
        public readonly string $rationale,
    ) {
    }
}

final class DiscussionTopic
{
    /**
     * @param string   $topic        讨论主题
     * @param string   $summary      讨论摘要
     * @param string[] $keyPoints    关键观点
     * @param string[] $participants 参与者
     */
    public function __construct(
        public readonly string $topic,
        public readonly string $summary,
        public readonly array $keyPoints,
        public readonly array $participants,
    ) {
    }
}

final class MeetingMinutes
{
    /**
     * @param string            $title         会议标题
     * @param string            $date          会议日期
     * @param string            $duration      时长（如 "45 分钟"）
     * @param string[]          $attendees     参会人员
     * @param string            $executiveSummary 一段话总结
     * @param DiscussionTopic[] $discussions   讨论主题
     * @param Decision[]        $decisions     决议
     * @param ActionItem[]      $actionItems   行动项
     * @param string            $nextMeeting   下次会议安排
     */
    public function __construct(
        public readonly string $title,
        public readonly string $date,
        public readonly string $duration,
        public readonly array $attendees,
        public readonly string $executiveSummary,
        public readonly array $discussions,
        public readonly array $decisions,
        public readonly array $actionItems,
        public readonly string $nextMeeting,
    ) {
    }
}
```

---

## Step 3：生成结构化会议纪要

```php
<?php

use App\Dto\MeetingMinutes;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$platform = PlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    $httpClient,
    eventDispatcher: $dispatcher,
);

$messages = new MessageBag(
    Message::forSystem(
        "你是专业的会议纪要整理专家。从会议转录文本中提取：\n"
        . "1. 参会人员（从对话中识别）\n"
        . "2. 每个讨论主题及其关键观点\n"
        . "3. 所有做出的决定\n"
        . "4. 明确的行动项（谁、做什么、什么时候）\n"
        . "5. 下次会议安排\n"
        . "如果转录文本中没有明确提到某项信息，标注为'未提及'。"
    ),
    Message::ofUser("请整理以下会议转录的纪要：\n\n{$transcript}"),
);

$result = $platform->invoke('gpt-4o', $messages, [
    'response_format' => MeetingMinutes::class,
]);

$minutes = $result->asObject();

// 输出格式化的会议纪要
echo "╔══════════════════════════════════════╗\n";
echo "║         📋 会议纪要                  ║\n";
echo "╚══════════════════════════════════════╝\n\n";

echo "📌 {$minutes->title}\n";
echo "📅 {$minutes->date} | ⏱ {$minutes->duration}\n";
echo "👥 参会：" . implode('、', $minutes->attendees) . "\n\n";

echo "📝 摘要：\n{$minutes->executiveSummary}\n\n";

// 讨论内容
echo "═══ 讨论内容 ═══\n\n";
foreach ($minutes->discussions as $i => $topic) {
    echo ($i + 1) . ". 【{$topic->topic}】\n";
    echo "   {$topic->summary}\n";
    foreach ($topic->keyPoints as $point) {
        echo "   • {$point}\n";
    }
    echo "   参与：" . implode('、', $topic->participants) . "\n\n";
}

// 决议
if ([] !== $minutes->decisions) {
    echo "═══ 决议 ═══\n\n";
    foreach ($minutes->decisions as $decision) {
        echo "✅ {$decision->topic}\n";
        echo "   结果：{$decision->outcome}\n";
        echo "   依据：{$decision->rationale}\n\n";
    }
}

// 行动项
if ([] !== $minutes->actionItems) {
    echo "═══ 行动项 ═══\n\n";
    foreach ($minutes->actionItems as $item) {
        $icon = match ($item->priority) {
            'high' => '🔴',
            'medium' => '🟡',
            'low' => '🟢',
            default => '⚪',
        };
        echo "{$icon} [{$item->owner}] {$item->task}\n";
        echo "   截止：{$item->deadline}\n\n";
    }
}

echo "📅 下次会议：{$minutes->nextMeeting}\n";
```

---

## Step 4：会议情感与参与度分析

```php
<?php

namespace App\Dto;

final class ParticipantAnalysis
{
    /**
     * @param string $name             参与者
     * @param int    $speakingShare    发言占比（0-100%）
     * @param string $role             角色（facilitator/contributor/observer）
     * @param string $sentiment        情感基调（positive/neutral/concerned/frustrated）
     * @param string $keyContribution  主要贡献
     */
    public function __construct(
        public readonly string $name,
        public readonly int $speakingShare,
        public readonly string $role,
        public readonly string $sentiment,
        public readonly string $keyContribution,
    ) {
    }
}

final class MeetingAnalysis
{
    /**
     * @param string                $meetingEfficiency    会议效率评级（high/medium/low）
     * @param ParticipantAnalysis[] $participantAnalysis  参与者分析
     * @param string[]              $improvementSuggestions 改进建议
     * @param int                   $actionableRate       行动项明确率（0-100%）
     */
    public function __construct(
        public readonly string $meetingEfficiency,
        public readonly array $participantAnalysis,
        public readonly array $improvementSuggestions,
        public readonly int $actionableRate,
    ) {
    }
}
```

```php
<?php

use App\Dto\MeetingAnalysis;

$messages = new MessageBag(
    Message::forSystem(
        '你是组织效能顾问。分析会议转录，评估会议效率和参与者表现。'
    ),
    Message::ofUser("分析以下会议：\n\n{$transcript}"),
);

$result = $platform->invoke('gpt-4o-mini', $messages, [
    'response_format' => MeetingAnalysis::class,
]);

$analysis = $result->asObject();

echo "=== 会议效能分析 ===\n";
echo "效率评级：{$analysis->meetingEfficiency}\n";
echo "行动项明确率：{$analysis->actionableRate}%\n\n";

echo "参与者分析：\n";
foreach ($analysis->participantAnalysis as $p) {
    echo "  👤 {$p->name}：{$p->speakingShare}% 发言 | {$p->role} | {$p->sentiment}\n";
    echo "     贡献：{$p->keyContribution}\n";
}

echo "\n💡 改进建议：\n";
foreach ($analysis->improvementSuggestions as $suggestion) {
    echo "  - {$suggestion}\n";
}
```

---

## Step 5：Gemini 直接分析音频

Gemini 支持直接输入音频文件分析，不需要先转录。

```php
<?php

use Symfony\AI\Platform\Bridge\Google\PlatformFactory as GeminiFactory;
use Symfony\AI\Platform\Message\Content\Audio;

$gemini = GeminiFactory::create($_ENV['GOOGLE_API_KEY'], $httpClient);

// Gemini 直接理解音频内容（不只是转录）
$messages = new MessageBag(
    Message::forSystem('从这段会议录音中提取关键决议和行动项。'),
    Message::ofUser(
        '请分析这段会议录音：',
        Audio::fromFile('/path/to/meeting.mp3'),
    ),
);

$result = $gemini->invoke('gemini-2.0-flash', $messages);
echo "Gemini 分析：\n" . $result->asText() . "\n";
```

---

## 完整流程

```
会议录音（mp3/wav/m4a）
    │
    ├──► [ElevenLabs STT] → 文字转录
    │         │
    │         ▼
    │    [GPT-4o 分析] → MeetingMinutes
    │         ├─ 讨论主题
    │         ├─ 决议
    │         ├─ 行动项
    │         └─ 效能分析
    │
    └──► [Gemini 直接分析] → 音频理解（无需转录）
              │
              ▼
         格式化输出 → 发送给参会者
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `scribe_v1` | ElevenLabs STT 模型 |
| `Audio::fromFile()` | 加载音频文件 |
| 嵌套 DTO | `MeetingMinutes` 含 `ActionItem[]`、`Decision[]` |
| 会议效能分析 | 参与度、情感、效率评估 |
| Gemini 音频 | 直接分析音频，无需先转录 |

## 下一步

如果你想用 AI 构建知识图谱并进行智能查询，请看 [35-knowledge-graph-construction.md](./35-knowledge-graph-construction.md)。
