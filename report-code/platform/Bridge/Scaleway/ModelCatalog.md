# `Scaleway/ModelCatalog.php` 分析

## 概述

Scaleway Bridge 的模型目录，继承 `AbstractModelCatalog`，预置 12 个 LLM 模型（`Scaleway::class`）和 2 个嵌入模型（`Embeddings::class`），涵盖 DeepSeek、Gemma、Llama、Mistral、Qwen 等主流开源模型。

## 关键方法

| 方法 | 说明 |
|------|------|
| `__construct(additionalModels)` | 构建默认模型列表，与 `additionalModels` 合并后赋值给 `$this->models` |

## 设计要点

- `pixtral-12b-2409` 额外具备 `INPUT_IMAGE` 能力（多模态）
- `qwen3.5-397b-a17b` 具备 `THINKING`（思维链）能力，其余 LLM 模型能力一致
- 嵌入模型仅具备 `INPUT_TEXT` 和 `EMBEDDINGS` 能力
- 支持通过 `additionalModels` 扩展自定义模型，遵循 `{class, capabilities}` 结构

## 关系

- 继承：`AbstractModelCatalog`
- 注册类型：`Scaleway::class`（LLM）和 `Embeddings::class`（嵌入）
- 被 `PlatformFactory::create()` 作为默认目录参数使用
