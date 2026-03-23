# Output/ClassificationResult.php 分析

## 概述

`ClassificationResult` 是表示分类任务结果的值对象集合，包含一组 `Classification` 对象（每个对象对应一个候选标签及其置信分数），用于音频分类和图像分类任务的结果封装。

## 关键方法分析

### `fromArray(array $data): self`（静态工厂）
将 API 返回的原始数组（格式为 `[{label: string, score: float}, ...]`）映射为 `Classification[]`，通过 `array_map` 批量构造。

## 关键模式

- **静态工厂方法**：`fromArray()` 是所有 Output 类统一的构造入口，隔离 API 响应格式与业务模型。
- **PHPDoc 数组形状注解**：`@param array<array{label: string, score: float}>` 精确描述输入格式，提供 IDE 类型提示支持。

## 关联关系

- 由 `ResultConverter::convert()` 在 `AUDIO_CLASSIFICATION` 和 `IMAGE_CLASSIFICATION` 任务时实例化，作为 `ObjectResult` 的载荷。
- 内部持有 `Classification[]` 数组。
