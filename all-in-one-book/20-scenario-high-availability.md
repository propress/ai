# 第 20 章：实战 —— 高可用 AI 服务架构

## 学习目标

- 理解生产环境中 AI 服务高可用的必要性
- 掌握 FailoverPlatform 的内部机制和 WeakMap 追踪策略
- 掌握 CachePlatform 的缓存键构成和使用陷阱
- 学会组合使用缓存与容灾构建三层防御架构
- 了解 AI Bundle 中的高可用配置方式

## 前置知识

- 熟悉 Symfony AI Platform 的基本使用（`invoke()` 方法）
- 了解 OpenAI / Anthropic 等平台的 API 调用方式
- 了解 Symfony Cache 组件（TagAwareAdapter、RedisAdapter）
- 了解速率限制器（RateLimiter）的基本概念

## 业务场景描述

生产环境中，AI 服务需要高可用性保障。单一平台可能出现故障、限速或延迟。解决方案是使用多平台容灾和智能缓存。这是每个生产级 AI 应用的基础设施。

**典型应用**：任何需要 99.9%+ 可用性的 AI 服务。

## 架构概述

```text
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

## 环境准备

确保已安装以下 Composer 包：

```bash
composer require symfony/ai-platform
composer require symfony/cache
```

需要的环境变量：

```dotenv
OPENAI_API_KEY=your-openai-key
ANTHROPIC_API_KEY=your-anthropic-key
```

## 核心实现

### FailoverPlatform 内部机制

`FailoverPlatform` 不是简单的轮询——它使用 `WeakMap` 追踪失败平台和速率限制器实现智能恢复：

```php
<?php

use Symfony\AI\Platform\Bridge\Failover\FailoverPlatform;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory as OpenAiFactory;
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory as AnthropicFactory;

// 创建多个平台实例
$openai = OpenAiFactory::create($_ENV['OPENAI_API_KEY']);
$anthropic = AnthropicFactory::create($_ENV['ANTHROPIC_API_KEY']);

// 创建容灾平台——接受平台数组和速率限制器，按顺序尝试
$platform = new FailoverPlatform([$openai, $anthropic], $rateLimiterFactory);

// 使用方式与单平台完全一致
$response = $platform->invoke($model, $messages);
```

> **知识扩展：FailoverPlatform 的内部行为**
>
> FailoverPlatform 的核心是一个闭包驱动的重试循环：
>
> ```
> FailoverPlatform::invoke() 的内部流程
> ═══════════════════════════════════
>
> for 每个 Platform in 列表:
> │
> ├─ 1. 检查速率限制器（RateLimiter）
> │ 如果该平台之前失败过，速率限制器控制多久后重试
> │
> ├─ 2. 检查失败记录（WeakMap<Platform, timestamp>）
> │ 如果该平台在 WeakMap 中有失败记录，且未通过速率限制器检查
> │ → 跳过该平台，尝试下一个
> │
> ├─ 3. 尝试调用
> │ try {
> │ $result = $platform->invoke($model, $messages, $options);
> │ // 成功：从失败记录中移除该平台
> │ unset($failedPlatforms[$platform]);
> │ return $result;
> │ }
> │
> └─ 4. 失败处理
> catch (\Throwable $e) {
> // 记录失败：$failedPlatforms[$platform] = now
> // 记录日志
> // 继续尝试下一个平台
> }
>
> // 所有平台都失败 → throw RuntimeException
> ```
>
> **关键设计**：
> - 使用 `WeakMap` 而不是普通数组——当平台对象被垃圾回收时，失败记录也自动清除
> - 速率限制器让失败的平台在一段时间后自动恢复尝试（而不是永久标记为失败）
> - 捕获的是 `\Throwable`（不仅是 AI 异常），确保任何类型的故障都能被处理

### CachePlatform 内部机制

```php
use Symfony\AI\Platform\Bridge\Cache\CachePlatform;
use Symfony\Component\Cache\Adapter\RedisAdapter;
use Symfony\Component\Cache\Adapter\TagAwareAdapter;

$cache = new TagAwareAdapter(new RedisAdapter(/* Redis 连接 */));

$platform = new CachePlatform(
    platform: $innerPlatform,
    cache: $cache,
);

