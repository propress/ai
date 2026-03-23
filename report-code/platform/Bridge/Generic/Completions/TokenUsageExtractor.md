# Generic/Completions/TokenUsageExtractor 分析报告

## 文件概述
从 OpenAI 格式响应的 `usage` 字段提取 Token 使用量，流式模式返回 null（流式 usage 在各 Bridge 的 StreamListener 中处理）。

## 提取字段映射
```
usage.prompt_tokens       → TokenUsage::promptTokens
usage.completion_tokens   → TokenUsage::completionTokens  
usage.num_cached_tokens   → TokenUsage::cachedTokens
usage.total_tokens        → TokenUsage::totalTokens
```

**注意**：不提取 `thinking_tokens`，因为通用 Completions 格式不包含此字段（仅 OpenAI Responses API 和 Anthropic 有）。

## 与其他文件的关系
- 被 `ResultConverter::getTokenUsageExtractor()` 返回
- `Platform` 在 `DeferredResult` 解析时自动调用
