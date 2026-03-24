# 内容安全审核流水线

## 业务场景

你在做一个 UGC（用户生成内容）社区平台。用户可以发帖、评论、上传图片。在内容上线前，需要 AI 对内容进行多维度安全审核：检查是否包含暴力/仇恨言论、是否泄露个人隐私信息（PII）、是否违反平台规则、是否为垃圾广告。审核通过才发布，否则拦截或送人工复审。

**典型应用：** UGC 社区内容审核、电商评论过滤、在线教育内容安全、企业通讯合规检查

## 涉及模块

| 模块 | Composer 包 | 用途 |
|------|------------|------|
| **Platform** | `symfony/ai-platform` | 连接 AI 平台 |
| **OpenAI Bridge** | `symfony/ai-openai-platform` | 连接 OpenAI（主要方案） |
| **Cache Bridge** | `symfony/ai-cache-platform` | 缓存审核结果，减少重复调用 |
| **StructuredOutput** | `symfony/ai-platform` | 将审核结果映射为结构化 PHP 对象 |
| **Agent** | `symfony/ai-agent` | 多阶段审核流水线编排 |
| **Platform 多模态** | `symfony/ai-platform` | 图片内容审核（`Image`、`ImageUrl`） |

## 项目流程图

```
┌──────────────┐     ┌───────────────┐     ┌──────────────────┐     ┌───────────────┐
│  用户提交     │ ──▶ │  缓存命中检查  │ ──▶ │  快速筛选         │ ──▶ │  深度审核       │
│ (文字/图片)   │     │ (CachePlatform)│     │ (gpt-4o-mini)    │     │ (gpt-4o)       │
└──────────────┘     └───────────────┘     └──────────────────┘     └───────────────┘
                            │                      │                        │
                       缓存命中？              明显无害？                 结构化报告
                       ↓ 是                    ↓ 是                      ↓
                    直接返回               直接通过           ┌──────────────────┐
                                                             │  置信度阈值判定    │
                                                             └──────────────────┘
                                                                ↓          ↓
                                                          ✅ 发布    ⚠️ 人工复审 / ❌ 拦截
```

---

## 前置准备

### 环境要求

- PHP 8.2+
- Composer

### 安装依赖

```bash
composer require symfony/ai-openai-platform symfony/ai-cache-platform symfony/ai-agent symfony/event-dispatcher symfony/cache
```

### 设置 API 密钥

```bash
export OPENAI_API_KEY="sk-your-api-key-here"
```

---

## Step 1：定义审核结果结构

使用 StructuredOutput 定义 DTO，让 AI 返回结构化的审核报告。

```php
<?php

namespace App\Dto;

/**
 * 单项审核维度的结果
 */
final class ModerationCheck
{
    /**
     * @param string $dimension  审核维度（toxicity/pii/spam/policy）
     * @param string $verdict    判定结果（pass/flag/block）
     * @param float  $confidence 置信度 0.0 ~ 1.0
     * @param string $reason     判定理由
     */
    public function __construct(
        public readonly string $dimension,
        public readonly string $verdict,
        public readonly float $confidence,
        public readonly string $reason,
    ) {
    }
}

/**
 * 完整审核报告
 */
final class ModerationReport
{
    /**
     * @param string            $finalVerdict 最终判定（approved/rejected/review）
     * @param ModerationCheck[] $checks       各维度审核结果
     * @param string            $summary      审核摘要
     * @param string[]          $flaggedItems 被标记的具体内容片段
     */
    public function __construct(
        public readonly string $finalVerdict,
        public readonly array $checks,
        public readonly string $summary,
        public readonly array $flaggedItems = [],
    ) {
    }
}
```

> [!TIP]
> **DTO 设计建议：** `confidence` 字段非常重要——它让你在后续可以根据置信度动态调整判定策略，而不是完全依赖 AI 的 `verdict`。建议始终保留该字段。

---

## Step 2：创建内容审核器

