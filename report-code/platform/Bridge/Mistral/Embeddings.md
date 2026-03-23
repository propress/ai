# Mistral/Embeddings.php

## 概述

`Embeddings` 是 Mistral 嵌入模型的标识类，继承自平台基类 `Model`，采用与 `Mistral`（对话模型）相同的标记类模式，用于区分嵌入模型与对话模型的请求路由。

## 设计特点

- **标记类（Marker Class）模式**：空类体，仅提供类型标识。
- 声明为 `final`，不可被继承。
- 命名空间位于 `Symfony\AI\Platform\Bridge\Mistral`（顶层），而嵌入的 `ModelClient` 和 `ResultConverter` 位于子命名空间 `Bridge\Mistral\Embeddings\`，类名冲突通过命名空间区分。

## 关联关系

- 继承 `Platform\Model`。
- 被 `Embeddings\ModelClient` 和 `Embeddings\ResultConverter` 通过 `$model instanceof Embeddings` 进行类型检查。
- 在 `ModelCatalog` 中，`mistral-embed` 模型的 `class` 字段指向此类，能力为 `[INPUT_MULTIPLE, EMBEDDINGS]`。
