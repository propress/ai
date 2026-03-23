# ResultConverter.php 分析报告

## 概述

`ResultConverter` 根据请求时指定的 `task` 选项，将 Pipeline 推理结果转换为平台统一的结果类型（`TextResult`、`VectorResult` 或 `ObjectResult`），支持文本生成、嵌入向量提取和通用任务输出。

## 关键方法分析

### `supports(Model $model): bool`
始终返回 `true`，与 `ModelClient::supports()` 保持一致，接受所有模型类型。

### `convert(RawResultInterface $result, array $options = []): ResultInterface`
根据 `$options['task']` 分派结果处理：

| Task 类型 | 处理逻辑 | 返回类型 |
|-----------|----------|----------|
| `Task::Text2TextGeneration` | 取第一个结果的 `generated_text` 字段 | `TextResult` |
| `Task::Embeddings` / `Task::FeatureExtraction` | 将每个向量数组映射为 `Vector` 对象 | `VectorResult` |
| 其他任务 | 原样返回完整数组数据 | `ObjectResult` |

### `getTokenUsageExtractor(): null`
返回 `null`，本地推理无 Token 用量概念。

## 设计模式

- **策略模式（Strategy）**：通过 `$options['task']` 动态选择转换策略，避免大量条件分支
- **开放扩展**：未匹配的 Task 类型回退到 `ObjectResult`，保持向前兼容性

## 与其他类的关系

- 消费 `RawPipelineResult::getData()` 返回的数组数据
- 使用 `Codewithkyrian\Transformers\Pipelines\Task` 枚举进行任务类型匹配
- 返回 `TextResult`、`VectorResult`（含 `Vector[]`）或 `ObjectResult`
