# Mistral/Llm/TokenUsageExtractor.php

## 概述

从 Mistral 对话 API 的响应中提取 Token 用量信息，同时处理流式响应的特殊情况（流式模式下无法即时提取）和速率限制头信息。

## 关键方法分析

### `extract(RawResultInterface $rawResult, array $options = []): ?TokenUsage`
1. **流式模式短路**：若 `$options['stream']` 为 `true`，直接返回 `null`（流式响应的 Token 用量分散在各 SSE 块中，需要由调用方手动汇总）。
2. **类型检查**：验证底层响应实现 `ResponseInterface`，否则返回 `null`。
3. **速率限制头提取**：读取 `x-ratelimit-limit-tokens-minute` 和 `x-ratelimit-limit-tokens-month` 响应头。
4. **响应体用量提取**：从 `usage` 字段读取 `prompt_tokens`、`completion_tokens`、`total_tokens`。

与 `Embeddings\TokenUsageExtractor` 的差异：本类额外提取 `completionTokens`（对话模型有输出 Token）。

## 设计模式

- **快速返回（Early Return）**：流式模式检查置于方法最顶部，避免不必要的处理。
- **可选值安全处理**：所有字段均通过 `?? null` 安全取值，确保部分响应时不抛出错误。

## 关联关系

- 实现 `TokenUsageExtractorInterface`。
- 由 `Llm\ResultConverter::getTokenUsageExtractor()` 创建并返回。
- 与 `Embeddings\TokenUsageExtractor` 结构完全对称，但包含 `completionTokens` 字段。
