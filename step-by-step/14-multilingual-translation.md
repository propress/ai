# 多语言内容翻译与本地化

## 业务场景

你的产品要进入海外市场。网站内容、产品描述、帮助文档、营销邮件都需要翻译成多种语言。不只是逐字翻译，还需要保持品牌调性，适应当地文化习惯，确保专有术语的一致性。翻译完成后还需要质量评分、分段置信度和人工校对流程，保证上线质量。

**典型应用：** 产品国际化、文档多语言化、营销内容本地化、客服多语言支持、SaaS 界面翻译

## 涉及模块

| 模块 | Composer 包 | 用途 |
|------|------------|------|
| **Platform** | `symfony/ai-platform` + `symfony/ai-gemini-platform` | 连接 AI 平台，发送翻译请求并获取回复 |
| **Bridge（Gemini）** | `symfony/ai-gemini-platform` | 本教程的主实现，通过 `PlatformFactory` 连接 Google Gemini |
| **StructuredOutput** | `symfony/ai-platform`（内置） | 翻译结果结构化输出，包含置信度和文化注释 |
| **Agent** | `symfony/ai-agent` | 智能翻译 Agent，注入术语记忆保证一致性 |
| **Chat** | `symfony/ai-chat` | 多轮翻译校对对话，持久化讨论上下文 |

> **💡 提示：** 本教程使用 **Google Gemini** 作为主要平台。Gemini 在多语言理解和生成方面表现出色，且 `gemini-2.5-flash` 模型在翻译场景中兼具质量与性价比。如果你更熟悉 OpenAI，只需替换 `PlatformFactory` 和模型名称即可——翻译逻辑与具体平台无关。

---

## 项目流程图

```
┌──────────────────┐     ┌──────────────────────┐     ┌───────────────────┐
│   原文内容输入     │ ──▶ │  StaticMemoryProvider │ ──▶ │   Gemini Platform  │
│ (中文营销文案)     │     │  (术语表 / Glossary)  │     │  (gemini-2.5-flash) │
└──────────────────┘     └──────────────────────┘     └─────────┬─────────┘
                                                                │
                                                                ▼
                                                    ┌───────────────────────┐
                                                    │   StructuredOutput    │
                                                    │ TranslationResult DTO │
                                                    │ (分段译文 + 置信度)    │
                                                    └───────────┬───────────┘
                                                                │
                              ┌──────────────────────────────────┤
                              ▼                                  ▼
                   ┌───────────────────┐              ┌───────────────────┐
                   │  质量 ≥ 阈值？      │    否        │   Chat 多轮校对    │
                   │  overallQuality    │ ──────────▶  │ (审校人员反馈迭代)  │
                   │  ≥ 0.85           │              └─────────┬─────────┘
                   └────────┬──────────┘                        │
                      是    │                                   │
                            ▼                                   ▼
                   ┌───────────────────┐              ┌───────────────────┐
                   │  直接输出译文       │              │  输出校对后终稿     │
                   └───────────────────┘              └───────────────────┘
```

---

## 前置准备

### 环境要求

