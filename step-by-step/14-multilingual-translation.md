# 多语言内容翻译与本地化

## 业务场景

你的产品要进入海外市场。网站内容、产品描述、帮助文档、营销邮件都需要翻译成多种语言。不只是逐字翻译，还需要保持品牌调性，适应当地文化习惯。翻译完成后还需要质量评分和人工校对。

**典型应用：** 产品国际化、文档多语言化、营销内容本地化、客服多语言支持

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台进行翻译 |
| **StructuredOutput** | 翻译结果包含质量评分和注释 |
| **Chat** | 多轮翻译校对对话 |
| **Agent** | 带工具的翻译 Agent（搜索术语库等） |

---

## Step 1：基础翻译

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());

$sourceText = '我们的产品帮助企业提升效率。无论团队规模大小，CloudFlow 都能让协作变得简单流畅。立即开始免费试用！';

$messages = new MessageBag(
    Message::forSystem(
        '你是一个专业翻译。将中文内容翻译为目标语言。'
        . '要求：1) 自然流畅，不是直译；2) 保持原文的营销语气；3) 适应目标语言的文化习惯。'
    ),
    Message::ofUser("请将以下内容翻译为英语：\n\n{$sourceText}"),
);

$result = $platform->invoke('gpt-4o-mini', $messages);
echo "英语：" . $result->asText() . "\n\n";

// 翻译成日语
$messages = new MessageBag(
    Message::forSystem(
        'あなたはプロの翻訳者です。中国語のコンテンツを日本語に翻訳してください。'
        . '自然な日本語で、マーケティングのトーンを維持してください。'
    ),
    Message::ofUser("以下の中国語を日本語に翻訳してください：\n\n{$sourceText}"),
);

$result = $platform->invoke('gpt-4o-mini', $messages);
echo "日语：" . $result->asText() . "\n";
```

---

## Step 2：带质量评分的结构化翻译

翻译不能只输出文字，还需要质量指标，帮助人工校对聚焦问题区域。

```php
<?php

namespace App\Dto;

final class TranslationSegment
{
    /**
     * @param string  $source       原文片段
     * @param string  $translation  翻译片段
     * @param float   $confidence   翻译置信度 0.0~1.0
     * @param ?string $note         翻译注释（文化差异、术语选择等说明）
     */
    public function __construct(
        public readonly string $source,
        public readonly string $translation,
        public readonly float $confidence,
        public readonly ?string $note = null,
    ) {
    }
}

final class TranslationResult
{
    /**
     * @param string               $sourceLanguage  源语言
     * @param string               $targetLanguage  目标语言
     * @param string               $fullTranslation 完整翻译
     * @param float                $overallQuality  整体质量评分 0.0~1.0
     * @param TranslationSegment[] $segments        分段翻译及评分
     * @param string[]             $culturalNotes   文化适配注释
     */
    public function __construct(
        public readonly string $sourceLanguage,
        public readonly string $targetLanguage,
        public readonly string $fullTranslation,
        public readonly float $overallQuality,
        public readonly array $segments,
        public readonly array $culturalNotes = [],
    ) {
    }
}
```

### 使用结构化翻译：

```php
<?php

use App\Dto\TranslationResult;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$platform = PlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);

$sourceText = <<<'TEXT'
CloudFlow 项目管理平台

高效协作，智能管理

无论你的团队是 5 人还是 5000 人，CloudFlow 都能帮你把工作安排得井井有条。
我们的看板视图让任务一目了然，智能提醒确保不错过任何截止日期。

立即注册，享 14 天免费试用。不需要信用卡。
TEXT;

$messages = new MessageBag(
    Message::forSystem(
        '你是专业的中英翻译，专注于 SaaS 产品营销内容。'
        . '将内容分段翻译，每段给出翻译置信度。'
        . '如果有文化差异需要注意，在注释中说明。'
    ),
    Message::ofUser("请将以下营销内容翻译为英语，并给出质量评估：\n\n{$sourceText}"),
);

$result = $platform->invoke('gpt-4o-mini', $messages, [
    'response_format' => TranslationResult::class,
]);

$translation = $result->asObject();

echo "=== 翻译结果 ===\n";
echo "语言对：{$translation->sourceLanguage} → {$translation->targetLanguage}\n";
echo "整体质量：" . round($translation->overallQuality * 100) . "%\n\n";

echo "完整翻译：\n{$translation->fullTranslation}\n\n";

