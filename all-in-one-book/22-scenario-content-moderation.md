# 第 22 章：实战 —— 内容审核流水线

## 学习目标

- 理解生产级内容审核系统的分层设计
- 掌握结构化输出（StructuredOutput）在审核中的应用
- 学会构建文本和图片审核服务
- 了解缓存优化在审核场景中的成本节约效果
- 实现完整的 Symfony 控制器集成

## 前置知识

- 熟悉 Symfony AI Platform 的 `invoke()` 和 `response_format` 选项
- 了解 PHP 8.1+ 枚举（Enum）和只读属性
- 了解 Symfony HTTP 基础（Controller、Request、JsonResponse）
- 了解 CachePlatform 的基本用法（参见 [第 20 章](20-scenario-high-availability.md)）

## 业务场景描述

构建一个生产级的内容审核系统，支持文本和图片审核，使用结构化输出确保审核结果格式一致，支持多平台交叉验证和缓存优化。

**典型应用**：UGC 平台、社交应用、电商评论系统、论坛管理。

## 架构概述

```text
内容审核流水线
══════════════

  用户提交内容
       │
       ▼
  ┌──────────────────────────────────────────┐
  │  1. 预过滤（快速规则检查）                    │
  │     - 敏感词列表匹配                        │
  │     - 正则表达式检查                        │
  │     - 长度/格式验证                         │
  │     → 明显违规：直接拒绝（不消耗 AI 额度）   │
  └──────────────┬───────────────────────────┘
                 │ 通过预过滤
                 ▼
  ┌──────────────────────────────────────────┐
  │  2. AI 审核（结构化输出）                     │
  │     - 发送内容到 AI                         │
  │     - 返回 ModerationResult DTO             │
  │     - 包含：safe/confidence/categories      │
  │     → 缓存相同内容的审核结果                  │
  └──────────────┬───────────────────────────┘
                 │
       ┌─────────┼─────────┐
       │         │         │
       ▼         ▼         ▼
   approve    review    reject
   (发布)     (人工复核)  (拒绝)
```

## 环境准备

确保已安装以下 Composer 包：

```bash
composer require symfony/ai-platform
composer require symfony/cache
```

需要的环境变量：

```dotenv
OPENAI_API_KEY=your-openai-key
```

## 核心实现

### 审核结果 DTO

```php
<?php

namespace App\Dto;

use Symfony\AI\Platform\Contract\JsonSchema\Attribute\With;

class ModerationResult
{
    public function __construct(
        #[With(description: '内容是否安全，可以发布')]
        public readonly bool $safe,

        #[With(description: '审核置信度 0-100，100 为最确定')]
        public readonly int $confidence,

        #[With(description: '检测到的违规类别列表，如：violence/sexual/hate/spam/privacy')]
        public readonly array $categories,

        #[With(description: '审核说明，解释判定理由')]
        public readonly string $reason,

        #[With(description: '建议操作：approve（发布）/review（人工复核）/reject（拒绝）')]
        public readonly ModerationAction $action,
    ) {}
}

enum ModerationAction: string
{
    case APPROVE = 'approve';
    case REVIEW = 'review';
    case REJECT = 'reject';
}
```

### 审核服务

```php
<?php

namespace App\Service;

use Symfony\AI\Platform\PlatformInterface;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\Message\Content\Image;

class ContentModerationService
{
    private const SYSTEM_PROMPT = <<<PROMPT
你是一个专业的内容审核系统。审核内容是否包含以下违规类别：
- violence: 暴力/血腥内容
- sexual: 色情/不雅内容
- hate: 仇恨言论/歧视
- spam: 垃圾信息/诈骗
- privacy: 个人隐私泄露（手机号、身份证号等）

审核规则：
1. confidence >= 90 且 safe=true → action=approve
2. confidence >= 90 且 safe=false → action=reject
3. confidence < 90 → action=review（交由人工复核）
4. 不确定时宁可交给人工复核，不要轻易放行
PROMPT;

    public function __construct(
        private readonly PlatformInterface $platform,
        private readonly string $model = 'gpt-4o-mini',  // 审核任务用便宜模型即可
    ) {}

    /**
     * 审核文本内容
     */
    public function moderateText(string $content): ModerationResult
    {
        $messages = new MessageBag(
            Message::forSystem(self::SYSTEM_PROMPT),
            Message::ofUser('请审核以下用户提交的内容：'."\n\n".$content),
        );

        $response = $this->platform->invoke($this->model, $messages, [
            'response_format' => ModerationResult::class,
            'temperature' => 0.1,  // 低 temperature 确保判定一致性
        ]);

        return $response->asObject();
    }

    /**
     * 审核图片内容
     */
    public function moderateImage(string $imagePath): ModerationResult
    {
        $messages = new MessageBag(
            Message::forSystem(self::SYSTEM_PROMPT),
            Message::ofUser(
                Image::fromFile($imagePath),
                '请审核这张用户上传的图片。',
            ),
        );

        $response = $this->platform->invoke('gpt-4o', $messages, [
            'response_format' => ModerationResult::class,
            'temperature' => 0.1,
        ]);

        return $response->asObject();
    }

    /**
     * 执行审核并分发结果
     */
    public function processContent(string $contentId, string $content): void
    {
        // 预过滤：快速规则检查
        if ($this->containsBannedWords($content)) {
            $this->rejectContent($contentId, '包含敏感词');
            return;
        }

        // AI 审核
        $result = $this->moderateText($content);

        match ($result->action) {
            ModerationAction::APPROVE => $this->publishContent($contentId),
            ModerationAction::REVIEW => $this->sendToManualReview($contentId, $result),
            ModerationAction::REJECT => $this->rejectContent($contentId, $result->reason),
        };
    }
}
```

