# Output/FillMaskResult.php 分析

## 概述

`FillMaskResult` 是 `FILL_MASK`（掩码语言模型填词）任务的结果集合，包含一组 `MaskFill` 候选填词对象，每个候选包含 token ID、token 字符串、完整预测序列和置信分数。

## 关键方法分析

### `fromArray(array $data): self`（静态工厂）
将 API 响应数组（格式 `[{token: int, token_str: string, sequence: string, score: float}, ...]`）批量映射为 `MaskFill[]`，按置信分数降序排列（由 API 保证，非本类处理）。

## 关键模式

- **静态工厂方法**：与其他 Output 类一致，通过 `fromArray()` 构造实例。
- **精确的 PHPDoc 形状注解**：入参数组形状标注了 4 个字段的具体类型。

## 关联关系

- 由 `ResultConverter::convert()` 在 `FILL_MASK` 任务时实例化。
- 内部持有 `MaskFill[]` 数组，`MaskFill` 是单个填词候选的值对象。