- PHP 8.2+
- Composer
- Google Gemini API 密钥（[获取地址](https://aistudio.google.com/apikey)）

### 安装依赖

```bash
composer require symfony/ai-platform symfony/ai-gemini-platform
composer require symfony/ai-agent
composer require symfony/ai-chat
```

### 设置 API 密钥

```bash
export GEMINI_API_KEY="your-gemini-api-key-here"
```

> **🔒 安全建议：** 永远不要将 API 密钥硬编码在源代码中。使用环境变量或 Symfony Secrets 管理敏感信息。在 `.env.local` 中存放密钥，并确保该文件已加入 `.gitignore`。

---

## Step 1：基础翻译

从最简单的场景开始——将中文内容翻译为英语和日语。

```php
<?php

require_once 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['GEMINI_API_KEY']);

$sourceText = '我们的产品帮助企业提升效率。无论团队规模大小，CloudFlow 都能让协作变得简单流畅。立即开始免费试用！';

// ===== 翻译为英语 =====
$messages = new MessageBag(
    Message::forSystem(
        '你是一个专业翻译。将中文内容翻译为目标语言。'
        . '要求：1) 自然流畅，不是直译；2) 保持原文的营销语气；3) 适应目标语言的文化习惯。'
    ),
    Message::ofUser("请将以下内容翻译为英语：\n\n{$sourceText}"),
);

$result = $platform->invoke('gemini-2.5-flash', $messages);
echo "英语：" . $result->asText() . "\n\n";

// ===== 翻译为日语 =====
$messages = new MessageBag(
    Message::forSystem(
        'あなたはプロの翻訳者です。中国語のコンテンツを日本語に翻訳してください。'
        . '自然な日本語で、マーケティングのトーンを維持してください。'
    ),
    Message::ofUser("以下の中国語を日本語に翻訳してください：\n\n{$sourceText}"),
);

$result = $platform->invoke('gemini-2.5-flash', $messages);
echo "日语：" . $result->asText() . "\n";
```

> **💡 提示：** 系统提示语言建议与目标语言一致。例如翻译为日语时，用日语写系统提示，能帮助模型更好地理解目标语言的表达习惯和语气要求。

> **⚠️ 注意：** 注意区分地区变体（Regional Variants）。英语有 `en-US`（美式）和 `en-GB`（英式）之分，例如 "color" vs "colour"、"organize" vs "organise"。在系统提示中明确指定目标地区，如"翻译为美式英语（en-US）"，可避免混用。葡萄牙语（`pt-BR` vs `pt-PT`）和西班牙语（`es-MX` vs `es-ES`）同理。

---

## Step 2：术语一致性保障

真实项目中，品牌名、产品功能名等专有术语必须保持一致。使用 `StaticMemoryProvider` 将术语表注入 Agent 记忆，AI 在翻译时会自动参考。

```php
<?php

require_once 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;
use Symfony\AI\Agent\Memory\StaticMemoryProvider;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['GEMINI_API_KEY']);

// ===== 术语表作为静态记忆 =====
$glossary = new StaticMemoryProvider(
    '术语：CloudFlow → CloudFlow（不翻译，保持品牌名）',
    '术语：看板 → Kanban Board',
    '术语：智能提醒 → Smart Reminders',
    '术语：工作流 → Workflow',
    '术语：免费试用 → Free Trial',
    '术语：企业版 → Enterprise Edition',
    '术语：基础版 → Starter Plan',
    '术语：专业版 → Professional Plan',
    '术语：截止日期 → Deadline（不要用 Due Date）',
);

// ===== 组装 Agent =====
$memoryProcessor = new MemoryInputProcessor([$glossary]);

$systemPrompt = new SystemPromptInputProcessor(
    '你是专业的 SaaS 产品翻译，将中文翻译为英语。'
    . '必须严格遵守术语表中的翻译规范，品牌名和产品术语要与术语表保持一致。'
    . '翻译风格：自然地道，适合英语母语用户阅读，保持营销语气。'
);

$agent = new Agent($platform, 'gemini-2.5-flash', [$systemPrompt, $memoryProcessor]);

// ===== 翻译时 AI 会自动参考术语表 =====
$result = $agent->call(new MessageBag(
    Message::ofUser(
        '翻译为英语：CloudFlow 专业版提供看板视图和智能提醒功能，帮你追踪每一个截止日期。'
        . '立即开始免费试用。'
    ),
));

echo $result->getContent() . "\n";
// 输出会遵循术语表：Professional Plan, Kanban Board, Smart Reminders, Deadline, Free Trial
```

> **🏭 生产建议：** 在实际项目中，术语表通常存储在数据库或 CSV 文件中。你可以实现自定义的 `MemoryProviderInterface`，从数据库动态加载特定领域的术语表。例如，医疗翻译和法律翻译分别加载不同的术语集。

> **💡 提示：** `MemoryInputProcessor` 将术语表内容注入到系统提示中，格式为 `# Conversation Memory` 段落。AI 模型会将术语表作为高优先级参考，这比在用户消息中附加术语表效果更好。

---

## Step 3：带质量评分的结构化翻译

翻译不能只输出文字，还需要质量指标——每个分段的置信度、文化差异注释、整体质量评分——帮助人工校对聚焦问题区域。

### 定义翻译结果 DTO

```php
<?php

namespace App\Dto;

final class TranslationSegment
{
    /**
     * @param string  $source      原文片段
     * @param string  $translation 翻译片段
     * @param float   $confidence  翻译置信度 0.0~1.0
     * @param ?string $note        翻译注释（文化差异、术语选择等说明）
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
     * @param string               $sourceLanguage  源语言代码（如 zh-CN）
     * @param string               $targetLanguage  目标语言代码（如 en-US）
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

### 使用结构化翻译

```php
<?php

require_once 'vendor/autoload.php';

use App\Dto\TranslationResult;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$platform = PlatformFactory::create(
    $_ENV['GEMINI_API_KEY'],
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
        . '将内容分段翻译，每段给出翻译置信度（0.0~1.0）。'
        . '如果有文化差异需要注意，在对应片段的 note 和 culturalNotes 中说明。'
        . '使用 en-US（美式英语）。'
    ),
    Message::ofUser("请将以下营销内容翻译为英语，并给出质量评估：\n\n{$sourceText}"),
);

