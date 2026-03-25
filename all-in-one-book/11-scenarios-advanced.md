# 第 11 章：实战场景（高级篇）

## 🎯 本章学习目标

通过 5 个高级实战场景，深入理解生产环境关键技术：FailoverPlatform 和 CachePlatform 的内部机制、本地模型私有化部署、内容审核流水线、CrewAI 风格多智能体团队协作、以及端到端企业知识库系统的完整实现。

---

## 1. 回顾

在 [第 10 章](10-scenarios-intermediate.md) 中，我们掌握了：

- Agent 工具调用循环的内部机制和完整事件链
- RAG 两阶段管线（Loader → Transformer → Vectorizer → Store → Retriever）
- 多智能体路由（Orchestrator + Handoff + 结构化 Decision）
- 联网搜索工具集成与来源追踪
- 记忆系统架构（Static / Embedding / Custom Provider）

本章聚焦**生产环境的关键技术**，解决可用性、性能、成本和安全等实际挑战。

**场景总览**：

| 场景 | 核心技术 | 难度 | 典型应用 |
|------|---------|------|---------|
| 高可用 AI 服务架构 | FailoverPlatform + CachePlatform | ⭐⭐⭐⭐ | 所有生产级应用 |
| 本地模型私有化部署 | Ollama Bridge | ⭐⭐⭐ | 数据敏感场景 |
| 内容审核流水线 | StructuredOutput + Agent | ⭐⭐⭐ | UGC 平台 |
| CrewAI 风格多智能体团队 | MultiAgent + 流水线 | ⭐⭐⭐⭐ | 内容生产 |
| 端到端企业知识库 | 全组件集成 | ⭐⭐⭐⭐⭐ | 企业级系统 |

---

## 2. 场景一：高可用 AI 服务架构

### 2.1 业务场景

生产环境中，AI 服务需要高可用性保障。单一平台可能出现故障、限速或延迟。解决方案是使用多平台容灾和智能缓存。这是每个生产级 AI 应用的基础设施。

**典型应用**：任何需要 99.9%+ 可用性的 AI 服务。

### 2.2 架构概述——三层防御

```
高可用架构的三层防御
══════════════════

  请求 ─────┐
            │
            ▼
  ┌─────────────────────────────┐
  │  第 1 层：CachePlatform       │  缓存命中 → 立即返回（0 API 成本、<1ms）
  │  ─────────────────────────── │
  │  检查缓存键：                  │
  │  model + prompt_cache_key    │
  │  + input_hash                │  缓存未命中 ↓
  └──────────┬──────────────────┘
             │
             ▼
  ┌─────────────────────────────┐
  │  第 2 层：FailoverPlatform    │  平台容灾切换
  │  ─────────────────────────── │
  │  尝试 Platform 1 (OpenAI)    │  成功 → 返回（写入缓存）
  │  失败 → 尝试 Platform 2      │  成功 → 返回（写入缓存）
  │  失败 → 尝试 Platform 3      │  成功 → 返回（写入缓存）
  │  全部失败 → 抛出异常           │
  └──────────┬──────────────────┘
             │
             ▼
  ┌─────────────────────────────┐
  │  第 3 层：优雅降级             │  最后的防线
  │  ─────────────────────────── │
  │  try/catch 捕获异常           │
  │  返回预设回复或人工客服入口    │
  └─────────────────────────────┘
```

### 2.3 FailoverPlatform 内部机制

`FailoverPlatform` 不是简单的轮询——它使用 `WeakMap` 追踪失败平台和速率限制器实现智能恢复：

```php
<?php

use Symfony\AI\Platform\Bridge\Failover\FailoverPlatform;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory as OpenAiFactory;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory as AnthropicFactory;

// 创建多个平台实例
$openai = OpenAiFactory::create($_ENV['OPENAI_API_KEY']);
$anthropic = AnthropicFactory::create($_ENV['ANTHROPIC_API_KEY']);

// 创建容灾平台——接受平台数组，按顺序尝试
$platform = new FailoverPlatform([$openai, $anthropic]);

// 使用方式与单平台完全一致
$response = $platform->invoke($model, $messages);
```

