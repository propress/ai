# 高可用 AI 服务架构

## 业务场景

你在生产环境中运行 AI 应用。某天 OpenAI API 突然限流或宕机，整个系统瘫痪。老板问：为什么没有备份方案？现在你需要构建高可用的 AI 服务：多平台故障转移（OpenAI 挂了自动切到 Anthropic）、响应缓存（重复问题不重复调用 API）、成本控制（简单任务用便宜模型，复杂任务用贵模型）。

**典型应用：** 企业级 AI 服务高可用、API 成本优化、响应缓存加速、多模型策略路由

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform Bridge Failover** | 多平台故障自动切换 |
| **Platform Bridge Cache** | 响应缓存，避免重复 API 调用 |
| **Platform (OpenAI)** | 主要 AI 平台 |
| **Platform (Anthropic)** | 备用 AI 平台 |
| **Platform (Ollama)** | 本地模型兜底 |

## 前置准备

```bash
composer require symfony/ai-platform
composer require symfony/ai-platform-openai
composer require symfony/ai-platform-anthropic
composer require symfony/ai-platform-ollama      # 可选：本地兜底
composer require symfony/cache                   # Symfony 缓存组件
```

---

## Step 1：基础故障转移（Failover）

当主平台不可用时，自动切换到备用平台。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory as AnthropicFactory;
use Symfony\AI\Platform\Bridge\Failover\PlatformFactory as FailoverFactory;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory as OpenAiFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create();

// 创建主平台和备用平台
$openai = OpenAiFactory::create($_ENV['OPENAI_API_KEY'], $httpClient);
$anthropic = AnthropicFactory::create($_ENV['ANTHROPIC_API_KEY'], $httpClient);

// 故障转移：OpenAI → Anthropic
$platform = FailoverFactory::create([$openai, $anthropic]);

// 正常使用 — 如果 OpenAI 失败，自动重试 Anthropic
$messages = new MessageBag(
    Message::ofUser('什么是依赖注入？用 PHP 代码举例。'),
);

$result = $platform->invoke('gpt-4o-mini', $messages);
echo $result->asText() . "\n";
// 如果 OpenAI 不可用，会自动用 Anthropic 的对应模型回答
```

---

## Step 2：响应缓存（Cache）

相同的问题不重复调用 API，直接从缓存返回。

```php
<?php

use Symfony\AI\Platform\Bridge\Cache\PlatformFactory as CacheFactory;
use Symfony\Component\Cache\Adapter\FilesystemAdapter;

// 创建缓存适配器（也可用 Redis、Memcached 等）
$cache = new FilesystemAdapter(
    namespace: 'ai_responses',
    defaultLifetime: 3600,    // 缓存 1 小时
    directory: '/tmp/ai-cache',
);

// 包装平台，添加缓存层
$cachedPlatform = CacheFactory::create($openai, $cache);

// 第一次调用 — 真实 API 请求
$result1 = $cachedPlatform->invoke('gpt-4o-mini', new MessageBag(
    Message::ofUser('PHP 的 readonly 属性有什么用？'),
));
echo "首次调用（API）：" . mb_substr($result1->asText(), 0, 100) . "...\n";

// 第二次调用同样的问题 — 直接从缓存返回，0 延迟，0 成本
$result2 = $cachedPlatform->invoke('gpt-4o-mini', new MessageBag(
    Message::ofUser('PHP 的 readonly 属性有什么用？'),
));
echo "缓存命中（0 成本）：" . mb_substr($result2->asText(), 0, 100) . "...\n";
```

---

## Step 3：缓存 + 故障转移组合

```php
<?php

// 终极组合：缓存 → 故障转移（OpenAI → Anthropic）
// 请求流程：缓存命中？→ 返回 | 未命中？→ OpenAI → 失败？→ Anthropic

$failoverPlatform = FailoverFactory::create([$openai, $anthropic]);
$fullPlatform = CacheFactory::create($failoverPlatform, $cache);

// 这个调用：
// 1. 先查缓存 → 有就直接返回
// 2. 没缓存 → 调 OpenAI
// 3. OpenAI 挂了 → 自动切 Anthropic
// 4. 成功后写入缓存
$result = $fullPlatform->invoke('gpt-4o-mini', new MessageBag(
    Message::ofUser('解释 PHP Fiber 的工作原理'),
));

echo $result->asText() . "\n";
```

---

## Step 4：智能模型路由

根据任务复杂度选择不同模型，节省成本。

```php
<?php

namespace App\Dto;

final class TaskComplexity
{
    /**
     * @param string $level       复杂度（simple/medium/complex）
     * @param string $reason      判断理由
     * @param string $suggestedModel 建议模型
     */
    public function __construct(
        public readonly string $level,
        public readonly string $reason,
        public readonly string $suggestedModel,
    ) {
    }
}
```

```php
<?php

use App\Dto\TaskComplexity;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$routerPlatform = OpenAiFactory::create(
    $_ENV['OPENAI_API_KEY'],
    $httpClient,
    eventDispatcher: $dispatcher,
);

/**
 * 智能路由：用便宜模型判断复杂度，再用合适模型处理
 */
