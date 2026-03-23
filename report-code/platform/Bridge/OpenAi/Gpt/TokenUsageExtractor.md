# OpenAi/Gpt/TokenUsageExtractor 分析报告

## 文件概述
从 OpenAI Responses API 格式提取 Token 使用量，支持非流式（从 HTTP 响应数据）和辅助流式（从数据帧中手动调用）。

## 字段映射（Responses API 格式）
```
usage.input_tokens                        → TokenUsage::promptTokens
usage.output_tokens                       → TokenUsage::completionTokens
usage.output_tokens_details.reasoning_tokens → TokenUsage::thinkingTokens（思维链）
usage.input_tokens_details.cached_tokens  → TokenUsage::cachedTokens
x-ratelimit-remaining-tokens（HTTP header）→ TokenUsage::remainingTokens
usage.total_tokens                        → TokenUsage::totalTokens
```

**差异**：使用 `input_tokens`/`output_tokens`（Responses API）而非旧的 `prompt_tokens`/`completion_tokens`（Chat Completions API）。支持 `reasoning_tokens`（o3/o3-mini 的思维链 Token）。

### `fromDataArray(array $data, ?string $remainingTokens): TokenUsage` *(public)*
供流式模式调用（`ResultConverter::convertStream()` 手动调用此方法）。
