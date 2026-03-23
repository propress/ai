# Contract/GeminiContract.php 分析报告

## 文件概述

VertexAi Bridge 的契约汇总类，注册所有针对 `VertexAi\Gemini\Model` 的消息/工具序列化器。命名为 `GeminiContract` 反映了 VertexAi 通过 Gemini 模型提供服务的本质。

## 关键方法分析

### `create(NormalizerInterface ...$normalizer): Contract`

与 Gemini Bridge 的 `GeminiContract` 结构完全相同，注册五个同名但不同命名空间的 Normalizer：

```
VertexAi\Contract\AssistantMessageNormalizer
VertexAi\Contract\MessageBagNormalizer
VertexAi\Contract\ToolNormalizer
VertexAi\Contract\ToolCallMessageNormalizer
VertexAi\Contract\UserMessageNormalizer
[...自定义 Normalizer]
```

## 与 Gemini Bridge GeminiContract 的差异

| 方面 | Gemini | VertexAi |
|------|--------|---------|
| 内置 Normalizer 命名空间 | `Gemini\Contract\*` | `VertexAi\Contract\*` |
| Normalizer 的 supportsModel 目标 | `instanceof Gemini\Gemini` | `instanceof VertexAi\Gemini\Model` |
| 字段风格（MessageBag） | snake_case | camelCase |

## 设计特点

- 两个桥接各自拥有独立的 `GeminiContract`，互不依赖，确保协议逻辑完全隔离
- 调用方可在 `PlatformFactory::create()` 的 `$contract` 参数中传入自定义 Contract 替换默认值

## 关联文件

- `PlatformFactory.php` — 调用 `GeminiContract::create()` 作为默认 Contract
- `VertexAi\Contract\*` 所有 Normalizer — 被此类实例化并注册