```php
<?php

require 'vendor/autoload.php';

use App\Dto\ModerationReport;
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

$moderationPrompt = <<<'PROMPT'
你是一个内容安全审核系统。对用户提交的内容进行全面审核。

审核维度：
1. **toxicity** — 是否包含仇恨言论、人身攻击、种族歧视、性别歧视等有害内容
2. **pii** — 是否包含个人隐私信息：手机号、身份证号、银行卡号、家庭住址等
3. **spam** — 是否为垃圾广告、恶意营销、刷单内容
4. **policy** — 是否违反平台规则：虚假信息、侵权内容、诱导关注等

判定标准：
- pass: 该维度无问题
- flag: 有轻微问题，需人工复审
- block: 严重违规，直接拦截

最终判定规则：
- 所有维度 pass → approved
- 有任意 block → rejected
- 有 flag 但无 block → review（送人工复审）
PROMPT;

// 模拟用户提交的内容
$submissions = [
    [
        'user' => 'user_001',
        'type' => 'post',
        'content' => '今天天气真好！和朋友一起去了西湖骑行，分享几张照片。骑行路线推荐：断桥→白堤→苏堤，大约 10 公里。',
    ],
    [
        'user' => 'user_002',
        'type' => 'comment',
        'content' => '这个产品太垃圾了，买了就是浪费钱，卖家就是骗子！所有人都不要买！',
    ],
    [
        'user' => 'user_003',
        'type' => 'post',
        'content' => '急售二手iPhone，加微信 wx_fake_123，手机号 138-0000-1234，'
            . '身份证号 110101199001011234。便宜转让只要 500 元！先到先得！',
    ],
    [
        'user' => 'user_004',
        'type' => 'comment',
        'content' => '关注我的店铺领优惠券！复制这段话打开淘宝 ¥aB1cDe2fG¥ 全场一折起！不买后悔一辈子！！！',
    ],
];

echo "=== 内容安全审核 ===\n\n";

foreach ($submissions as $submission) {
    $messages = new MessageBag(
        Message::forSystem($moderationPrompt),
        Message::ofUser(
            "请审核以下用户提交的内容：\n\n"
            . "用户 ID：{$submission['user']}\n"
            . "内容类型：{$submission['type']}\n"
            . "内容：{$submission['content']}"
        ),
    );

    $result = $platform->invoke('gpt-4o-mini', $messages, [
        'response_format' => ModerationReport::class,
    ]);

    $report = $result->asObject();

    // 显示审核结果
    $icon = match ($report->finalVerdict) {
        'approved' => '✅',
        'rejected' => '❌',
        'review' => '⚠️',
        default => '❓',
    };

    echo "{$icon} [{$submission['user']}] {$report->finalVerdict}\n";
    echo "   摘要：{$report->summary}\n";

    foreach ($report->checks as $check) {
        $checkIcon = match ($check->verdict) {
            'pass' => '✓',
            'flag' => '⚡',
            'block' => '✗',
            default => '?',
        };
        echo "   {$checkIcon} {$check->dimension}: {$check->verdict} ({$check->confidence}) - {$check->reason}\n";
    }

    if ([] !== $report->flaggedItems) {
        echo "   标记内容：" . implode(' | ', $report->flaggedItems) . "\n";
    }
    echo "\n";
}
```

---

## Step 3：使用 Cache Bridge 缓存审核结果

在 UGC 平台中，同一内容可能被重复提交（如转发、引用）。使用 `CachePlatform` 可以避免重复调用 AI，节省成本并提升响应速度。

