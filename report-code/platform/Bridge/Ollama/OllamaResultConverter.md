# OllamaResultConverter.php 分析

## 概述

`OllamaResultConverter` 是 Ollama Bridge 的结果转换器，实现 `ResultConverterInterface`，处理对话补全、向量嵌入和流式三种响应类型，并在流式模式下实现 Tool Call 的累积聚合。

## 关键方法分析

### `convert(RawResultInterface $result, array $options): ResultInterface`
根据 `$options['stream']` 分发：
- 流式 → `StreamResult`（包裹 `convertStream()` 生成器）
- 非流式 → 检查响应数据是否含有 `embeddings` 键，分发至 `doConvertEmbeddings()` 或 `doConvertCompletion()`

### `doConvertCompletion(array $data): ResultInterface`（私有）
检查响应中的 `message.content` 字段（非空则返回 `TextResult`），并处理 `message.tool_calls`：若存在则返回 `ToolCallResult`（工具调用结果优先于文本结果）。

### `doConvertEmbeddings(array $data): ResultInterface`（私有）
将 `embeddings` 数组（每个元素为 float 数组）映射为 `Vector[]`，包装为 `VectorResult` 返回。

### `convertStream(RawResultInterface $result): \Generator`（私有）
遍历数据流，实现流式工具调用的聚合逻辑：
1. 若当前块含有 `tool_calls`，将其追加到 `$toolCalls` 累积数组
2. 若 `$toolCalls` 非空且 `done === true`，yield 完整的 `ToolCallResult`
3. 每个块均 yield 对应的 `OllamaMessageChunk`

## 关键模式

- **流式 Tool Call 聚合**：Ollama 在最后一个块中一次性发送完整的工具调用，转换器通过状态累积确保 `ToolCallResult` 仅在流结束时被 yield，且与最终的 `OllamaMessageChunk` 一起发出。
- **嵌入/补全自动区分**：通过响应体中 `embeddings` 键的存在性自动区分请求类型，无需调用方额外传递类型标志。

## 关联关系

- 被 `PlatformFactory` 注入 `Platform`。
- 流式模式 yield `OllamaMessageChunk` 对象，非流式模式返回 `TextResult`、`ToolCallResult` 或 `VectorResult`。
- `getTokenUsageExtractor()` 返回 `TokenUsageExtractor` 实例。
