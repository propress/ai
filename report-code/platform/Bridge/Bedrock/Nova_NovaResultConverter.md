# Bedrock/Nova/NovaResultConverter.php

## 概述

`NovaResultConverter` 将 AWS Bedrock 返回的 Nova 模型响应转换为平台通用的 `TextResult` 或 `ToolCallResult`，Nova 的响应结构与 Claude 和 Llama 均不同，使用 `output.message.content` 路径。

## 关键方法分析

### `convert(RawResultInterface|RawBedrockResult $result, array $options = []): ToolCallResult|TextResult`
解析 Nova 特有的响应结构：
1. **顶层校验**：检查 `output` 字段是否存在且非空。
2. **文本字段校验**：检查 `output.message.content[0].text` 路径是否存在。
3. **工具调用检测**：遍历 `output.message.content`，查找含 `toolUse` 键的条目，从 `toolUse.toolUseId`、`toolUse.name`、`toolUse.input` 构建 `ToolCall` 对象。
4. **返回结果**：有工具调用则返回 `ToolCallResult`，否则返回文本内容的 `TextResult`。

注意：当前实现在存在工具调用时，文本字段校验（步骤 2）仍会执行，若 `content[0]` 为 `toolUse` 条目而无 `text` 键，则会抛出异常（潜在的逻辑问题）。

### `getTokenUsageExtractor(): ?TokenUsageExtractorInterface`
返回 `null`，不提取 Token 用量。

## 关联关系

- 消费 `RawBedrockResult`（由 `NovaModelClient` 生成）。
- 依赖 `Nova` 模型类进行类型识别。
- 类声明为 `class`（非 `final`）。