```php
<?php

require 'vendor/autoload.php';

use App\Dto\ModerationReport;
use Symfony\AI\Platform\Bridge\Cache\CachePlatform;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\Cache\Adapter\ArrayAdapter;
use Symfony\Component\Cache\Adapter\TagAwareAdapter;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

// 1. 创建底层 OpenAI 平台
$innerPlatform = PlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);

// 2. 用 CachePlatform 包装，启用缓存层
$platform = new CachePlatform(
    $innerPlatform,
    cache: new TagAwareAdapter(new ArrayAdapter()),
);

// 3. 调用时指定缓存键和 TTL
$content = '急售二手iPhone，加微信 wx_fake_123';
$cacheKey = 'moderation_' . md5($content);

$messages = new MessageBag(
    Message::forSystem('你是内容安全审核系统。审核用户内容并生成报告。'),
    Message::ofUser($content),
);

$result = $platform->invoke('gpt-4o-mini', $messages, [
    'response_format' => ModerationReport::class,
    'prompt_cache_key' => $cacheKey,
    'prompt_cache_ttl' => 3600, // 缓存 1 小时
]);

$report = $result->asObject();
echo "审核结果：{$report->finalVerdict}\n";

// 再次审核相同内容时，直接命中缓存，不调用 AI
$result2 = $platform->invoke('gpt-4o-mini', $messages, [
    'response_format' => ModerationReport::class,
    'prompt_cache_key' => $cacheKey,
    'prompt_cache_ttl' => 3600,
]);
```

> [!TIP]
> **缓存策略建议：** 在生产环境中，建议将 `ArrayAdapter` 替换为 `RedisAdapter` 或 `FilesystemAdapter`，以实现跨请求持久化。`prompt_cache_key` 最佳实践是使用内容哈希（如 `md5($content)`），确保相同内容命中同一缓存条目。

> [!NOTE]
> **缓存 TTL 选择：** 审核策略可能随时调整（如新增违禁词），因此 TTL 不宜过长。建议：文本审核 1-4 小时，图片审核 15-30 分钟（图片内容可能被替换）。

---

## Step 4：图片 + 文字组合审核

对于带图片的内容，同时审核文字和图片。支持本地文件和 URL 两种方式。

```php
<?php

use Symfony\AI\Platform\Message\Content\Image;
use Symfony\AI\Platform\Message\Content\ImageUrl;

// 方式一：审核本地上传的图片（适合用户上传场景）
$messages = new MessageBag(
    Message::forSystem($moderationPrompt),
    Message::ofUser(
        "请审核以下用户提交的图文内容：\n\n"
        . "用户 ID：user_005\n"
        . "内容类型：图文帖子\n"
        . "文字内容：看看我做的美食！\n"
        . "附带图片如下：",
        Image::fromFile('/path/to/user-uploaded-image.jpg'),
    ),
);

$result = $platform->invoke('gpt-4o-mini', $messages, [
    'response_format' => ModerationReport::class,
]);

$report = $result->asObject();
echo "图文审核结果：{$report->finalVerdict}\n";
echo "摘要：{$report->summary}\n";

// 方式二：审核 URL 引用的图片（适合外链图片、CDN 图片）
$messages = new MessageBag(
    Message::forSystem($moderationPrompt),
    Message::ofUser(
        "请审核以下用户提交的图文内容：\n\n"
        . "用户 ID：user_006\n"
        . "内容类型：评论（含外链图片）\n"
        . "文字内容：这个地方超好看，推荐大家去！\n"
        . "附带图片如下：",
        new ImageUrl('https://example.com/user-shared-photo.jpg'),
    ),
);

$result = $platform->invoke('gpt-4o-mini', $messages, [
    'response_format' => ModerationReport::class,
]);

// 方式三：多张图片同时审核
$messages = new MessageBag(
    Message::forSystem($moderationPrompt),
    Message::ofUser(
        "请审核以下用户提交的图文帖子，包含多张图片：\n\n"
        . "用户 ID：user_007\n"
        . "文字内容：旅行日记第三天\n"
        . "附带图片如下：",
        Image::fromFile('/uploads/photo1.jpg'),
        Image::fromFile('/uploads/photo2.jpg'),
        new ImageUrl('https://cdn.example.com/photo3.jpg'),
    ),
);

$result = $platform->invoke('gpt-4o-mini', $messages, [
    'response_format' => ModerationReport::class,
]);
```

