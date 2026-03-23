# Mistral/Embeddings/TokenUsageExtractor.php

## 概述

从 Mistral 嵌入 API 的 HTTP 响应中提取 Token 用量信息，同时读取响应体中的 `usage` 字段和响应头中的速率限制信息。

## 关键方法分析

### `extract(RawResultInterface $rawResult, array $options = []): ?TokenUsageInterface`
1. **类型检查**：验证底层响应对象是否实现 `ResponseInterface`，否则返回 `null`。
2. **速率限制头提取**：读取 HTTP 响应头：
   - `x-ratelimit-limit-tokens-minute`：每分钟 Token 限额
   - `x-ratelimit-limit-tokens-month`：每月 Token 限额
3. **响应体用量提取**：从 `usage.prompt_tokens` 和 `usage.total_tokens` 字段获取实际用量。
4. 返回 `TokenUsage` 对象，包含上述所有信息（各字段均允许 `null`，使用 `??` 运算符安全取值）。

## 设计特点

- 嵌入 API 仅提取 `promptTokens`（无 `completionTokens`，因为嵌入不生成文本）。
- 速率限制信息以整数形式存储（字符串头值需强制类型转换）。

## 关联关系

- 实现 `TokenUsageExtractorInterface`。
- 由 `Embeddings\ResultConverter::getTokenUsageExtractor()` 创建并返回。
- 与 `Llm\TokenUsageExtractor` 设计相似，但后者还提取 `completionTokens`。
