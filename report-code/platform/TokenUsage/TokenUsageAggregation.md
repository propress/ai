# TokenUsageAggregation 分析报告

## 文件概述
`TokenUsageAggregation` 聚合多次 AI 调用的 Token 使用量，实现 `TokenUsageInterface` 和 `MergeableMetadataInterface`，通过 sum/min 操作提供汇总统计。

## 类定义
- **类型**: `final class`，实现 `TokenUsageInterface`、`MergeableMetadataInterface`

## 方法分析

### `add(TokenUsageInterface $tokenUsage): void`
- 追加一个使用量记录到内部数组

### `merge(MergeableMetadataInterface $metadata): self`
- 合并产生新的 `TokenUsageAggregation`（包含原有 + 新增），不修改原对象（不可变语义）

### `count(): int`
- 递归统计包含的使用量记录数（包括嵌套 Aggregation）

### `getPromptTokens() / getCompletionTokens() / ...` — 使用 `sum()` 汇总
### `getRemainingTokens() / getRemainingTokensMinute() / getRemainingTokensMonth()` — 使用 `min()` 汇总（剩余量取最小值，最保守估计）

### `sum(\Closure $mapFunction): ?int` / `min(\Closure $mapFunction): ?int` *(private)*
- `array_filter(array_map($fn, $usages))` 过滤 null，空数组返回 null（避免虚假的 0）
- **技巧**：返回 null 而非 0，区分"不支持此字段"和"真实值为 0"两种情况

## 设计模式
**组合模式（Composite）**：聚合类自身也是 `TokenUsageInterface`，支持嵌套聚合（`TokenUsageAggregation` 内可以包含另一个 `TokenUsageAggregation`）。

## 与其他文件的关系
- 由 `TokenUsage::merge()` 创建
- 被 `Metadata::add()` 通过 `MergeableMetadataInterface` 自动触发合并
- 在多轮对话或批量调用场景中自动积累