// 缓存调用——必须提供 prompt_cache_key
$response = $platform->invoke($model, $messages, [
    'prompt_cache_key' => 'product-faq',     // 缓存键前缀（必须）
    'prompt_cache_ttl' => 3600,               // 缓存 TTL（可选，默认永不过期）
]);
```

> **知识扩展：CachePlatform 的缓存键构成**
>
> CachePlatform 的缓存键由三部分组成：
>
> ```
> 缓存键 = prompt_cache_key + 模型名(camelCase) + 输入哈希
>
> 输入哈希的计算规则：
> - string 输入 → MD5($input)
> - array 输入 → MD5(json_encode($input))
> - MessageBag → MessageBag::getId()->toString() ← 注意！
> ```
>
> **重要陷阱**：`MessageBag` 使用 UUID v7 作为 ID，每次 `new MessageBag()` 都会生成新 ID。这意味着：
> ```php
> // 这两个 MessageBag 内容相同，但 ID 不同——不会命中缓存！
> $bag1 = new MessageBag(Message::ofUser('你好'));
> $bag2 = new MessageBag(Message::ofUser('你好'));
> // $bag1->getId() !== $bag2->getId()
>
> // 要实现内容缓存，需要复用同一个 MessageBag 实例
> // 或者在缓存键中使用内容哈希而非 MessageBag ID
> ```
>
> CachePlatform 还会：
> - 使用 Tag-Aware Caching（按模型名标记），支持按模型清除缓存
> - 在缓存的 Metadata 中添加 `cached: true`、`cache_key`、`cached_at` 时间戳
> - 使用 `ResultNormalizer` + Symfony Serializer 进行结果序列化/反序列化

### 组合使用——完整的高可用调用链

```php
// 构建三层防御：缓存 → 容灾 → 实际平台

// 第 1 步：创建实际平台
$openai = OpenAiFactory::create($_ENV['OPENAI_API_KEY']);
$anthropic = AnthropicFactory::create($_ENV['ANTHROPIC_API_KEY']);

// 第 2 步：容灾平台
$failover = new FailoverPlatform([$openai, $anthropic], $rateLimiterFactory);

// 第 3 步：缓存平台（包装容灾平台）
$platform = new CachePlatform($failover, cache: $cache);

// 请求的完整路径：
// 1. CachePlatform 检查缓存 → 命中则直接返回（0 成本、<1ms）
// 2. 缓存未命中 → FailoverPlatform 尝试 OpenAI
// 3. OpenAI 失败 → FailoverPlatform 自动切换到 Anthropic
// 4. 成功响应 → CachePlatform 写入缓存
// 5. 下次相同请求 → 步骤 1 直接返回
```

### AI Bundle 配置

```yaml
# config/packages/ai.yaml
ai:
    platform:
        openai:
            api_key: '%env(OPENAI_API_KEY)%'
        anthropic:
            api_key: '%env(ANTHROPIC_API_KEY)%'
        # 容灾平台
        failover:
            main:
                platforms:
                    - ai.platform.openai
                    - ai.platform.anthropic
                rate_limiter: limiter.failover
        # 缓存平台（包装容灾平台）
        cache:
            cached:
                platform: ai.platform.failover.main
                service: cache.ai
```

## 运行与验证

1. 配置好环境变量和 Redis/缓存后端
2. 首次请求：观察日志确认走的是 FailoverPlatform → OpenAI
3. 再次发送相同请求：确认缓存命中（响应元数据包含 `cached: true`）
4. 模拟 OpenAI 故障（如临时修改 API Key），确认自动切换到 Anthropic
5. 恢复 OpenAI API Key，等速率限制器窗口过后确认 OpenAI 重新可用

## 错误处理

- **全部平台失败**：FailoverPlatform 在所有平台尝试后抛出 `RuntimeException`，应在业务层 try/catch 并返回预设回复或引导至人工客服
- **缓存后端不可用**：CachePlatform 应配置为缓存失败时透传请求（不因缓存故障阻塞服务）
- **速率限制**：通过 RateLimiter 控制对已失败平台的重试频率，避免雪崩

## 生产环境注意事项

### 成本分析

假设一个月 100,000 次 AI 调用：

| 优化措施 | 实际 API 调用次数 | 成本（以 GPT-4o 计） |
|---------|:----------------:|:-----------------:|
| 无优化 | 100,000 | ~$500 |
| CachePlatform（70% 命中率） | 30,000 | ~$150 |
| + 简单任务用 mini 模型 | 30,000 | ~$35 |

> **经验值**：FAQ 类应用的缓存命中率通常可达 60-80%，API 成本降低效果显著。

## 扩展方向

- 添加更多平台（如 Google Gemini、Azure OpenAI）到 FailoverPlatform 列表
- 实现基于延迟的智能路由（优先选择响应最快的平台）
- 配置分层缓存策略（内存缓存 + Redis 缓存）
- 添加 Prometheus/Grafana 监控面板，跟踪缓存命中率和平台切换频率

## 完整源代码

本章涉及的完整配置和代码片段已在各小节中给出。核心要点：

1. 使用 `FailoverPlatform` 包装多个平台实现容灾
2. 使用 `CachePlatform` 包装容灾平台实现缓存
3. 在 AI Bundle 中通过 YAML 配置声明式组合

## 下一步

下一个场景我们将学习如何在本地部署私有化模型，实现数据零泄漏——参见 [第 21 章：实战 —— 本地模型私有化部署](21-scenario-local-model.md)。
