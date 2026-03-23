# Voyage/ModelCatalog.php

## 概述

`ModelCatalog` 是 Voyage Bridge 的模型能力注册表，注册了 Voyage AI 平台提供的所有嵌入模型，包括通用嵌入、领域专用嵌入以及唯一支持多模态输入的 `voyage-multimodal-3`。

## 已注册模型统计

| 模型 | 能力 | 说明 |
|------|------|------|
| `voyage-3.5` | `INPUT_MULTIPLE`、`EMBEDDINGS` | 最新通用嵌入模型 |
| `voyage-3.5-lite` | 同上 | 轻量版 |
| `voyage-3` | 同上 | 标准通用模型 |
| `voyage-3-lite` | 同上 | 轻量版 |
| `voyage-3-large` | 同上 | 高维度版本 |
| `voyage-finance-2` | 同上 | 金融领域专用 |
| `voyage-multilingual-2` | 同上 | 多语言专用 |
| `voyage-law-2` | 同上 | 法律领域专用 |
| `voyage-code-3` | 同上 | 代码专用（最新） |
| `voyage-code-2` | 同上 | 代码专用 |
| `voyage-multimodal-3` | `INPUT_MULTIPLE`、**`INPUT_MULTIMODAL`**、`EMBEDDINGS` | 唯一多模态模型 |

共 11 个预置模型。`voyage-multimodal-3` 是唯一带有 `INPUT_MULTIMODAL` 能力的条目，触发多模态规范化逻辑。

## 设计模式

- **注册表模式**：通过能力标志区分普通嵌入与多模态嵌入，驱动 `ModelClient` 的端点选择和 Contract 的规范化器选择。
- 支持 `$additionalModels` 扩展参数。

## 关联关系

- 继承 `Platform\ModelCatalog\AbstractModelCatalog`。
- 所有模型均使用 `Voyage::class` 作为模型类。
- `INPUT_MULTIMODAL` 能力是 `ModelClient` 中端点分支和所有多模态规范化器 `supportsModel()` 判断的依据。
