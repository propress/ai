# TokenUsageInterface 分析报告

## 文件概述
`TokenUsageInterface` 定义 Token 使用量数据的完整访问契约，覆盖 OpenAI、Anthropic 等主流提供商的所有 Token 类型字段。

## 方法一览（均返回 `?int`，未知时返回 null）

| 方法 | 说明 |
|---|---|
| `getPromptTokens()` | 输入（Prompt）消耗的 Token 数 |
| `getCompletionTokens()` | 输出（Completion）消耗的 Token 数 |
| `getThinkingTokens()` | 推理（Thinking）消耗的 Token 数（Claude 3.7+ Extended Thinking） |
| `getToolTokens()` | 工具调用相关 Token |
| `getCachedTokens()` | 命中缓存的 Token 数 |
| `getCacheCreationTokens()` | 写入 Prompt Cache 的 Token 数（Anthropic） |
| `getCacheReadTokens()` | 从 Prompt Cache 读取的 Token 数（Anthropic） |
| `getRemainingTokens()` | 剩余可用 Token |
| `getRemainingTokensMinute()` | 本分钟剩余 Token |
| `getRemainingTokensMonth()` | 本月剩余 Token |
| `getTotalTokens()` | 总 Token 数 |

## 设计模式
**接口隔离（ISP）+ 最大兼容**：所有字段都是 nullable，允许各提供商只报告自己支持的字段，实现类可以选择性实现（对不支持的字段返回 null）。

## 与其他文件的关系
- 实现类：`TokenUsage`（单次调用）、`TokenUsageAggregation`（多次汇总）
- 被 `TokenUsageExtractorInterface::extract()` 返回
- 被 `Metadata::add()` 存储（通过 `MergeableMetadataInterface` 自动合并）
