# Contract/MessageBagNormalizer.php

**命名空间**：`Symfony\AI\Platform\Bridge\ClaudeCode\Contract`

## 概述

`MessageBagNormalizer` 继承 `ModelContractNormalizer`，实现 `NormalizerAwareInterface`，将 `MessageBag` 序列化为 Claude Code CLI 要求的格式：提取最后一条用户消息文本作为 `prompt`，可选地提取系统消息作为 `system_prompt`，同时过滤掉系统消息以避免重复。

## 关键方法

- `normalize(mixed $data, ?string $format, array $context): array` — 反向遍历消息列表，找到最后一条用户消息的文本内容作为 `prompt`；若存在系统消息则附加 `system_prompt` 字段。
- `supportedDataClass(): string` — 返回 `MessageBag::class`，限定处理范围。
- `supportsModel(Model $model): bool` — 仅在模型为 `ClaudeCode` 实例时启用。

## 设计模式

- **反向遍历提取最后用户消息**：使用 `array_reverse()` 从后往前查找，确保取到最新的用户输入。
- **内容块感知**：同时支持字符串和数组（`type=text` 块）两种消息内容格式。

## 关联关系

- 由 `ClaudeCodeContract::create()` 注册。
- 依赖 `NormalizerAwareTrait` 委托给父级 Normalizer 处理消息列表序列化。
