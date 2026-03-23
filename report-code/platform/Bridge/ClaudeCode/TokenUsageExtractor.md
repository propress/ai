# TokenUsageExtractor.php

**命名空间**：`Symfony\AI\Platform\Bridge\ClaudeCode`

## 概述

`TokenUsageExtractor` 实现 `TokenUsageExtractorInterface`，从 Claude Code CLI 返回的原始结果中解析 Token 用量数据，支持提取输入 Token、输出 Token 以及缓存创建和读取 Token。

## 关键方法

- `extract(RawResultInterface $rawResult, array $options): ?TokenUsageInterface` — 流式模式下返回 `null`；否则读取 `getData()['usage']` 字段，构建并返回 `TokenUsage` 对象（含 `input_tokens`、`output_tokens`、缓存 Token 之和）。

## 设计模式

- **可选提取**：未包含 `usage` 字段时返回 `null`，而非抛出异常，符合 API 响应不总包含用量数据的实际情况。

## 关联关系

- 实现 `TokenUsageExtractorInterface`。
- 由 `ResultConverter::getTokenUsageExtractor()` 返回并注册到平台。
- 依赖 `TokenUsage` 值对象封装结果。