> 📝 **知识扩展：FailoverPlatform 的内部行为**
>
> FailoverPlatform 的核心是一个闭包驱动的重试循环：
>
> ```
> FailoverPlatform::invoke() 的内部流程
> ═══════════════════════════════════
>
> for 每个 Platform in 列表:
>   │
>   ├─ 1. 检查速率限制器（RateLimiter）
>   │      如果该平台之前失败过，速率限制器控制多久后重试
>   │
>   ├─ 2. 检查失败记录（WeakMap<Platform, timestamp>）
>   │      如果该平台在 WeakMap 中有失败记录，且未通过速率限制器检查
>   │      → 跳过该平台，尝试下一个
>   │
>   ├─ 3. 尝试调用
>   │      try {
>   │          $result = $platform->invoke($model, $messages, $options);
>   │          // 成功：从失败记录中移除该平台
>   │          unset($failedPlatforms[$platform]);
>   │          return $result;
>   │      }
>   │
>   └─ 4. 失败处理
>          catch (\Throwable $e) {
>              // 记录失败：$failedPlatforms[$platform] = now
>              // 记录日志
>              // 继续尝试下一个平台
>          }
>
> // 所有平台都失败 → throw RuntimeException
> ```
>
> **关键设计**：
> - 使用 `WeakMap` 而不是普通数组——当平台对象被垃圾回收时，失败记录也自动清除
> - 速率限制器让失败的平台在一段时间后自动恢复尝试（而不是永久标记为失败）
> - 捕获的是 `\Throwable`（不仅是 AI 异常），确保任何类型的故障都能被处理

### 2.4 CachePlatform 内部机制

```php
use Symfony\AI\Platform\Bridge\Cache\CachePlatform;
use Symfony\Component\Cache\Adapter\RedisAdapter;

$cache = new RedisAdapter(/* Redis 连接 */);

$platform = new CachePlatform(
    $innerPlatform,
    $cache,
);

// 缓存调用——必须提供 prompt_cache_key
$response = $platform->invoke($model, $messages, [
    'prompt_cache_key' => 'product-faq',     // 缓存键前缀（必须）
    'prompt_cache_ttl' => 3600,               // 缓存 TTL（可选，默认永不过期）
]);
```

> 📝 **知识扩展：CachePlatform 的缓存键构成**
>
> CachePlatform 的缓存键由三部分组成：
>
> ```
> 缓存键 = prompt_cache_key + 模型名(camelCase) + 输入哈希
>
> 输入哈希的计算规则：
> - string 输入 → MD5($input)
> - array 输入  → MD5(json_encode($input))
> - MessageBag  → MessageBag::getId()->toString()  ← 注意！
> ```
>
> **重要陷阱**：`MessageBag` 使用 UUID v7 作为 ID，每次 `new MessageBag()` 都会生成新 ID。这意味着：
> ```php
> // ❌ 这两个 MessageBag 内容相同，但 ID 不同——不会命中缓存！
> $bag1 = new MessageBag(Message::ofUser('你好'));
> $bag2 = new MessageBag(Message::ofUser('你好'));
> // $bag1->getId() !== $bag2->getId()
>
> // ✅ 要实现内容缓存，需要复用同一个 MessageBag 实例
> // 或者在缓存键中使用内容哈希而非 MessageBag ID
> ```
>
> CachePlatform 还会：
> - 使用 Tag-Aware Caching（按模型名标记），支持按模型清除缓存
> - 在缓存的 Metadata 中添加 `cached: true`、`cache_key`、`cached_at` 时间戳
> - 使用 `ResultNormalizer` + Symfony Serializer 进行结果序列化/反序列化