> [!TIP]
> **图片审核最佳实践：** `Image::fromFile()` 会将图片编码为 Base64 发送，适合用户直接上传的场景。`ImageUrl` 只传递 URL 引用，适合已经上传到 CDN 的图片——传输数据量更小、速度更快。建议在用户上传后先存储到对象存储，再用 `ImageUrl` 进行审核。

> [!WARNING]
> **图片安全注意：** 审核系统接收的图片 URL 必须经过校验——防止 SSRF（服务端请求伪造）攻击。务必限制 URL 白名单（如只允许你的 CDN 域名），禁止内网地址和 `file://` 协议。

---

## Step 5：置信度阈值策略

AI 返回的置信度可以帮助你实现更精细的审核策略，而不是简单地依赖 `verdict` 字段。

```php
<?php

/**
 * 基于置信度的审核决策引擎
 *
 * @param array{
 *     block_threshold: float,
 *     flag_threshold: float,
 *     auto_approve_threshold: float,
 * } $thresholds
 */
function applyConfidenceStrategy(
    ModerationReport $report,
    array $thresholds = [
        'block_threshold' => 0.9,        // 置信度 ≥ 0.9 才自动拦截
        'flag_threshold' => 0.7,         // 置信度 ≥ 0.7 标记为需人工复审
        'auto_approve_threshold' => 0.85, // 所有维度 pass 且置信度 ≥ 0.85 才自动通过
    ],
): string {
    foreach ($report->checks as $check) {
        // 高置信度 block → 直接拦截
        if ('block' === $check->verdict && $check->confidence >= $thresholds['block_threshold']) {
            return 'rejected';
        }

        // 低置信度 block → 不自动拦截，送人工复审
        if ('block' === $check->verdict && $check->confidence < $thresholds['block_threshold']) {
            return 'review';
        }

        // flag 且达到阈值 → 人工复审
        if ('flag' === $check->verdict && $check->confidence >= $thresholds['flag_threshold']) {
            return 'review';
        }
    }

    // 所有维度都是 pass，但检查置信度是否足够高
    foreach ($report->checks as $check) {
        if ($check->confidence < $thresholds['auto_approve_threshold']) {
            return 'review'; // 置信度不够高，保守处理
        }
    }

    return 'approved';
}

// 使用示例
$finalDecision = applyConfidenceStrategy($report);

echo match ($finalDecision) {
    'approved' => "✅ 自动通过\n",
    'rejected' => "❌ 自动拦截\n",
    'review' => "⚠️ 送人工复审\n",
};
```

> [!TIP]
> **阈值调优建议：** 初期上线建议设置偏保守的阈值（block_threshold 较高，auto_approve_threshold 较高），宁可多送人工复审也不要误放。随着积累审核数据，逐步调整阈值。可以按内容类型（帖子、评论、私信）设置不同的阈值策略。

> [!NOTE]
> **误判处理（False Positive）：** AI 审核不可避免会出现误判。建议：1）为被拦截的内容提供**申诉机制**，用户可以请求人工复审；2）记录所有申诉成功的案例，定期分析误判模式，优化审核提示词；3）对高频误判的内容类型，适当降低该维度的阈值。

---

## Step 6：审核流水线编排

在生产环境中，审核通常分为多个阶段，形成流水线——平衡成本、速度和准确性：

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;

/**
 * 多阶段审核流水线：快速筛选 → 深度审核 → 置信度判定
 *
 * @return array{verdict: string, stage: string, report: ?ModerationReport}
 */
