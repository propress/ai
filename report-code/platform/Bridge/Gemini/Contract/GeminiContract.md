# Contract/GeminiContract.php 分析报告

## 文件概述

`GeminiContract` 是 Gemini Bridge 的契约汇总类，继承 `Contract` 并静态注册所有消息/工具序列化器，是外部配置 Gemini 序列化行为的唯一入口。

## 关键方法分析

### `create(NormalizerInterface ...$normalizer): Contract`

- 调用父类 `Contract::create()`，按固定顺序注册五个内置 Normalizer
- 支持可变参数 `...$normalizer`，允许调用方追加自定义 Normalizer（注入点位于内置 Normalizer 之后）

## 注册顺序

```
AssistantMessageNormalizer
MessageBagNormalizer
ToolNormalizer
ToolCallMessageNormalizer
UserMessageNormalizer
[...自定义 Normalizer]
```

> 顺序影响 Symfony Serializer 的 `supportsNormalization` 优先级。

## 设计特点

- **工厂方法模式**：`static create()` 而非构造函数，符合 `Contract` 基类约定
- **开闭原则**：内置行为封闭，通过可变参数扩展
- VertexAi Bridge 中存在同名类 `VertexAi\Contract\GeminiContract`，二者结构完全相同，但内置 Normalizer 指向不同命名空间

## 关联文件

- `PlatformFactory.php` — 调用 `GeminiContract::create()` 作为默认 Contract
- `Contract/` 下所有 Normalizer — 被此类实例化并注册