### 缓存优化

对于相同或相似内容的重复审核，使用 CachePlatform 避免重复调用 AI：

```php
use Symfony\AI\Platform\Bridge\Cache\CachePlatform;

$cachedPlatform = new CachePlatform(platform: $innerPlatform, cache: $cache);

// 自定义缓存键——基于内容哈希
$moderator = new ContentModerationService($cachedPlatform);

// 在调用时提供缓存键
$response = $cachedPlatform->invoke($model, $messages, [
    'response_format' => ModerationResult::class,
    'prompt_cache_key' => 'moderation-'.md5($content),  // 基于内容哈希
    'prompt_cache_ttl' => 86400,   // 审核结果缓存 24 小时
]);

// 相同内容的重复提交会直接命中缓存——零 API 成本
```

### 完整的 Symfony 集成

```php
<?php

namespace App\Controller;

use App\Service\ContentModerationService;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Routing\Attribute\Route;

class CommentController
{
    public function __construct(
        private readonly ContentModerationService $moderator,
    ) {}

    #[Route('/api/comments', methods: ['POST'])]
    public function create(Request $request): JsonResponse
    {
        $content = $request->getPayload()->getString('content');

        $result = $this->moderator->moderateText($content);

        if (ModerationAction::REJECT === $result->action) {
            return new JsonResponse([
                'status' => 'rejected',
                'reason' => $result->reason,
            ], 422);
        }

        if (ModerationAction::REVIEW === $result->action) {
            // 保存为待审核状态
            return new JsonResponse([
                'status' => 'pending_review',
                'message' => '您的评论已提交，正在审核中。',
            ]);
        }

        // 直接发布
        return new JsonResponse([
            'status' => 'published',
            'message' => '评论已发布。',
        ]);
    }
}
```

## 运行与验证

1. 配置 `ContentModerationService` 并注入 Platform
2. 提交安全内容，验证返回 `approve` 且 confidence >= 90
3. 提交明显违规内容（如敏感词），验证预过滤直接拒绝（不消耗 AI 额度）
4. 提交模糊内容，验证返回 `review` 并进入人工复核队列
5. 重复提交相同内容，确认缓存命中（检查响应元数据 `cached: true`）

## 错误处理

- **AI 返回格式异常**：结构化输出通过 DTO 类型约束，解析失败时应 catch 异常并将内容送入人工复核
- **平台不可用**：结合 FailoverPlatform 进行容灾；全部失败时将内容暂存到待审核队列
- **图片审核失败**：检查图片格式和大小限制，不支持的格式应提前校验并返回友好错误

## 生产环境注意事项

- 使用 `gpt-4o-mini` 等低成本模型进行文本审核，仅图片审核使用多模态模型
- 设置合理的缓存 TTL（如 24 小时），平衡实时性与成本
- 监控审核队列长度和人工复核比例，优化 system prompt 降低 `review` 率
- 定期评估审核准确率，必要时调整 confidence 阈值

## 扩展方向

- 添加视频帧抽取 + 审核支持
- 实现多模型交叉验证（两个模型同时审核，结果不一致时送人工复核）
- 构建审核结果反馈闭环，将人工复核结果用于优化 system prompt
- 集成 Symfony Messenger 实现异步审核队列

## 完整源代码

本章涉及的完整代码片段已在各小节中给出。核心组件：

1. `ModerationResult` DTO + `ModerationAction` 枚举：结构化审核结果
2. `ContentModerationService`：审核服务（文本 + 图片 + 预过滤 + 分发）
3. `CommentController`：Symfony 控制器集成示例

## 下一步

下一个场景我们将学习如何构建 CrewAI 风格的多智能体协作团队——参见 [第 23 章：实战 —— CrewAI 风格多智能体团队](23-scenario-crew-multi-agent.md)。