### 2.5 组合使用——完整的高可用调用链

```php
// 构建三层防御：缓存 → 容灾 → 实际平台

// 第 1 步：创建实际平台
$openai = OpenAiFactory::create($_ENV['OPENAI_API_KEY']);
$anthropic = AnthropicFactory::create($_ENV['ANTHROPIC_API_KEY']);

// 第 2 步：容灾平台
$failover = new FailoverPlatform([$openai, $anthropic]);

// 第 3 步：缓存平台（包装容灾平台）
$platform = new CachePlatform($failover, $cache);

// 请求的完整路径：
// 1. CachePlatform 检查缓存 → 命中则直接返回（0 成本、<1ms）
// 2. 缓存未命中 → FailoverPlatform 尝试 OpenAI
// 3. OpenAI 失败 → FailoverPlatform 自动切换到 Anthropic
// 4. 成功响应 → CachePlatform 写入缓存
// 5. 下次相同请求 → 步骤 1 直接返回
```

### 2.6 AI Bundle 配置

```yaml
# config/packages/ai.yaml
ai:
    platform:
        open_ai:
            api_key: '%env(OPENAI_API_KEY)%'
        anthropic:
            api_key: '%env(ANTHROPIC_API_KEY)%'
        # 容灾平台
        failover:
            type: failover
            platforms:
                - ai.platform.open_ai
                - ai.platform.anthropic
        # 缓存平台（包装容灾平台）
        cached:
            type: cache
            platform: ai.platform.failover
            cache_pool: cache.ai
```

### 2.7 成本分析

假设一个月 100,000 次 AI 调用：

| 优化措施 | 实际 API 调用次数 | 成本（以 GPT-4o 计） |
|---------|:----------------:|:-----------------:|
| 无优化 | 100,000 | ~$500 |
| CachePlatform（70% 命中率） | 30,000 | ~$150 |
| + 简单任务用 mini 模型 | 30,000 | ~$35 |

> 💡 **经验值**：FAQ 类应用的缓存命中率通常可达 60-80%，API 成本降低效果显著。

---

## 3. 场景二：本地模型私有化部署

### 3.1 业务场景

某些场景（数据安全、合规要求、离线运行、成本控制）需要使用本地模型，避免数据外传。Ollama 让本地运行大模型变得简单。

**典型应用**：金融、医疗、政府等数据敏感行业；离线环境；内部开发工具。

### 3.2 架构概述

```
本地模型部署架构
══════════════

  ┌──────────────────────────────────────────────────────────┐
  │  企业内网                                                  │
  │                                                          │
  │  ┌──────────┐     ┌──────────────┐     ┌──────────────┐ │
  │  │ Symfony   │────▶│ Ollama Server │────▶│ 本地 GPU/CPU  │ │
  │  │ 应用      │     │ :11434        │     │ Llama 3.1    │ │
  │  │           │◀────│              │◀────│ Mistral      │ │
  │  └──────────┘     └──────────────┘     │ nomic-embed  │ │
  │                                        └──────────────┘ │
  │  数据完全不出内网 ✅                                        │
  └──────────────────────────────────────────────────────────┘

  vs. 云端模型：

  ┌──────────┐     ┌─────── 互联网 ───────┐     ┌──────────┐
  │ Symfony   │────▶│                      │────▶│ OpenAI   │
  │ 应用      │◀────│  数据通过互联网传输 ⚠️  │◀────│ API      │
  └──────────┘     └──────────────────────┘     └──────────┘
```

### 3.3 安装 Ollama

```bash
# macOS / Linux
curl -fsSL https://ollama.com/install.sh | sh

# 下载模型（根据需求选择）
ollama pull llama3.1          # 对话模型（8B 参数，需要 ~4.7GB 显存）
ollama pull mistral           # 另一个优秀的对话模型
ollama pull nomic-embed-text  # Embedding 模型（RAG 必需）
ollama pull codellama         # 代码专用模型

# 验证安装
ollama list    # 查看已下载的模型
ollama serve   # 启动服务器（默认 :11434）
```

