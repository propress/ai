# Output/DetectedObject.php 分析

## 概述

`DetectedObject` 是目标检测任务中单个检测对象的不可变值对象，包含类别标签、置信分数以及边界框的四个坐标（xmin、ymin、xmax、ymax）。

## 关键方法分析

构造函数接受六个 `readonly` 属性，所有坐标以 `float` 类型存储（HuggingFace 返回的坐标为整数像素值或归一化浮点数，统一以 float 兼容）。

## 关键模式

- **扁平化坐标**：将 `box: {xmin, ymin, xmax, ymax}` 中的嵌套坐标展开为四个独立属性，在 `ObjectDetectionResult::fromArray()` 中通过 `$item['box']['xmin']` 等方式提取。
- **不可变值对象**：全部属性为 `readonly`，符合值对象语义。

## 关联关系

- 被 `ObjectDetectionResult::fromArray()` 批量实例化，作为 `ObjectDetectionResult::$objects` 数组的元素。
