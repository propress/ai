# Output/TokenClassificationResult.php 分析

## 概述

`TokenClassificationResult` 是 `TOKEN_CLASSIFICATION` 任务（命名实体识别、词性标注等）的结果集合，包含一组 `Token` 对象，每个对象代表识别出的一个实体/词性标注结果。

## 关键方法分析

### `fromArray(array $data): self`（静态工厂）
将 API 响应数组（格式 `[{entity_group, score, word, start, end}, ...]`）映射为 `Token[]`，将蛇形命名的 `entity_group` 字段传入 `Token` 的 `$entityGroup` 参数。

## 关键模式

- **字段名映射**：在 `fromArray()` 中处理 API 原始字段名（`entity_group`）到 PHP 属性名（`entityGroup`）的转换，与 PHPDoc 形状注解 `@param array<array{entity_group: string, ...}>` 严格对应。

## 关联关系

- 由 `ResultConverter::convert()` 在 `TOKEN_CLASSIFICATION` 任务时实例化，封装为 `ObjectResult`。
- 内部元素类型为 `Token`。
