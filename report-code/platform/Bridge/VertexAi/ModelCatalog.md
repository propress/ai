# ModelCatalog.php 分析报告

## 文件概述

`ModelCatalog` 维护 VertexAi Bridge 的预定义模型清单，包含 Gemini 聊天模型和多种嵌入模型（含多语言版本），支持自定义模型注入。

## 关键方法分析

### `__construct(array $additionalModels = [])`

与 Gemini Bridge 的 `ModelCatalog` 结构相同，通过 `array_merge` 支持外部扩展。

## 模型分类

| 类别 | 模型 | 类 | 特殊说明 |
|------|------|-----|---------|
| **Gemini 聊天** | `gemini-3-pro-preview` | `Gemini::class`（来自 Gemini Bridge！） | 遗留条目，可能是 bug |
| **Gemini 聊天** | `gemini-2.5-pro` | `GeminiModel::class` | 标准 VertexAi 模型类 |
| **Gemini 聊天** | `gemini-2.5-flash` | `GeminiModel::class` | |
| **Gemini 聊天** | `gemini-2.0-flash` | `GeminiModel::class` | 无 `OUTPUT_STRUCTURED` |
| **Gemini 聊天** | `gemini-2.5-flash-lite` | `GeminiModel::class` | |
| **Gemini 聊天** | `gemini-2.0-flash-lite` | `GeminiModel::class` | 无 `OUTPUT_STRUCTURED` |
| **嵌入** | `gemini-embedding-001` | `EmbeddingsModel::class` | 含 `INPUT_MULTIPLE` |
| **嵌入** | `text-embedding-005` | `EmbeddingsModel::class` | 含 `INPUT_MULTIPLE` |
| **多语言嵌入** | `text-multilingual-embedding-002` | `EmbeddingsModel::class` | 含 `INPUT_MULTIPLE` |

## 设计特点

- **`gemini-3-pro-preview` 使用 `Gemini::class`（Gemini Bridge 的类）**：这是一个潜在的 bug，该模型条目使用了来自 `Symfony\AI\Platform\Bridge\Gemini\Gemini` 的类，而不是 VertexAi 自己的 `GeminiModel::class`。这可能导致 Contract Normalizer 无法正确匹配（VertexAi Normalizer 检查 `instanceof VertexAi\Gemini\Model`，而非 `instanceof Gemini\Gemini`）。
- **`INPUT_MULTIPLE`**：嵌入模型声明批量输入能力，Gemini Bridge 的嵌入模型不含此能力
- VertexAi 不含 TTS 模型（Gemini Bridge 有 3 款）

## 关联文件

- `PlatformFactory.php` — 默认传入 `new ModelCatalog()` 实例
- `VertexAi\Gemini\Model` — 聊天模型类（`GeminiModel::class`）
- `VertexAi\Embeddings\Model` — 嵌入模型类（`EmbeddingsModel::class`）
