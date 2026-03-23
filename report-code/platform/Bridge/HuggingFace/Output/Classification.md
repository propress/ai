# Output/Classification.php 分析

## 概述

`Classification` 是表示单个分类预测结果的不可变值对象，持有标签名称（`label`）和置信分数（`score`），用作 `ClassificationResult` 的元素类型，适用于音频分类（`AUDIO_CLASSIFICATION`）和图像分类（`IMAGE_CLASSIFICATION`）任务。

## 关键方法分析

构造函数接受两个 `readonly` 属性，无额外方法。实例创建后内容不可变。

## 关键模式

- **不可变值对象**：所有属性均为 `readonly`，通过构造函数一次性赋值。
- **最小化设计**：仅封装 API 响应中最核心的两个字段，不引入额外逻辑。

## 关联关系

- 被 `ClassificationResult::fromArray()` 批量实例化。
- 通过 `ClassificationResult` 包裹后作为 `ObjectResult` 的负载返回给调用方。
