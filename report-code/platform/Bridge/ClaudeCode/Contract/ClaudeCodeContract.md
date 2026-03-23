# Contract/ClaudeCodeContract.php

**命名空间**：`Symfony\AI\Platform\Bridge\ClaudeCode\Contract`

## 概述

`ClaudeCodeContract` 继承平台基类 `Contract`，通过静态工厂方法 `create()` 注册 `MessageBagNormalizer`，构建 ClaudeCode 专用的序列化契约，并支持注入额外的 `NormalizerInterface` 实例。

## 关键方法

- `create(NormalizerInterface ...$normalizer): Contract` — 以 `MessageBagNormalizer` 为首，加上可变参数 `$normalizer`，调用父类 `create()` 构造 Contract 实例。

## 设计模式

- **契约工厂**：通过静态方法集中注册序列化器，确保 ClaudeCode 特定的消息格式始终被正确处理。

## 关联关系

- 由 `PlatformFactory` 在未提供自定义 `$contract` 时作为默认契约使用。
- 内部注册 `MessageBagNormalizer`。
