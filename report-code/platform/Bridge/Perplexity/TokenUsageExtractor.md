# TokenUsageExtractor.php 分析

## 概述

`TokenUsageExtractor` 实现 `TokenUsageExtractorInterface`，从 Perplexity API 的同步响应中提取 Token 用量信息，支持普通 Token（prompt/completion）和推理 Token（`reasoning_tokens`，对应 sonar-reasoning 系列模型）。

## 关键方法分析

### `extract(RawResultInterface $rawResult, array $options): ?TokenUsageInterface`
- **流式模式**（`$options['stream']` 为 true）：返回 `null`，流式场景的 Token 用量需要调用方手动从流数据中处理
- **同步模式**：检查响应数据中是否存在 `usage` 键，不存在则返回 `null`；存在则构建 `TokenUsage` 对象，其中 `reasoning_tokens` 映射到 `TokenUsage::$thinkingTokens`

## 关键模式

- **thinking tokens 支持**：将 Perplexity 的 `reasoning_tokens` 映射到平台层的 `thinkingTokens` 字段，为 sonar-reasoning 系列模型提供推理 Token 可见性。
- **防御性 `null` 返回**：对流式模式和缺少 `usage` 字段的响应均返回 `null`，避免在不可用时抛出异常。

## 关联关系

- 由 `ResultConverter::getTokenUsageExtractor()` 返回，被 Platform 基础设施在每次请求后调用以记录用量。
- 对应 Perplexity API 文档中 `usage.prompt_tokens`、`usage.completion_tokens`、`usage.reasoning_tokens`、`usage.total_tokens` 字段。