### 3.4 使用 Ollama Bridge

```php
<?php

use Symfony\AI\Platform\Bridge\Ollama\PlatformFactory;

// Ollama 默认运行在 http://localhost:11434
$platform = PlatformFactory::create();

// 使用本地模型——API 与云端模型完全一致
$response = $platform->invoke(
    $messages,
    'llama3.1',   // 模型名就是 ollama pull 时的名字
);

echo $response->getContent();
```

### 3.5 本地 RAG——数据零泄漏

```php
use Symfony\AI\Platform\Bridge\Ollama\PlatformFactory;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Bridge\Postgres\Store;

// 使用本地平台——所有数据处理在本地完成
$platform = PlatformFactory::create();

// 向量化文档——使用本地 Embedding 模型
$vectorizer = new Vectorizer($platform, 'nomic-embed-text');
$vectorizer->vectorize($documents);

// 存储到本地 PostgreSQL
$store = new Store($connectionPool);
$store->add($documents);

// 查询也使用本地 LLM
$agentProcessor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'llama3.1', [$agentProcessor], [$agentProcessor]);
$response = $agent->call($messages);

// 整个流程中，数据完全不离开本地网络
```

### 3.6 混合部署：本地 + 云端

```php
// 敏感数据处理用本地模型——确保数据安全
$localPlatform = OllamaFactory::create();

// 非敏感的通用任务用云端模型——质量更好
$cloudPlatform = OpenAiFactory::create($_ENV['OPENAI_API_KEY']);

// 策略 1：根据数据敏感性选择平台
$platform = $this->isSensitiveData($input) ? $localPlatform : $cloudPlatform;

// 策略 2：使用 FailoverPlatform，云端优先、本地兜底
// 好处：即使云端 API 故障，服务也不中断
$platform = new FailoverPlatform([
    $cloudPlatform,    // 优先：质量好
    $localPlatform,    // 兜底：永远可用（本地服务器不会宕机）
]);
```

### 3.7 本地模型选型指南

| 模型 | 参数量 | 显存需求 | 特长 | 推荐用途 |
|------|:------:|:-------:|------|---------|
| Llama 3.1 8B | 8B | ~5GB | 通用对话 | 一般问答 |
| Llama 3.1 70B | 70B | ~40GB | 复杂推理 | 专业分析 |
| Mistral 7B | 7B | ~4GB | 效率高 | 快速响应 |
| CodeLlama | 7-34B | 4-20GB | 代码生成 | 开发辅助 |
| nomic-embed-text | - | ~0.3GB | Embedding | RAG 向量化 |
| Phi-3 | 3.8B | ~2.4GB | 轻量级 | 资源受限环境 |

---

## 4. 场景三：内容审核流水线

### 4.1 业务场景

构建一个生产级的内容审核系统，支持文本和图片审核，使用结构化输出确保审核结果格式一致，支持多平台交叉验证和缓存优化。

**典型应用**：UGC 平台、社交应用、电商评论系统、论坛管理。

### 4.2 架构概述

```
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

### 4.3 审核结果 DTO

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

### 4.4 审核服务

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

### 4.5 缓存优化

对于相同或相似内容的重复审核，使用 CachePlatform 避免重复调用 AI：

```php
use Symfony\AI\Platform\Bridge\Cache\CachePlatform;

$cachedPlatform = new CachePlatform($innerPlatform, $cache);

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

### 4.6 完整的 Symfony 集成

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

---

## 5. 场景四：CrewAI 风格多智能体团队

### 5.1 业务场景

构建一个多智能体协作团队——研究员搜集信息、写手撰写内容、编辑审核修改，模拟真实团队的分工协作。这是「多 Agent 工作流」的经典模式。

