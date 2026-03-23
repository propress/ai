# TokenUsage 分析报告

## 文件概述
`TokenUsage` 是单次 AI 调用的 Token 使用量值对象，实现 `TokenUsageInterface` 和 `MergeableMetadataInterface`，当有多次调用时可通过 `merge()` 自动合并为 `TokenUsageAggregation`。

## 类定义
- **类型**: `final class`，实现 `MergeableMetadataInterface`、`TokenUsageInterface`

## 构造函数
所有参数均可选（`?int = null`），允许只填写提供商实际返回的字段。

## `merge(MergeableMetadataInterface $metadata): TokenUsageAggregation`
合并逻辑：将 `$this` 和 `$metadata` 组合为 `TokenUsageAggregation`（不是相加，而是委托给聚合类汇总）。

**为什么不直接相加**：直接相加丢失各次调用的细节；`TokenUsageAggregation` 保留所有历史记录，支持更复杂的统计（如 min/max/count）。

## 与其他文件的关系
- 由 Bridge 的 `TokenUsageExtractor` 创建
- 存储在 `Result.getMetadata()['token_usage']` 中
- 合并产生 `TokenUsageAggregation`
