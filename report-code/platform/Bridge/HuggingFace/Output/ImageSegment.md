# Output/ImageSegment.php 分析

## 概述

`ImageSegment` 是图像分割任务中单个分割区域的不可变值对象，包含区域标签、可选置信分数和 Base64 编码的掩码图像字符串。

## 关键方法分析

构造函数接受三个 `readonly` 属性，其中 `$score` 为可空 `float`（部分分割模型不输出置信分数）。

## 关键模式

- **可空分数**：`?float $score` 的设计兼容了不同分割模型的输出差异（语义分割模型通常不输出 per-segment 置信分数）。
- `$mask` 存储 Base64 编码的黑白掩码图像，消费方可将其解码为图像数据。

## 关联关系

- 被 `ImageSegmentationResult::fromArray()` 批量实例化。
- `ImageSegmentationResult` 通过 `ObjectResult` 返回给调用方。
