# Bedrock/Nova/Contract/AssistantMessageNormalizer.php

## 概述

将平台通用的 `AssistantMessage` 序列化为 Amazon Nova 模型所要求的 JSON 格式，支持纯文本内容与工具调用两种情形。

## 关键方法分析

### `normalize(mixed $data, ?string $format = null, array $context = []): array`
处理 `AssistantMessage`（参数 `$data`）的两种情形：
- **含工具调用**：将每个 `ToolCall` 映射为 `toolUse` 结构（字段名 `toolUseId`/`name`/`input`），空参数时使用 `new \stdClass()` 以确保序列化为 JSON 对象而非数组。
- **纯文本**：返回 `content: [['text' => '...']]` 结构。

返回类型带有详细的 PHPDoc 数组形状注解。

## 设计模式

- **模板方法模式**：继承 `ModelContractNormalizer`，通过 `supportedDataClass()` 和 `supportsModel()` 两个方法限定作用范围。
- **条件分支而非多态**：工具调用与文本内容的区分通过 `hasToolCalls()` 判断，而非子类化。

## 关联关系

- 继承 `Platform\Contract\Normalizer\ModelContractNormalizer`。
- 仅对 `Nova` 模型实例生效（`supportsModel()` 检查）。
- 处理的数据类：`AssistantMessage`（`supportedDataClass()` 返回值）。
- 与 Anthropic 的同名规范化器类似，但字段名不同（Nova 用 `toolUse`，Anthropic 用 `tool_use`）。
