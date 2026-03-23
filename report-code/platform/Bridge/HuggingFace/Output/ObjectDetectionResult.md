# Output/ObjectDetectionResult.php 分析

## 概述

`ObjectDetectionResult` 是 `OBJECT_DETECTION` 任务的结果集合，包含一组 `DetectedObject` 对象，每个对象包含检测到的物体类别、置信分数和边界框坐标。

## 关键方法分析

### `fromArray(array $data): self`（静态工厂）
将 API 响应数组（格式 `[{label, score, box: {xmin, ymin, xmax, ymax}}, ...]`）映射为 `DetectedObject[]`，在构造 `DetectedObject` 时将嵌套的 `box` 对象展开为六个独立参数。

## 关键模式

- **嵌套展开**：`fromArray()` 负责将 API 响应的 `box` 嵌套对象展平，使 `DetectedObject` 的使用者直接通过属性访问坐标（而无需 `$obj->box['xmin']`）。
- **精确 PHPDoc 形状**：入参数组形状清晰记录 `box` 的嵌套结构。

## 关联关系

- 由 `ResultConverter::convert()` 在 `OBJECT_DETECTION` 任务时实例化，作为 `ObjectResult` 载荷。
- 内部元素类型为 `DetectedObject`。
