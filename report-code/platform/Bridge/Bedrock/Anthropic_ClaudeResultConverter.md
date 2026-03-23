# Bedrock/Anthropic/ClaudeResultConverter.php

## 概述

`ClaudeResultConverter` 负责将 AWS Bedrock 返回的 Claude 模型原始响应（`RawBedrockResult`）转换为平台统一的结果类型（`TextResult` 或 `ToolCallResult`）。

## 关键方法分析

### `supports(Model $model): bool`
检查模型是否为 `Claude` 实例。

### `convert(RawResultInterface|RawBedrockResult $result, array $options = []): ToolCallResult|TextResult`
从 `RawBedrockResult` 中提取 JSON 数据并进行转换：
1. **内容存在性校验**：若 `content` 字段缺失或为空数组，抛出 `RuntimeException`。
2. **字段完整性校验**：若 `content[0]` 既无 `text` 也无 `type` 字段，抛出 `RuntimeException`。
3. **工具调用检测**：遍历所有 `content` 条目，若 `type` 为 `tool_use`，则创建 `ToolCall` 对象（包含 `id`、`name`、`input`）。
4. **结果返回**：若存在工具调用则返回 `ToolCallResult`，否则返回 `TextResult`。

### `getTokenUsageExtractor(): ?TokenUsageExtractorInterface`
返回 `null`，表明 Bedrock 上的 Claude **不提取 Token 用量信息**。

## 设计模式

- **策略模式**：`supports()` 方法确保该转换器仅作用于 Claude 模型。
- **防御性编程**：多层次的响应结构校验，使用项目专用异常 `RuntimeException`（而非 PHP 原生异常）。

## 关联关系

- 消费 `RawBedrockResult`（由 `ClaudeModelClient` 生成）。
- 依赖 `Bridge/Anthropic/Claude` 模型类进行类型识别。
- 输出 `TextResult` 或 `ToolCallResult`（平台通用结果接口）。
