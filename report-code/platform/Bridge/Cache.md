# Cache Bridge 分析报告

## 文件概述
`Cache` Bridge 是 `PlatformInterface` 的**缓存装饰器**，通过包装任意 Platform 实例，对相同输入的调用返回缓存结果，避免重复 API 调用和费用浪费。

## 目录结构
```
Cache/
├── CachePlatform.php    # 装饰器（核心逻辑）
└── ResultNormalizer.php # Result 对象的序列化/反序列化
```

## CachePlatform 分析

### 类定义
- **类型**: `final class`，实现 `PlatformInterface`
- **装饰目标**: 任意 `PlatformInterface` 实现

### 构造参数
| 参数 | 说明 |
|---|---|
| `$platform` | 被装饰的真实 Platform |
| `$clock` | 时钟接口（默认 MonotonicClock，用于记录缓存时间戳） |
| `$cache` | Symfony Cache TagAwareAdapter（带标签支持，可按模型名失效） |
| `$serializer` | Result 序列化器（默认内置序列化器） |
| `$cacheKey` | 全局缓存键前缀 |
| `$cacheTtl` | 全局 TTL（秒） |

### 缓存激活条件
`invoke()` 方法只在**同时满足**以下条件时使用缓存：
1. `$cache` 不为 null
2. options 中有 `prompt_cache_key`（且不为空字符串）

否则直接透传给真实 Platform。

### 缓存键构成
```
<prompt_cache_key> + <model-name-camel-case> + <input-hash>
```
- string 输入 → `md5($input)`
- array 输入 → `json_encode($input)`
- MessageBag 输入 → `$input->getId()->toString()`（UUID，唯一标识消息历史）

**技巧**：使用 `Symfony\Component\String\UnicodeString::join()` 拼接，优雅处理多部分键名。

### 缓存存储内容
```php
[
    'result' => $serializer->normalize($result),  // Result 对象序列化
    'raw_data' => $deferredResult->getRawResult()->getData(), // 原始响应数据
    'metadata' => $result->getMetadata()->all(),  // TokenUsage 等元数据
    'cached_at' => $clock->now()->getTimestamp(),
    'cache_key' => $cacheKey,
]
```

### 恢复时的元数据注入
从缓存恢复时，额外添加 `cached: true`、`cache_key`、`cached_at` 到 Metadata，让消费者感知到这是缓存结果。

### Cache Tag
使用 `$item->tag($modelNameCamelCase)` 为每个缓存项打标签，支持 `cache->invalidateTags(['gpt4oMini'])` 按模型批量失效。

## ResultNormalizer 分析

### 职责
将 `ResultInterface` 子类序列化/反序列化，保存到 Symfony Cache。

### 序列化格式
```json
{"class": "Symfony\\AI\\Platform\\Result\\TextResult", "payload": "Hello World"}
```

### 支持的 Result 类型
| 类型 | 序列化方式 |
|---|---|
| TextResult | payload = 内容字符串 |
| BinaryResult | payload = base64 + mimeType |
| ToolCallResult | payload = toolCall[] 数组 |
| VectorResult | payload = [{data, dimensions}][] |
| ChoiceResult | 递归序列化每个 choice |
| ObjectResult | payload = {type, content} |
| **StreamResult** | **不支持（抛出异常）** |

**注意**：StreamResult 无法缓存（流式响应是实时的），尝试缓存流式结果会抛出 `InvalidArgumentException`。

## 设计模式
**装饰器（Decorator）**：完全透明地包装 Platform，调用方无需知道缓存层的存在。配合 Symfony AI Bundle 的 `@autoconfigure`，可以通过配置一行启用/禁用缓存。

## 与其他文件的关系
- 包装任意 `PlatformInterface` 实现
- 依赖 `symfony/cache` 的 `CacheInterface & TagAwareAdapterInterface`
- `ResultNormalizer` 处理 `platform` 核心层的所有 Result 类型
