# `Replicate/Contract/LlamaMessageBagNormalizer.php` 分析

## 概述

继承 `ModelContractNormalizer`，将 Symfony AI 的 `MessageBag` 转换为 Replicate Llama API 所需的 `{system, prompt}` 格式，是 Replicate Bridge 中唯一的请求契约规范化器。

## 关键方法

| 方法 | 说明 |
|------|------|
| `normalize(MessageBag, format, context): array{system: string, prompt: string}` | 提取系统消息和用户消息，分别转换为 Llama 格式的 `system` 和 `prompt` 字段 |
| `supportedDataClass(): string` | 返回 `MessageBag::class`，限定处理的数据类型 |
| `supportsModel(Model): bool` | 仅对 `Meta\Llama` 实例生效 |

## 设计要点

- 依赖 `LlamaPromptConverter`（来自 `Bridge\Meta`）进行实际的消息格式转换
- 系统消息缺失时使用空的 `SystemMessage('')` 作为默认值，避免 null 问题
- 使用 `withoutSystemMessage()` 从 `MessageBag` 中移除系统消息后再转换 prompt，防止重复

## 关系

- 继承：`Contract\Normalizer\ModelContractNormalizer`
- 依赖：`Bridge\Meta\LlamaPromptConverter`
- 处理数据类型：`Message\MessageBag`
- 支持模型类型：`Bridge\Meta\Llama`
- 被 `PlatformFactory` 通过 `Contract::create()` 注册