$result = $platform->invoke('gemini-2.5-flash', $messages, [
    'response_format' => TranslationResult::class,
]);

$translation = $result->asObject();

// ===== 输出翻译结果 =====
echo "=== 翻译结果 ===\n";
echo "语言对：{$translation->sourceLanguage} → {$translation->targetLanguage}\n";
echo "整体质量：" . round($translation->overallQuality * 100) . "%\n\n";

echo "完整翻译：\n{$translation->fullTranslation}\n\n";

// ===== 分段详情 =====
echo "分段详情：\n";
foreach ($translation->segments as $segment) {
    $quality = round($segment->confidence * 100);
    echo "  原文：{$segment->source}\n";
    echo "  译文：{$segment->translation}\n";
    echo "  置信度：{$quality}%";
    if (null !== $segment->note) {
        echo " | 注释：{$segment->note}";
    }
    echo "\n\n";
}

// ===== 文化适配注释 =====
if ([] !== $translation->culturalNotes) {
    echo "文化适配注释：\n";
    foreach ($translation->culturalNotes as $note) {
        echo "  📝 {$note}\n";
    }
}
```

> **💡 提示：** `PlatformSubscriber` 会拦截包含 `response_format` 选项的请求，自动将 DTO 类转换为 JSON Schema 发送给模型，并将返回的 JSON 反序列化为对象。调用 `$result->asObject()` 即可获得类型安全的 `TranslationResult` 实例。

> **⚠️ 注意：** 置信度（confidence）是模型的自我评估，并非绝对准确。建议将 `overallQuality < 0.85` 的译文标记为需要人工复审。对于法律、医疗等高风险场景，即使高置信度也应进行人工校对。

---

## Step 4：多轮翻译校对

结构化翻译完成后，审校人员可以和 AI 进行多轮对话，逐步打磨译文。`Chat` 组件会自动保存对话历史，AI 始终了解完整上下文。

```php
<?php

require_once 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\InMemory\Store as ChatStore;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['GEMINI_API_KEY']);
$agent = new Agent($platform, 'gemini-2.5-flash');
$chat = new Chat($agent, new ChatStore());

// ===== 初始化校对会话 =====
$chat->initiate(new MessageBag(
    Message::forSystem(
        '你是中英翻译校对专家。根据审校人员的反馈调整翻译。'
        . '保持营销语气，确保译文自然地道（美式英语）。'
        . '每次修改后，说明修改原因。'
    ),
));

// ===== 第一轮：审校人员提出修改意见 =====
$response = $chat->submit(
    Message::ofUser(
        '以下翻译中，"把工作安排得井井有条" 翻译为 "organize your work perfectly"，'
        . '感觉不够地道，在英文营销中有没有更好的表达？'
    ),
);
echo "AI：" . $response->getContent() . "\n\n";

// ===== 第二轮：继续讨论细节 =====
$response = $chat->submit(
    Message::ofUser(
        '"keep everything on track" 不错。另外 "14天免费试用" 这段，'
        . '海外用户更习惯怎么表达？需要加 "no credit card required" 吗？'
    ),
);
echo "AI：" . $response->getContent() . "\n\n";

// ===== 第三轮：请求最终版本 =====
$response = $chat->submit(
    Message::ofUser('好的，请输出最终校对后的完整英文版本。'),
);
echo "最终译文：\n" . $response->getContent() . "\n";
```

> **💡 提示：** `Chat` 组件通过 `MessageStoreInterface` 存储对话历史。本教程使用 `InMemory\Store` 进行演示，生产环境可替换为基于数据库的持久化存储（如 DBAL Store、MongoDB Store），使校对会话在页面刷新后仍能继续。

> **🏭 生产建议：** 对于大型翻译项目，建议将校对流程与项目管理工具集成。每条译文的校对历史可作为审计记录保留，方便回溯翻译决策过程。

---

## Step 5：批量多语言翻译

将同一份内容翻译成多种目标语言。结合前面的术语表和结构化输出，确保每种语言的翻译都经过质量评估。

```php
<?php

