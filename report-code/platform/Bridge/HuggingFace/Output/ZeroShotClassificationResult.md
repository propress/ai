# Output/ZeroShotClassificationResult.php 分析

## 概述

`ZeroShotClassificationResult` 是 `ZERO_SHOT_CLASSIFICATION` 任务的结果值对象，包含标签列表、对应的置信分数列表和可选的原始输入序列文本，兼容两种不同的 API 响应格式（HF Inference Serverless 格式和标准 HF Inference 格式）。

## 关键方法分析

### `fromArray(array $data): self`（静态工厂）
实现了两种响应格式的兼容逻辑：
- **Serverless 格式**：`[{label: string, score: float}, ...]`（检测 `$data[0]['label']` 是否存在），需展开标签和分数到独立数组
- **标准格式**：`{labels: string[], scores: float[], sequence?: string}`，直接映射

## 关键模式

- **双格式兼容**：在同一 `fromArray()` 方法中通过条件分支处理两种 API 响应结构，避免外部调用者感知格式差异。这体现了统一化适配的设计思路。
- **可选 `$sequence`**：标准格式包含原始输入序列，Serverless 格式不返回该字段，通过默认 `null` 兼容。

## 关联关系

- 由 `ResultConverter::convert()` 在 `ZERO_SHOT_CLASSIFICATION` 任务时实例化，封装为 `ObjectResult`。
- 是 Output 类中逻辑最复杂的一个，因为需要兼容两种不同的 HuggingFace 响应格式。
