# Mistral/Llm/ResultConverter.php

## 概述

将 Mistral 对话 API 的 HTTP 响应转换为平台结果类型，支持文本响应、工具调用、流式响应和多选择响应，通过 `CompletionsConversionTrait` 复用 OpenAI 兼容的流式处理逻辑。

## 关键方法分析

### `convert(RawResultInterface|RawHttpResult $result, array $options = []): ResultInterface`
主转换流程：
1. **流式模式检测**：若 `$options['stream']` 为 `true`，返回包含异步迭代器的 `StreamResult`（通过 Trait 的 `convertStream()` 方法）。
2. **HTTP 状态码验证**：非 200 状态码抛出 `RuntimeException`。
3. **choices 字段校验**：缺失时抛出 `RuntimeException`。
4. **多选择处理**：将每个 choice 通过 `convertChoice()` 转换，若仅一个则直接返回，否则包装为 `ChoiceResult`。

### `convertChoice(array $choice): ToolCallResult|TextResult`（受保护）
根据 `finish_reason` 分发：
- `tool_calls`：调用 `convertToolCall()` 构建 `ToolCallResult`。
- `stop`：从 `message.content` 提取文本，构建 `TextResult`。
- 其他值：抛出 `RuntimeException`。

### `convertToolCall(array $toolCall): ToolCall`（受保护）
将工具调用的 `function.arguments`（JSON 字符串）解码为 PHP 数组，构造 `ToolCall` 对象。

### `getTokenUsageExtractor(): TokenUsageExtractor`
返回 `TokenUsageExtractor` 实例（非 `null`，对话 API 支持 Token 用量）。

## 设计模式

- **Trait 复用**：`CompletionsConversionTrait` 提供了与 OpenAI 兼容的流式 SSE 解析逻辑，避免代码重复。
- **策略 + 分支**：`finish_reason` 驱动的转换分发。

## 关联关系

- 使用 `Bridge\Generic\Completions\CompletionsConversionTrait`（跨 Bridge 复用）。
- 返回 `ResultInterface` 的多个实现：`TextResult`、`ToolCallResult`、`StreamResult`、`ChoiceResult`。
