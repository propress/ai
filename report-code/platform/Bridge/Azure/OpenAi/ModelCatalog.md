# `Azure/OpenAi/ModelCatalog.php` 分析

## 概述

Azure OpenAI 专用的模型目录，继承 `AbstractModelCatalog`，在 OpenAI 标准模型目录基础上将 `Gpt::class` 替换为 `ResponsesModel::class`、将 `Embeddings::class` 替换为 `EmbeddingsModel::class`，以匹配 Azure 端点所需的模型类型。

## 关键方法

| 方法 | 说明 |
|------|------|
| `__construct(additionalModels)` | 读取 `OpenAiModelCatalog` 的全部模型，替换类映射后与 `additionalModels` 合并 |

## 设计要点

- 复用 `Bridge\OpenAi\ModelCatalog` 中的模型列表，避免重复维护模型能力定义
- 仅替换模型类（`class` 字段），保留所有 `capabilities` 不变
- 支持通过 `additionalModels` 参数扩展自定义模型

## 关系

- 继承：`AbstractModelCatalog`
- 依赖：`Bridge\OpenAi\ModelCatalog`（获取基础模型列表）
- 关联类替换：`Gpt` → `ResponsesModel`，`Embeddings` → `EmbeddingsModel`
- 被 `Azure\OpenAi\PlatformFactory` 使用（作为默认目录）