require_once 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['GEMINI_API_KEY']);

$targetLanguages = [
    'en-US' => ['name' => 'English (US)', 'direction' => 'ltr'],
    'ja'    => ['name' => '日本語', 'direction' => 'ltr'],
    'ko'    => ['name' => '한국어', 'direction' => 'ltr'],
    'es-MX' => ['name' => 'Español (México)', 'direction' => 'ltr'],
    'ar'    => ['name' => 'العربية', 'direction' => 'rtl'],
    'de'    => ['name' => 'Deutsch', 'direction' => 'ltr'],
];

$sourceText = '立即注册 CloudFlow，享 14 天免费试用。无需信用卡。';

foreach ($targetLanguages as $code => $lang) {
    $directionHint = 'rtl' === $lang['direction']
        ? '注意：目标语言是 RTL（从右到左书写），请确保翻译符合 RTL 排版习惯。'
        : '';

    $messages = new MessageBag(
        Message::forSystem(
            "你是专业翻译。将中文翻译为 {$lang['name']}（{$code}）。"
            . '保持营销语气，简洁有力。品牌名 CloudFlow 不翻译。'
            . $directionHint
        ),
        Message::ofUser($sourceText),
    );

    $result = $platform->invoke('gemini-2.5-flash', $messages);
    $dir = $lang['direction'];
    echo "[{$code}] ({$dir}) {$result->asText()}\n";
}
```

> **⚠️ 注意：** 翻译阿拉伯语、希伯来语等 RTL（从右到左）语言时，除了翻译文字外，还需要在前端处理排版方向。在 HTML 中使用 `dir="rtl"` 属性，CSS 中使用 `direction: rtl`。标点符号、数字和品牌名（如 CloudFlow）在 RTL 文本中会自动左对齐，但需要测试确保混排效果正确。

> **🏭 生产建议：** 批量翻译时注意 API 成本。`gemini-2.5-flash` 相比 `gemini-2.5-pro` 价格更低，适合大批量翻译初稿。建议策略：先用 Flash 模型批量翻译 → 结构化输出评估质量 → 仅对低置信度片段使用 Pro 模型重译，可节省 60%~80% 的 API 成本。

---

## 完整示例

以下是一个完整的翻译流水线示例，结合术语表、结构化输出和质量评估：

```php
<?php

require_once 'vendor/autoload.php';

use App\Dto\TranslationResult;
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;
use Symfony\AI\Agent\Memory\StaticMemoryProvider;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

// ===== 1. 初始化平台（启用结构化输出） =====
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$platform = PlatformFactory::create(
    $_ENV['GEMINI_API_KEY'],
    eventDispatcher: $dispatcher,
);

// ===== 2. 定义术语表 =====
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

// ===== 3. 组装翻译 Agent =====
$memoryProcessor = new MemoryInputProcessor([$glossary]);
$systemPrompt = new SystemPromptInputProcessor(
    '你是专业的 SaaS 产品翻译，将中文翻译为美式英语（en-US）。'
    . '严格遵守术语表中的翻译规范。'
    . '将内容分段翻译，每段给出翻译置信度（0.0~1.0）。'
    . '如果有文化差异需要注意，在 culturalNotes 中说明。'
);

$agent = new Agent($platform, 'gemini-2.5-flash', [$systemPrompt, $memoryProcessor]);

// ===== 4. 执行结构化翻译 =====
$sourceText = <<<'TEXT'
CloudFlow 项目管理平台

高效协作，智能管理

无论你的团队是 5 人还是 5000 人，CloudFlow 都能帮你把工作安排得井井有条。
我们的看板视图让任务一目了然，智能提醒确保不错过任何截止日期。

升级到专业版，解锁高级工作流自动化。
立即注册，享 14 天免费试用。不需要信用卡。
TEXT;

$result = $agent->call(
    new MessageBag(
        Message::ofUser("请翻译以下营销内容并给出质量评估：\n\n{$sourceText}"),
    ),
    ['response_format' => TranslationResult::class],
);

$translation = $result->getContent();