echo "分段详情：\n";
foreach ($translation->segments as $segment) {
    $quality = round($segment->confidence * 100);
    echo "  原文：{$segment->source}\n";
    echo "  译文：{$segment->translation}\n";
    echo "  质量：{$quality}%";
    if (null !== $segment->note) {
        echo " | 注释：{$segment->note}";
    }
    echo "\n\n";
}

if ([] !== $translation->culturalNotes) {
    echo "文化适配注释：\n";
    foreach ($translation->culturalNotes as $note) {
        echo "  📝 {$note}\n";
    }
}
```

---

## Step 3：多轮翻译校对

翻译完成后，审校人员可以和 AI 讨论调整。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\InMemory\Store as ChatStore;

$agent = new Agent($platform, 'gpt-4o-mini');
$chat = new Chat($agent, new ChatStore());

$proofreadingId = 'proofread-homepage-en';

$chat->initiate(new MessageBag(
    Message::forSystem(
        '你是中英翻译校对专家。根据审校人员的反馈调整翻译。'
        . '保持营销语气，确保译文自然地道。'
    ),
), $proofreadingId);

// 审校人员提出修改意见
$response = $chat->submit(
    Message::ofUser(
        '以下翻译中，"把工作安排得井井有条" 翻译为 "organize your work perfectly"，'
        . '感觉不够地道，在英文营销中有没有更好的表达？'
    ),
    $proofreadingId,
);
echo "AI：" . $response->getContent() . "\n\n";

// 审校人员确认
$response = $chat->submit(
    Message::ofUser(
        '"keep everything on track" 不错。另外 "14天免费试用" 这段，'
        . '海外用户更习惯怎么表达？需要加 "no credit card required" 吗？'
    ),
    $proofreadingId,
);
echo "AI：" . $response->getContent() . "\n\n";

// 请求最终版本
$response = $chat->submit(
    Message::ofUser('好的，请输出最终校对后的完整英文版本。'),
    $proofreadingId,
);
echo "最终译文：\n" . $response->getContent() . "\n";
```

---

## Step 4：批量多语言翻译

一次性翻译成多种语言。

```php
<?php

$targetLanguages = [
    'en' => 'English',
    'ja' => '日本語',
    'ko' => '한국어',
    'es' => 'Español',
    'de' => 'Deutsch',
];

$sourceText = '立即注册 CloudFlow，享 14 天免费试用。';

foreach ($targetLanguages as $code => $language) {
    $messages = new MessageBag(
        Message::forSystem("你是专业翻译。将中文翻译为 {$language}。保持营销语气，简洁有力。"),
        Message::ofUser($sourceText),
    );

    $result = $platform->invoke('gpt-4o-mini', $messages);
    echo "[{$code}] {$result->asText()}\n";
}
```

---

## Step 5：术语一致性保障

对于有品牌专有术语的内容，使用 Agent 记忆注入术语表。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;
use Symfony\AI\Agent\Memory\StaticMemoryProvider;

// 术语表作为静态记忆
$glossary = new StaticMemoryProvider(
    '术语：CloudFlow → CloudFlow（不翻译，保持品牌名）',
    '术语：看板 → Kanban Board',
    '术语：智能提醒 → Smart Reminders',
    '术语：工作流 → Workflow',
    '术语：免费试用 → Free Trial',
    '术语：企业版 → Enterprise Edition',
    '术语：基础版 → Starter Plan',
    '术语：专业版 → Professional Plan',
);

$memoryProcessor = new MemoryInputProcessor([$glossary]);

$systemPrompt = new SystemPromptInputProcessor(
    '你是专业翻译，必须严格遵守术语表中的翻译规范。'
    . '品牌名和术语要与术语表保持一致。'
);

$agent = new Agent($platform, 'gpt-4o-mini', [$systemPrompt, $memoryProcessor]);

// 翻译时 AI 会自动参考术语表
$result = $agent->call(new MessageBag(
    Message::ofUser('翻译为英语：CloudFlow 专业版提供看板视图和智能提醒功能。立即开始免费试用。'),
));

echo $result->getContent() . "\n";
// 输出会遵循术语表：Professional Plan, Kanban Board, Smart Reminders, Free Trial
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| 系统提示定义翻译风格 | 通过 System Prompt 控制翻译语气和风格 |
| 结构化翻译结果 | 包含置信度、分段评分、文化注释 |
| 多轮校对 | Chat 持久化翻译讨论上下文 |
| 术语表注入 | StaticMemoryProvider 确保术语一致性 |
| 批量多语言 | 循环调用，一份内容翻译成多种语言 |

## 下一步

如果你需要一个智能出行助手，能查天气、查地点、综合多种实时信息给出建议，请看 [15-weather-travel-planner.md](./15-weather-travel-planner.md)。