**典型应用**：内容生产流水线、研究报告自动生成、自动化写作、营销素材批量生产。

### 5.2 架构概述

```
CrewAI 风格：多智能体顺序流水线
══════════════════════════════

  任务输入："写一篇关于 Symfony AI 的技术博客"
       │
       ▼
  ┌────────────────────────────────────────┐
  │  🔍 研究员 Agent (Researcher)            │
  │  ─────────────────────────────────     │
  │  工具：Tavily 搜索 + Wikipedia          │
  │  任务：收集 Symfony AI 的最新信息         │
  │  输出：结构化研究报告（数据、引用、来源）  │
  └──────────┬─────────────────────────────┘
             │ 研究报告作为下一个 Agent 的输入
             ▼
  ┌────────────────────────────────────────┐
  │  ✍️ 写手 Agent (Writer)                 │
  │  ─────────────────────────────────     │
  │  工具：无（纯文本生成）                   │
  │  任务：基于研究报告撰写技术文章            │
  │  输出：文章草稿（Markdown 格式）          │
  └──────────┬─────────────────────────────┘
             │ 草稿作为下一个 Agent 的输入
             ▼
  ┌────────────────────────────────────────┐
  │  📝 编辑 Agent (Editor)                 │
  │  ─────────────────────────────────     │
  │  工具：无                               │
  │  任务：审核准确性、逻辑性、可读性          │
  │  输出：最终版本或修改建议                  │
  └──────────┬─────────────────────────────┘
             │
             ▼
  最终输出：完成的技术博客文章
```

### 5.3 定义角色

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Agent\Bridge\Tavily\Tavily;
use Symfony\AI\Agent\Bridge\Wikipedia\Wikipedia;
use Symfony\Component\HttpClient\HttpClient;

// 1. 研究员——负责收集信息
$httpClient = HttpClient::create();
$researchToolbox = new Toolbox([
    new Tavily($httpClient, $_ENV['TAVILY_API_KEY']),
    new Wikipedia($httpClient),
]);
$researchProcessor = new AgentProcessor($researchToolbox);
$researcher = new Agent($platform, $model,
    inputProcessors: [$researchProcessor],
    outputProcessors: [$researchProcessor],
    name: 'researcher',
);

// 2. 写手——负责撰写内容
$writer = new Agent($platform, $model, name: 'writer');

// 3. 编辑——负责审核和优化
$editor = new Agent($platform, $model, name: 'editor');
```

### 5.4 顺序流水线执行

```php
// 任务：撰写一篇关于 Symfony AI 的技术文章

// ═════════════════════════════════════════
// 步骤 1：研究员搜集信息
// ═════════════════════════════════════════
echo "🔍 研究员开始工作...\n";
$research = $researcher->call(new MessageBag(
    Message::ofUser(
        '请研究 Symfony AI 框架的最新动态和核心特性。'
        .'重点关注：架构设计、支持的平台、Agent 系统、RAG 能力。'
        .'与 LangChain（Python）和 Spring AI（Java）做简要对比。'
    ),
));
echo "📋 研究报告完成（".mb_strlen($research->getContent())." 字）\n";

// ═════════════════════════════════════════
// 步骤 2：写手基于研究报告撰写文章
// ═════════════════════════════════════════
echo "✍️ 写手开始撰写...\n";
$draft = $writer->call(new MessageBag(
    Message::forSystem('以下是研究报告，请据此撰写一篇技术博客文章：'),
    Message::ofUser($research->getContent()),
));
echo "📄 草稿完成（".mb_strlen($draft->getContent())." 字）\n";

// ═════════════════════════════════════════
// 步骤 3：编辑审核并优化
// ═════════════════════════════════════════
echo "📝 编辑开始审核...\n";
$final = $editor->call(new MessageBag(
    Message::forSystem('请审核并优化以下文章草稿。如有修改，输出完整的最终版本：'),
    Message::ofUser($draft->getContent()),
));
echo "✅ 最终版本完成\n\n";