// ===== 5. 输出翻译结果 =====
echo "=== 翻译结果 ===\n";
echo "语言对：{$translation->sourceLanguage} → {$translation->targetLanguage}\n";
echo "整体质量：" . round($translation->overallQuality * 100) . "%\n\n";
echo "完整翻译：\n{$translation->fullTranslation}\n\n";

// ===== 6. 质量检查 =====
echo "分段详情：\n";
$needsReview = false;
foreach ($translation->segments as $segment) {
    $quality = round($segment->confidence * 100);
    $flag = $segment->confidence < 0.85 ? '⚠️' : '✅';
    echo "  {$flag} [{$quality}%] {$segment->source}\n";
    echo "     → {$segment->translation}\n";
    if (null !== $segment->note) {
        echo "     📝 {$segment->note}\n";
    }
    if ($segment->confidence < 0.85) {
        $needsReview = true;
    }
    echo "\n";
}

if ([] !== $translation->culturalNotes) {
    echo "文化适配注释：\n";
    foreach ($translation->culturalNotes as $note) {
        echo "  📝 {$note}\n";
    }
    echo "\n";
}

if ($needsReview) {
    echo "⚠️ 存在低置信度片段，建议进入人工校对流程。\n";
}
```

---

## 其他实现方案

### 使用 OpenAI 平台

将 Gemini 替换为 OpenAI，只需更换 Bridge 包和平台初始化代码：

```bash
# 替换 Bridge 包
composer require symfony/ai-open-ai-platform
# composer remove symfony/ai-gemini-platform  # 如果不再需要
```

```php
// 修改前（Gemini）
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
$platform = PlatformFactory::create($_ENV['GEMINI_API_KEY']);
$agent = new Agent($platform, 'gemini-2.5-flash', [$systemPrompt, $memoryProcessor]);

// 修改后（OpenAI）
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);
$agent = new Agent($platform, 'gpt-4o-mini', [$systemPrompt, $memoryProcessor]);
```

> **💡 提示：** 术语表（`StaticMemoryProvider`）、结构化输出（`TranslationResult` DTO）和校对流程（`Chat`）的代码完全不需要修改——这就是 Symfony AI 桥接架构的优势：业务逻辑与具体平台解耦。

### 使用高质量模型精翻

对于低置信度片段，可以使用更强的模型进行重译：

```php
// 初稿：使用 gemini-2.5-flash（快速、低成本）
$draftAgent = new Agent($platform, 'gemini-2.5-flash', [$systemPrompt, $memoryProcessor]);

// 精翻：使用 gemini-2.5-pro（更高质量）
$refinedAgent = new Agent($platform, 'gemini-2.5-pro', [$systemPrompt, $memoryProcessor]);

// 对低置信度片段使用 Pro 模型重译
foreach ($translation->segments as $segment) {
    if ($segment->confidence < 0.85) {
        $result = $refinedAgent->call(new MessageBag(
            Message::ofUser("请重新翻译这段内容，注意文化适配：\n\n{$segment->source}"),
        ));
        echo "重译：{$result->getContent()}\n";
    }
}
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| **系统提示定义翻译风格** | 通过 System Prompt 控制翻译语气、目标地区和风格要求 |
| **StaticMemoryProvider 术语表** | 将术语表作为静态记忆注入 Agent，确保品牌名和专有术语翻译一致 |
| **MemoryInputProcessor** | 将术语记忆自动合并到系统提示中，AI 以高优先级参考 |
| **StructuredOutput（TranslationResult DTO）** | 结构化输出包含分段置信度、文化注释、整体质量评分 |
| **Chat 多轮校对** | 持久化翻译讨论上下文，审校人员可迭代打磨译文 |
| **地区变体** | 明确指定 en-US / en-GB、es-MX / es-ES 等地区代码，避免混用 |
| **RTL 语言处理** | 阿拉伯语、希伯来语等需要额外处理排版方向和混排 |
| **批量翻译成本优化** | Flash 模型批量初稿 + Pro 模型定点精翻，可节省 60%~80% API 成本 |
| **翻译记忆缓存** | 生产环境应缓存已翻译片段，避免重复调用 API 翻译相同内容 |
| **领域术语表** | 不同业务领域（医疗、法律、电商）维护独立术语表，通过 `MemoryProviderInterface` 动态加载 |

---

## 下一步

如果你需要一个智能出行助手，能查天气、查地点、综合多种实时信息给出建议，请看 [15-weather-travel-planner.md](./15-weather-travel-planner.md)。
