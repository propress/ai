# TokenUsageExtractor.php 分析

## 概述

`TokenUsageExtractor` 实现 `TokenUsageExtractorInterface`，从 Ollama API 的非流式响应中提取 Token 用量信息，使用 Ollama 特有的 `prompt_eval_count`（prompt tokens）和 `eval_count`（completion tokens）字段。

## 关键方法分析

### `extract(RawResultInterface $rawResult, array $options): ?TokenUsageInterface`
- **流式模式**（`$options['stream']` 为 true）：返回 `null`，流式场景的 Token 统计在最后一个 `done=true` 的块中，需调用方手动从 `OllamaMessageChunk::$raw` 中提取
- **非流式模式**：检查响应数据中是否同时存在 `prompt_eval_count` 和 `eval_count`，若缺少则返回 `null`；否则构建 `TokenUsage` 对象

## 关键模式

- **Ollama 特有字段名**：Ollama 使用 `prompt_eval_count`/`eval_count` 而非 OpenAI 风格的 `prompt_tokens`/`completion_tokens`，此提取器封装了字段名差异。
- **防御性检查**：同时检查两个字段的存在性（`isset($payload['prompt_eval_count'], $payload['eval_count'])`），避免部分数据下的构建失败。

## 关联关系

- 由 `OllamaResultConverter::getTokenUsageExtractor()` 返回，被 Platform 基础设施在每次请求后调用。
- 嵌入请求响应中不包含这两个字段，提取器将返回 `null`。
