# Embeddings/TaskType.php 分析报告

## 文件概述

`TaskType` 枚举定义了 Gemini 嵌入 API 支持的任务类型常量，用于指导模型生成针对特定场景优化的嵌入向量。

## 枚举值说明

| 常量 | 值 | 用途 |
|------|----|------|
| `TaskTypeUnspecified` | `TASK_TYPE_UNSPECIFIED` | 默认（由 API 自动选择） |
| `RetrievalQuery` | `RETRIEVAL_QUERY` | 检索查询文本 |
| `RetrievalDocument` | `RETRIEVAL_DOCUMENT` | 被检索的文档 |
| `SemanticSimilarity` | `SEMANTIC_SIMILARITY` | 语义相似度评估 |
| `Classification` | `CLASSIFICATION` | 文本分类 |
| `Clustering` | `CLUSTERING` | 文本聚类 |
| `QuestionAnswering` | `QUESTION_ANSWERING` | 问答场景 |
| `FactVerification` | `FACT_VERIFICATION` | 事实核查 |
| `CodeRetrievalQuery` | `CODE_RETRIEVAL_QUERY` | 代码检索 |

## 设计特点

- 使用 PHP `enum` 定义（`string` backed enum），但所有值通过 `public const` 而非 `case` 声明
- 这意味着此枚举实际上是零 case 的枚举（只有常量），无法使用 `TaskType::from()` 或 `TaskType::cases()`
- 与 VertexAi 的 `TaskType` 类似但有差异：Gemini 版本多 `TaskTypeUnspecified`；VertexAi 默认值为 `RETRIEVAL_QUERY`

## 关联文件

- `Embeddings.php` — `options['task_type']` 接受 `TaskType|string` 类型
- `Embeddings/ModelClient.php` — 读取 `task_type` 选项并传递给 API 的 `taskType` 字段