function moderateContent(string $content, object $platform): array
{
    // 第一阶段：快速筛选（用轻量模型，低成本高速）
    $quickScreenAgent = new Agent(
        $platform, 'gpt-4o-mini',
        [new SystemPromptInputProcessor(
            '你是快速内容筛选器。只判断内容是否明显违规。'
            . '回复 PASS（明显无问题）或 DEEP_REVIEW（需要深度审核）。'
            . '宁可多送审，不可漏放。'
        )],
    );

    $quickResult = $quickScreenAgent->call(new MessageBag(
        Message::ofUser("快速筛选：{$content}"),
    ));

    if (str_contains($quickResult->getContent(), 'PASS')) {
        return ['verdict' => 'approved', 'stage' => 'quick_screen', 'report' => null];
    }

    // 第二阶段：深度审核（用更强模型，全维度结构化检查）
    $messages = new MessageBag(
        Message::forSystem($moderationPrompt),
        Message::ofUser($content),
    );

    $result = $platform->invoke('gpt-4o', $messages, [
        'response_format' => ModerationReport::class,
    ]);

    $report = $result->asObject();

    // 第三阶段：置信度阈值判定
    $finalDecision = applyConfidenceStrategy($report);

    return ['verdict' => $finalDecision, 'stage' => 'deep_review', 'report' => $report];
}
```

> [!TIP]
> **成本优化：** 快速筛选阶段使用 `gpt-4o-mini`（成本约为 `gpt-4o` 的 1/10）。大部分正常内容在第一阶段即可通过，只有可疑内容才进入深度审核。在典型 UGC 场景中，约 80-90% 的内容是无害的，这种分级策略可以大幅降低审核成本。

---

## 完整示例：内容审核微服务

```php
<?php

require 'vendor/autoload.php';

use App\Dto\ModerationCheck;
use App\Dto\ModerationReport;
use Symfony\AI\Platform\Bridge\Cache\CachePlatform;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Image;
use Symfony\AI\Platform\Message\Content\ImageUrl;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\Cache\Adapter\ArrayAdapter;
use Symfony\Component\Cache\Adapter\TagAwareAdapter;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

// --- 平台初始化（带缓存） ---

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

$innerPlatform = PlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);

$platform = new CachePlatform(
    $innerPlatform,
    cache: new TagAwareAdapter(new ArrayAdapter()),
);

// --- 审核提示词 ---

$moderationPrompt = <<<'PROMPT'
你是内容安全审核系统。从 toxicity、pii、spam、policy 四个维度审核内容。
所有 pass → approved；有 block → rejected；有 flag → review。
对每个维度给出置信度评分（0.0 ~ 1.0）和判定理由。
PROMPT;

/**
 * 内容审核函数 — 可作为 API 端点的核心逻辑
 *
 * @param string[] $imagePaths 本地图片路径列表
 * @param string[] $imageUrls  图片 URL 列表
 */
function moderateUserContent(
    object $platform,
    string $moderationPrompt,
    string $userId,
    string $contentType,
    string $content,
    array $imagePaths = [],
    array $imageUrls = [],
): ModerationReport {
    $cacheKey = 'moderation_' . md5($userId . $contentType . $content . implode(',', $imagePaths) . implode(',', $imageUrls));

    // 构建消息内容（文字 + 图片）
    $userContent = ["用户 {$userId} 的 {$contentType}：\n{$content}"];

    foreach ($imagePaths as $path) {
        $userContent[] = Image::fromFile($path);
    }
    foreach ($imageUrls as $url) {
        $userContent[] = new ImageUrl($url);
    }

    $messages = new MessageBag(
        Message::forSystem($moderationPrompt),
        Message::ofUser(...$userContent),
    );

    $result = $platform->invoke('gpt-4o-mini', $messages, [
        'response_format' => ModerationReport::class,
        'prompt_cache_key' => $cacheKey,
        'prompt_cache_ttl' => 3600,
    ]);

    return $result->asObject();
}

// --- 模拟 API 调用 ---

// 纯文本审核
$report = moderateUserContent(
    $platform,
    $moderationPrompt,
    'user_008',
    'comment',
    '这篇文章写得真好，学到了很多新知识！',
);

echo "审核结果：{$report->finalVerdict}\n";
echo "摘要：{$report->summary}\n";

// 图文审核
$report = moderateUserContent(
    $platform,
    $moderationPrompt,
    'user_009',
    'post',
    '分享一下今天的午餐',
    imagePaths: ['/uploads/user_009/lunch.jpg'],
);

echo "图文审核结果：{$report->finalVerdict}\n";

