# Output/TableQuestionAnsweringResult.php 分析

## 概述

`TableQuestionAnsweringResult` 是 `TABLE_QUESTION_ANSWERING`（TAPAS 类模型）任务的结果值对象，包含自然语言答案、单元格坐标、单元格值和聚合操作类型（如 SUM、COUNT）。

## 关键方法分析

### `fromArray(array $data): self`（静态工厂）
将 API 响应数组解析为对应属性，`coordinates`、`cells`、`aggregator` 均为可选字段（以 `?? []`/`?? null` 处理缺失情况）。

## 关键模式

- **可选字段兼容**：`$coordinates`、`$cells` 默认为空数组，`$aggregator` 默认为 `null`，兼容不同 TAPAS 模型的输出差异（部分模型不返回坐标信息）。
- **复杂类型注解**：`$aggregator` 类型为 `array<string>|string|null`，兼容模型返回聚合操作为字符串或字符串数组的情况。

## 关联关系

- 由 `ResultConverter::convert()` 在 `TABLE_QUESTION_ANSWERING` 任务时实例化，封装为 `ObjectResult`。