function smartRoute(object $routerPlatform, object $openai, string $userQuery): string
{
    // 第一步：用最便宜的模型判断复杂度
    $complexityResult = $routerPlatform->invoke('gpt-4o-mini', new MessageBag(
        Message::forSystem(
            '判断用户问题的复杂度。'
            . "\nsimple: 事实性问题、简单翻译、格式转换 → 建议 gpt-4o-mini"
            . "\nmedium: 需要推理、代码编写、内容创作 → 建议 gpt-4o"
            . "\ncomplex: 复杂分析、多步推理、专业领域 → 建议 gpt-4o"
        ),
        Message::ofUser($userQuery),
    ), [
        'response_format' => TaskComplexity::class,
    ]);

    $complexity = $complexityResult->asObject();
    $model = $complexity->suggestedModel;

    echo "📊 复杂度：{$complexity->level} → 使用 {$model}\n";
    echo "   理由：{$complexity->reason}\n";

    // 第二步：用合适的模型处理
    $result = $openai->invoke($model, new MessageBag(
        Message::ofUser($userQuery),
    ));

    return $result->asText();
}

// 简单问题 → gpt-4o-mini（便宜）
echo smartRoute($routerPlatform, $openai, 'PHP 8 什么时候发布的？') . "\n\n";

// 复杂问题 → gpt-4o（贵但强）
echo smartRoute($routerPlatform, $openai, '设计一个高并发电商秒杀系统的架构方案') . "\n\n";
```

---

## Step 5：三层兜底架构

```php
<?php

use Symfony\AI\Platform\Bridge\Ollama\PlatformFactory as OllamaFactory;

// 生产环境推荐的三层架构：
// 第1层：OpenAI（最强模型，主力）
// 第2层：Anthropic（同等质量，备用）
// 第3层：Ollama 本地模型（质量稍低，但永不宕机）

$openai = OpenAiFactory::create($_ENV['OPENAI_API_KEY'], $httpClient);
$anthropic = AnthropicFactory::create($_ENV['ANTHROPIC_API_KEY'], $httpClient);
$ollama = OllamaFactory::create($httpClient);  // 本地运行，无需 API key

// 三层故障转移
$resilientPlatform = FailoverFactory::create([
    $openai,      // 优先
    $anthropic,   // 备用
    $ollama,      // 终极兜底
]);

// 再加缓存
$productionPlatform = CacheFactory::create(
    $resilientPlatform,
    new FilesystemAdapter('ai', 3600, '/tmp/ai-cache'),
);

// 这个调用几乎不可能失败
$result = $productionPlatform->invoke('gpt-4o-mini', new MessageBag(
    Message::ofUser('什么是微服务架构？'),
));

echo $result->asText() . "\n";
```

---

## Step 6：Redis 缓存（生产推荐）

```php
<?php

use Symfony\Component\Cache\Adapter\RedisAdapter;

// 生产环境用 Redis 缓存（更快、支持分布式）
$redis = RedisAdapter::createConnection('redis://localhost:6379');
$redisCache = new RedisAdapter($redis, 'ai_responses', 7200);

$productionPlatform = CacheFactory::create(
    FailoverFactory::create([$openai, $anthropic]),
    $redisCache,
);

// 优势：
// 1. 多个 PHP 进程共享缓存
// 2. 分布式部署也能命中缓存
// 3. 精确控制 TTL
// 4. 可手动清除特定缓存
```

---

## 成本对比

| 策略 | 每月 API 调用量 | 月成本 | 可用性 |
|------|---------------|--------|--------|
| 无优化 | 100,000 | ~$200 | 99.5% |
| + 缓存 | 30,000（70% 命中） | ~$60 | 99.5% |
| + 故障转移 | 30,000 | ~$65 | 99.99% |
| + 智能路由 | 30,000（60% 用 mini） | ~$35 | 99.99% |
| + 本地兜底 | 30,000 | ~$35 | 99.999% |

---

## 完整架构

```
用户请求
    │
    ▼
[缓存层] ──命中──► 直接返回（0 成本，<1ms）
    │
    未命中
    │
    ▼
[智能路由] → 判断复杂度 → 选择模型
    │
    ▼
[故障转移链]
    ├─ 第1层：OpenAI ────► 成功 → 写缓存 → 返回
    │   失败 ↓
    ├─ 第2层：Anthropic ──► 成功 → 写缓存 → 返回
    │   失败 ↓
    └─ 第3层：Ollama ────► 成功 → 写缓存 → 返回
                              （本地，永不宕机）
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `FailoverFactory::create()` | 多平台自动故障转移 |
| `CacheFactory::create()` | 响应缓存包装 |
| `FilesystemAdapter` | 文件缓存（开发用） |
| `RedisAdapter` | Redis 缓存（生产用） |
| 智能路由 | 用便宜模型判断复杂度，选合适模型 |
| 三层兜底 | OpenAI → Anthropic → Ollama |
| 成本优化 | 缓存 + 路由，月成本降低 80%+ |

## 下一步

如果你想用本地模型保护数据隐私（不发送数据到外部 API），请看 [29-local-model-private-deployment.md](./29-local-model-private-deployment.md)。