// 根据结果做业务处理
match ($report->finalVerdict) {
    'approved' => publishContent(),    // 直接发布
    'rejected' => blockContent(),      // 拦截并通知用户
    'review' => queueForReview(),      // 进入人工复审队列
};
```

> [!NOTE]
> **业务函数说明：** `publishContent()`、`blockContent()`、`queueForReview()` 为你的业务逻辑函数，需要根据实际项目实现。

---

## 其他实现方案：使用 Anthropic (Claude)

Anthropic Claude 同样支持内容审核和多模态分析。只需更换 Bridge 即可。

### 安装 Anthropic Bridge

```bash
composer require symfony/ai-anthropic-platform
```

### 代码调整

```php
<?php

// 仅需更换 PlatformFactory 的引入、API 密钥和模型名称
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;

$innerPlatform = PlatformFactory::create(
    $_ENV['ANTHROPIC_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);

// 同样可以用 CachePlatform 包装
$platform = new CachePlatform(
    $innerPlatform,
    cache: new TagAwareAdapter(new ArrayAdapter()),
);

// 其余代码完全一致，只需更换模型名称
$result = $platform->invoke('claude-sonnet-4-20250514', $messages, [
    'response_format' => ModerationReport::class,
    'prompt_cache_key' => $cacheKey,
    'prompt_cache_ttl' => 3600,
]);
```

> [!TIP]
> **平台选择建议：** OpenAI `gpt-4o-mini` 在内容审核场景中性价比很高，速度快且成本低。Anthropic Claude 在细微语境理解方面有优势，适合需要更细腻判断的场景。你也可以将两者结合——快速筛选用 OpenAI，深度审核用 Claude。

---

## 生产环境建议

> [!TIP]
> **批量处理：** 在高流量场景下，不要逐条审核。使用消息队列（如 Symfony Messenger）将审核请求异步化，配合批量处理降低延迟。同时设置合理的并发数，避免触发 API 速率限制。

> [!TIP]
> **速率限制（Rate Limiting）：** OpenAI 和 Anthropic 都有 API 调用限制。建议：1）使用令牌桶算法控制调用频率；2）设置退避重试机制；3）在高峰期优先处理付费用户的内容。

> [!TIP]
> **审计日志（Audit Logging）：** 内容审核系统必须保留完整的审计日志——包括原始内容、审核报告（`ModerationReport`）、最终判定、操作时间戳和操作人。这不仅是合规要求，也是优化审核策略的重要数据来源。建议将日志存入独立的数据库表，设置合理的保留期限。

> [!WARNING]
> **安全注意事项：** 1）审核系统本身需要安全防护——防止攻击者通过精心构造的内容绕过审核（Prompt Injection）；2）审核日志包含敏感内容，需要严格的访问控制；3）定期审计审核系统的准确率，防止模型退化导致大量漏放；4）不要在日志中明文存储用户的 PII 信息——审核后应脱敏处理。

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| 多维度审核 | 单次调用同时检查 toxicity、pii、spam、policy 四个安全维度 |
| 结构化审核报告 | AI 返回结构化 `ModerationReport` 对象，便于程序化处理 |
| 缓存审核结果 | `CachePlatform` 包装底层平台，避免重复内容重复调用 AI |
| 置信度阈值策略 | 基于 `confidence` 动态判定，比硬编码 `verdict` 更灵活 |
| 图片审核 | `Image::fromFile()` 审核本地图片，`ImageUrl` 审核 URL 图片 |
| 多阶段流水线 | 快筛（`gpt-4o-mini`）→ 深审（`gpt-4o`）→ 阈值判定，平衡成本和安全 |
| 误判处理 | 提供申诉机制，记录误判案例，持续优化审核策略 |
| 平台可切换 | OpenAI / Anthropic 通过 Bridge 模式无缝切换 |

## 下一步

如果你需要 AI 帮助审查代码质量、查找 bug 或建议改进，请看 [13-ai-code-review-assistant.md](./13-ai-code-review-assistant.md)。
