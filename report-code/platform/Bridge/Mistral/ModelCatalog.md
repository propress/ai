# Mistral/ModelCatalog.php

## 概述

`ModelCatalog` 是 Mistral Bridge 的模型能力注册表，继承自 `AbstractModelCatalog`，注册了 Mistral AI 全系列模型及其各自支持的能力，涵盖对话、代码、视觉、音频和嵌入模型。

## 已注册模型统计

| 模型系列 | 代表模型 | 典型能力 |
|---------|---------|---------|
| 大型对话模型 | `mistral-large-latest`、`codestral-latest` | `INPUT_MESSAGES`、`OUTPUT_TEXT`、`OUTPUT_STREAMING`、`OUTPUT_STRUCTURED`、`TOOL_CALLING` |
| 视觉模型（Pixtral 系列） | `pixtral-large-latest`、`pixtral-12b-latest` | 同上 + `INPUT_IMAGE` |
| 中小型模型 | `mistral-small-latest`、`ministral-8b-latest` | 部分含 `INPUT_IMAGE` |
| 代码/开发模型 | `devstral-medium-latest`、`devstral-small-latest` | 无图像能力 |
| 音频模型 | `voxtral-small-latest`、`voxtral-mini-latest` | `INPUT_AUDIO`（替代 `INPUT_IMAGE`） |
| 嵌入模型 | `mistral-embed` | `INPUT_MULTIPLE`、`EMBEDDINGS` |
| 地区/语言专用 | `mistral-saba-latest`（Arabic/South Asian） | 无工具调用 |

总计约 16 个预置模型，均支持通过 `$additionalModels` 参数追加自定义模型。

## 设计模式

- **注册表模式**：集中管理模型元数据。
- **开放扩展**：构造函数参数 `$additionalModels` 通过 `array_merge` 合并，覆盖时以用户定义的为准。

## 关联关系

- 继承 `Platform\ModelCatalog\AbstractModelCatalog`。
- 引用两个模型类：`Mistral::class`（对话模型）和 `Embeddings::class`（嵌入模型）。
- 由 `PlatformFactory::create()` 作为默认目录参数使用。