echo $final->getContent();
```

### 5.5 使用 MultiAgent 实现自动编排

对于更复杂的场景，使用 MultiAgent 让 AI 自动决定工作流程：

```php
use Symfony\AI\Agent\MultiAgent\MultiAgent;
use Symfony\AI\Agent\MultiAgent\Handoff;

$orchestratorAgent = new Agent($platform, $model, name: 'orchestrator');
$multiAgent = new MultiAgent($orchestratorAgent, [
    new Handoff($researcher, ['搜集信息', '调研数据', '分析竞品']),
    new Handoff($writer, ['撰写文章', '写报告', '写文案']),
    new Handoff($editor, ['审核', '修改', '优化文章内容']),
], $writer);  // fallback 到写手

// MultiAgent 自动分析任务，决定路由到哪个 Agent
$response = $multiAgent->call(new MessageBag(
    Message::ofUser('请帮我完成一篇关于 PHP 性能优化的技术博客'),
));
```

### 5.6 质量保证——多轮迭代

```php
// 如果编辑对文章不满意，可以多轮迭代
$maxIterations = 3;
$article = $draft->getContent();

for ($i = 0; $i < $maxIterations; $i++) {
    $review = $editor->call(new MessageBag(
        Message::forSystem(
            '审核以下文章。如果质量达标（评分>=8/10），直接输出 "APPROVED" + 评分。'
            .'如果需要修改，输出修改后的完整版本。'
        ),
        Message::ofUser($article),
    ));

    $reviewText = $review->getContent();

    if (str_contains($reviewText, 'APPROVED')) {
        echo "✅ 编辑通过（第 {$i} 轮）\n";
        break;
    }

    // 编辑返回了修改版本——用于下一轮
    $article = $reviewText;
    echo "🔄 第 {$i} 轮修改完成\n";
}
```

---

## 6. 场景五：端到端企业知识库系统

### 6.1 业务场景

构建一个完整的企业知识库系统，整合几乎所有 Symfony AI 组件：文档索引管线、向量检索、多轮对话、工具调用、缓存和持久化。这是一个综合性的架构设计场景。

**典型应用**：企业知识管理、内部 wiki 智能问答、客户自助服务。

### 6.2 完整架构

```
端到端企业知识库系统架构
══════════════════════

  ┌─────────────────────────────────────────────────────────────┐
  │                     企业知识库系统                             │
  │                                                             │
  │  ┌──────────────────┐    ┌──────────────┐    ┌───────────┐ │
  │  │  文档索引管线       │    │  多轮对话引擎  │    │  AI Agent  │ │
  │  │  ──────────────   │    │  ──────────── │    │  ────────  │ │
  │  │  MarkdownLoader   │    │  Chat 组件     │    │  知识检索   │ │
  │  │  CsvLoader        │    │  Redis Store  │    │  工具      │ │
  │  │  TextSplitTransf. │    │              │    │            │ │
  │  │  TextTrimTransf.  │    │  initiate()  │    │  Toolbox:  │ │
  │  │  Vectorizer       │    │  submit()    │    │  - search  │ │
  │  └────────┬─────────┘    └──────┬───────┘    │  - clock   │ │
  │           │                     │            └─────┬─────┘ │
  │           │                     │                  │       │
  │           └─────────────────────┼──────────────────┘       │
  │                                 │                          │
  │                        ┌────────▼────────┐                 │
  │                        │  平台层            │                 │
  │                        │  ────────────     │                 │
  │                        │  CachePlatform   │                 │
  │                        │  └─ FailoverPlatform              │
  │                        │     ├─ OpenAI    │                 │
  │                        │     └─ Anthropic │                 │
  │                        └─────────────────┘                 │
  │                                                             │
  │  ┌──────────────────┐    ┌──────────────┐                  │
  │  │  PostgreSQL       │    │  Redis        │                  │
  │  │  (pgvector)       │    │  ──────────   │                  │
  │  │  ──────────────   │    │  对话历史     │                  │
  │  │  文档向量存储      │    │  响应缓存     │                  │
  │  └──────────────────┘    └──────────────┘                  │
  └─────────────────────────────────────────────────────────────┘
