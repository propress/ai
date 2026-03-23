# Anthropic/TokenUsageExtractor 分析报告

## 提取字段映射（Anthropic 格式）
```
usage.input_tokens                    → promptTokens
usage.output_tokens                   → completionTokens
usage.server_tool_use.web_search_requests → toolTokens
usage.cache_creation_input_tokens     → cacheCreationTokens（Anthropic 专有）
usage.cache_read_input_tokens         → cacheReadTokens（Anthropic 专有）
cacheCreation + cacheRead             → cachedTokens（合计，便于只需总数的消费者）
```

**Anthropic 专有**：`cache_creation_input_tokens` 和 `cache_read_input_tokens` 分别计量 Prompt Cache 写入和读取的 Token 数，可用于精确计算缓存节省的费用（读取 token 通常比写入便宜 90%）。
