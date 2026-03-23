# Bedrock/Nova/Contract/MessageBagNormalizer.php

## 概述

将 `MessageBag`（消息集合）序列化为 Amazon Nova API 所需的格式，负责将系统消息从消息列表中分离并放入独立的 `system` 字段。

## 关键方法分析

### `normalize(mixed $data, ?string $format = null, array $context = []): array`
处理 `MessageBag`（参数 `$data`）：
1. **系统消息分离**：若存在系统消息，将其内容放入 `system: [['text' => '...']]` 数组结构（Nova 要求数组格式，区别于某些 API 使用字符串）。
2. **消息列表递归序列化**：调用 `$this->normalizer->normalize()`（通过 `NormalizerAwareTrait` 注入的父级规范化器）对 `withoutSystemMessage()` 后的消息列表进行递归处理。

## 设计模式

- **装饰器/委托模式**：通过 `NormalizerAwareInterface` + `NormalizerAwareTrait` 获取链中的父级规范化器，实现递归序列化。
- **关注点分离**：仅处理消息包的顶层结构（系统消息分离与消息列表封装），具体消息内容的序列化委托给子规范化器。

## 关联关系

- 实现 `NormalizerAwareInterface`，在 Symfony Serializer 链中被自动注入父级规范化器。
- 仅对 `Nova` 模型实例生效。
- 处理的数据类：`MessageBag`。
- 调用 `MessageBag::withoutSystemMessage()` 和 `MessageBag::getSystemMessage()` 方法。