```

### 6.3 完整 AI Bundle 配置

```yaml
# config/packages/ai.yaml
ai:
    platform:
        open_ai:
            api_key: '%env(OPENAI_API_KEY)%'
        anthropic:
            api_key: '%env(ANTHROPIC_API_KEY)%'
        failover:
            type: failover
            platforms: [ai.platform.open_ai, ai.platform.anthropic]
        cached:
            type: cache
            platform: ai.platform.failover
            cache_pool: cache.ai

    store:
        knowledge_base:
            platform: ai.platform.cached
            store: ai.store.postgres

    agent:
        knowledge_assistant:
            platform: ai.platform.cached
            model: gpt-4o
            system: |
                你是企业知识库助手。

                行为准则：
                1. 使用 similarity_search 工具检索相关文档后回答问题
                2. 每个回答都引用来源文档
                3. 如果找不到相关信息，明确告知用户
                4. 不要编造信息——只基于检索到的文档回答
                5. 用中文回答，格式清晰
```

### 6.4 文档索引服务

```php
<?php

namespace App\Service;

use Symfony\AI\Store\Document\Loader\MarkdownLoader;
use Symfony\AI\Store\Document\Loader\CsvLoader;
use Symfony\AI\Store\Document\Transformer\ChainTransformer;
use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;
use Symfony\AI\Store\Document\Transformer\TextTrimTransformer;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\StoreInterface;
use Psr\Log\LoggerInterface;

class DocumentIndexService
{
    public function __construct(
        private readonly StoreInterface $store,
        private readonly Vectorizer $vectorizer,
        private readonly LoggerInterface $logger,
    ) {}

    /**
     * 索引指定目录下的所有文档
     */
    public function indexDirectory(string $directory): int
    {
        $this->logger->info('开始索引文档', ['directory' => $directory]);

        // 加载文档
        $loader = new MarkdownLoader();
        $documents = $loader->load($directory);

        // 转换管线
        $transformer = new ChainTransformer([
            new TextTrimTransformer(),
            new TextSplitTransformer(
                maxLength: 500,
                overlap: 50,
                separator: "\n\n",
            ),
        ]);
        $chunks = $transformer->transform($documents);

        // 向量化
        $this->vectorizer->vectorize($chunks);

        // 存储
        $this->store->add($chunks);

        $this->logger->info('文档索引完成', [
            'documents' => count($documents),
            'chunks' => count($chunks),
        ]);

        return count($chunks);
    }
}
```

### 6.5 对话控制器

```php
<?php

namespace App\Controller;

use Symfony\AI\Agent\AgentInterface;
use Symfony\AI\Chat\ChatInterface;
use Symfony\AI\Platform\Message\Message;
use Symfony\Component\DependencyInjection\Attribute\Target;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\StreamedResponse;
use Symfony\Component\Routing\Attribute\Route;

#[Route('/api/kb')]
class KnowledgeBaseController
{
    public function __construct(
        #[Target('knowledge_assistant')]
        private readonly AgentInterface $agent,
        private readonly ChatInterface $chat,
        private readonly PlatformInterface $platform,
        private readonly MessageStoreInterface&ManagedStoreInterface $store,
    ) {}

    /**
     * 创建新对话
     */
    #[Route('/conversations', methods: ['POST'])]
    public function newConversation(): JsonResponse
    {
        // initiate() 清空历史并保存初始消息，返回 void
        $this->chat->initiate(new MessageBag(
            Message::forSystem('你是企业知识库助手。'),
        ));

        return new JsonResponse(['status' => 'ok']);
    }

