# Output/ImageSegmentationResult.php 分析

## 概述

`ImageSegmentationResult` 是 `IMAGE_SEGMENTATION` 任务的结果集合，包含一组 `ImageSegment` 对象，每个对象代表图像中的一个分割区域（含标签、分数、掩码）。

## 关键方法分析

### `fromArray(array $data): self`（静态工厂）
将 API 响应数组（格式 `[{label: string, score: float, mask: string}, ...]`）映射为 `ImageSegment[]`。注意 `score` 可能为 `null`（与 `ImageSegment` 的 `?float $score` 对应）。

## 关键模式

- **与 `ClassificationResult` 相同的结构模式**：集合类 + 单体类的二层设计，集合类持有单体数组，单体类持有具体字段。
- **PHPDoc 形状注解**：标注了三字段的精确类型（`label: string, score: float, mask: string`）。

## 关联关系

- 由 `ResultConverter::convert()` 在 `IMAGE_SEGMENTATION` 任务时实例化，封装为 `ObjectResult`。
- 内部元素类型为 `ImageSegment`。
