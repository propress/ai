# 内容安全审核流水线

## 业务场景

你在做一个 UGC（用户生成内容）社区平台。用户可以发帖、评论、上传图片。在内容上线前，需要 AI 对内容进行多维度安全审核：检查是否包含暴力/仇恨言论、是否泄露个人隐私信息（PII）、是否违反平台规则、是否为垃圾广告。审核通过才发布，否则拦截或送人工复审。

**典型应用：** UGC 社区内容审核、电商评论过滤、在线教育内容安全、企业通讯合规检查

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台 |
| **StructuredOutput** | 将审核结果映射为结构化 PHP 对象 |
| **Agent** | 多阶段审核流水线编排 |
| **Platform 多模态** | 图片内容审核（使用 Image Content） |

---

## Step 1：定义审核结果结构

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

## Step 3：图片 + 文字组合审核

对于带图片的内容，同时审核文字和图片。

```php
<?php

use Symfony\AI\Platform\Message\Content\Image;

// 图文内容审核
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
```

---

## Step 4：审核流水线编排

在生产环境中，审核通常分为多个阶段，形成流水线：

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;

/**
 * 审核流水线：快速筛选 → 深度审核 → 人工复审队列
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
        return ['verdict' => 'approved', 'stage' => 'quick_screen'];
    }

    // 第二阶段：深度审核（用更强模型，全维度检查）
    // 此处使用上面 Step 2 的完整审核逻辑
    // ...
    return ['verdict' => 'review', 'stage' => 'deep_review'];
}
```

---

## 完整示例：内容审核微服务

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

/**
 * 内容审核函数 — 可作为 API 端点的核心逻辑
 */
function moderateUserContent(
    object $platform,
    string $userId,
    string $contentType,
    string $content,
): ModerationReport {
    $prompt = '你是内容安全审核系统。从 toxicity、pii、spam、policy 四个维度审核内容。'
        . '所有 pass → approved；有 block → rejected；有 flag → review。';

    $messages = new MessageBag(
        Message::forSystem($prompt),
        Message::ofUser("用户 {$userId} 的 {$contentType}：\n{$content}"),
    );

    $result = $platform->invoke('gpt-4o-mini', $messages, [
        'response_format' => ModerationReport::class,
    ]);

    return $result->asObject();
}

// 模拟 API 调用
$report = moderateUserContent(
    $platform,
    'user_006',
    'comment',
    '这篇文章写得真好，学到了很多新知识！',
);

echo "审核结果：{$report->finalVerdict}\n";

// 根据结果做业务处理
match ($report->finalVerdict) {
    'approved' => publishContent(),    // 直接发布
    'rejected' => blockContent(),      // 拦截并通知用户
    'review' => queueForReview(),      // 进入人工复审队列
};
```

> 注意：`publishContent()`、`blockContent()`、`queueForReview()` 为你的业务逻辑函数。

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| 多维度审核 | 单次调用同时检查多个安全维度 |
| 结构化审核报告 | AI 返回结构化 `ModerationReport` 对象 |
| 置信度评分 | 每个维度包含置信度，辅助人工决策 |
| 图文组合审核 | 同时审核文字和图片内容 |
| 多阶段流水线 | 快筛 → 深审 → 人工复审，平衡成本和安全 |
| 标记内容片段 | `flaggedItems` 精确定位违规内容 |

## 下一步

如果你需要 AI 帮助审查代码质量、查找 bug 或建议改进，请看 [13-ai-code-review-assistant.md](./13-ai-code-review-assistant.md)。