    /**
     * 提交问题（普通模式）
     */
    #[Route('/conversations/messages', methods: ['POST'])]
    public function ask(Request $request): JsonResponse
    {
        $question = $request->getPayload()->getString('question');

        try {
            // submit() 接收 UserMessage，返回 AssistantMessage
            $response = $this->chat->submit(Message::ofUser($question));

            return new JsonResponse([
                'answer' => $response->getContent(),
                'metadata' => $response->getMetadata()->all(),
            ]);
        } catch (\Throwable $e) {
            return new JsonResponse([
                'error' => '回答生成失败，请稍后重试。',
            ], 500);
        }
    }

    /**
     * 流式问答——需要绕过 Chat 组件，直接管理消息历史
     * 因为 Chat::submit() 返回 AssistantMessage，不支持流式输出
     */
    #[Route('/conversations/stream', methods: ['POST'])]
    public function stream(Request $request): StreamedResponse
    {
        $question = $request->getPayload()->getString('question');

        return new StreamedResponse(function () use ($question) {
            header('Content-Type: text/event-stream');
            header('Cache-Control: no-cache');

            // 1. 加载历史并追加用户消息
            $messages = $this->store->load();
            $messages->add(Message::ofUser($question));

            // 2. 直接调用 Platform，启用流式
            $response = $this->platform->invoke('gpt-4o', $messages, [
                'stream' => true,
            ]);

            // 3. 流式输出
            $fullContent = '';
            foreach ($response->asStream() as $chunk) {
                $fullContent .= $chunk;

                echo 'data: '.json_encode(
                    ['text' => $chunk],
                    JSON_UNESCAPED_UNICODE,
                )."\n\n";

                if (ob_get_level() > 0) {
                    ob_flush();
                }
                flush();
            }

            // 4. 保存完整回复到历史
            $messages->add(Message::ofAssistant($fullContent));
            $this->store->save($messages);

            echo "data: [DONE]\n\n";
            flush();
        });
    }
}
```

### 6.6 生产部署检查清单

| 检查项 | 状态 | 说明 |
|--------|:----:|------|
| FailoverPlatform 配置 | ☐ | 至少配置 2 个 AI 平台 |
| CachePlatform 配置 | ☐ | 使用 Redis 缓存池 |
| PostgreSQL pgvector 扩展 | ☐ | `CREATE EXTENSION vector;` |
| Redis 连接 | ☐ | 对话历史 + 响应缓存 |
| Nginx 流式配置 | ☐ | `proxy_buffering off;` |
| API Key 环境变量 | ☐ | 不要硬编码 |
| 日志和监控 | ☐ | Platform 事件监听 |
| 错误处理 | ☐ | try/catch + 优雅降级 |
| 文档定期重新索引 | ☐ | Cron / Symfony Messenger |

---

## 7. 本章小结

通过五个高级场景，我们掌握了以下生产级模式和关键内部机制：

| 模式 | 场景 | 关键组件 | 核心知识 |
|------|------|---------|---------|
| **高可用** | 容灾架构 | FailoverPlatform + CachePlatform | WeakMap 失败追踪 + UUID 缓存键陷阱 |
| **私有化** | 本地模型 | Ollama Bridge | 混合部署 + 本地 RAG |
| **安全** | 内容审核 | StructuredOutput + Agent | 预过滤 + AI 审核 + 缓存优化 |
| **协作** | 多智能体团队 | MultiAgent + 顺序流水线 | 角色分工 + 多轮迭代 |
| **端到端** | 企业知识库 | 全组件集成 | 索引管线 + 对话引擎 + 流式输出 |

---

## 8. 下一步

在 [第 12 章](12-architecture.md) 中，我们将系统性地总结高级架构设计原则——包括 Bridge 模式的设计哲学、完整的异常层次体系、处理器管线扩展点、安全防护策略、性能优化矩阵、可观测性方案和测试策略。
