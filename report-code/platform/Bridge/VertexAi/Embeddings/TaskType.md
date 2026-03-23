# Embeddings/TaskType.php 分析报告

## 文件概述

`VertexAi\Embeddings\TaskType` 枚举定义了 VertexAi 嵌入 API 支持的任务类型常量，并通过注释说明了各类型的适用场景，`ModelClient` 将 `RETRIEVAL_QUERY` 作为默认值。

## 枚举值说明

| 常量 | 值 | 用途 |
|------|----|------|
| `CLASSIFICATION` | `CLASSIFICATION` | 按预设标签分类文本 |
| `CLUSTERING` | `CLUSTERING` | 基于相似度聚类文本 |
| `RETRIEVAL_DOCUMENT` | `RETRIEVAL_DOCUMENT` | 被检索语料库中的文档 |
| `RETRIEVAL_QUERY` | `RETRIEVAL_QUERY` | 检索查询（**默认值**） |
| `QUESTION_ANSWERING` | `QUESTION_ANSWERING` | 问答场景 |
| `FACT_VERIFICATION` | `FACT_VERIFICATION` | 事实核查 |
| `CODE_RETRIEVAL_QUERY` | `CODE_RETRIEVAL_QUERY` | 代码检索（文档侧用 RETRIEVAL_DOCUMENT） |
| `SEMANTIC_SIMILARITY` | `SEMANTIC_SIMILARITY` | 语义相似度（非检索场景） |

## 与 Gemini Bridge TaskType 的差异

| 方面 | Gemini Bridge | VertexAi |
|------|--------------|---------|
| `TaskTypeUnspecified` | ✅ 有 | ❌ 无 |
| 默认值使用 | 无默认值（API 自选） | `RETRIEVAL_QUERY`（ModelClient 硬编码） |
| 注释详细程度 | 简短 | 详细说明用法和注意事项 |
| 枚举 case 数量 | 9 个 | 8 个 |

## 设计特点

- 与 Gemini Bridge 的 `TaskType` 相同：所有值通过 `public const` 定义，而非 `case`
- 零 case 枚举（仅常量），无法使用 `::from()` 或 `::cases()` 进行枚举操作
- `SEMANTIC_SIMILARITY` 注释明确指出"不适用于检索场景"

## 关联文件

- `Embeddings/ModelClient.php` — 将 `TaskType::RETRIEVAL_QUERY` 作为 `task_type` 的默认值
