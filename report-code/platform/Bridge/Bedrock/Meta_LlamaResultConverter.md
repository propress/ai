# Bedrock/Meta/LlamaResultConverter.php

## 概述

`LlamaResultConverter` 将 AWS Bedrock 返回的 Llama 模型响应（`RawBedrockResult`）转换为 `TextResult`，是三个 ResultConverter 中最简单的一个，仅支持纯文本输出（不支持工具调用）。

## 关键方法分析

### `supports(Model $model): bool`
检查模型是否为 `Llama` 实例。

### `convert(RawResultInterface|RawBedrockResult $result, array $options = []): TextResult`
Llama 的响应格式与 Claude 和 Nova 不同：
1. **字段检查**：检查 `generation` 字段（而非 `content` 或 `output`），若缺失则抛出 `RuntimeException`。
2. **直接返回**：将 `$data['generation']` 包装为 `TextResult` 返回，无工具调用处理逻辑。

### `getTokenUsageExtractor(): ?TokenUsageExtractorInterface`
返回 `null`，不提取 Token 用量。

## 设计模式

- **策略模式**：通过 `supports()` 方法限定作用范围。
- **最小化原则**：仅实现必要功能，不添加未支持的特性（工具调用、结构化输出）。

## 关联关系

- 消费 `RawBedrockResult`（由 `LlamaModelClient` 生成）。
- 依赖 `Bridge/Meta/Llama` 模型类。
- 类声明为 `class`（非 `final`），与 `ClaudeResultConverter` 的 `final` 一致性不同，可被子类扩展。
- 输出类型仅为 `TextResult`（相比 Claude/Nova 的联合类型 `TextResult|ToolCallResult` 更受限）。
