# Mistral/Mistral.php

## 概述

`Mistral` 是 Mistral 对话（LLM）模型的标识类，继承自平台基类 `Model`，采用标记类模式用于类型分发，与 `Embeddings` 类配合区分两类不同的 Mistral 模型。

## 设计特点

- **标记类（Marker Class）模式**：空类体，仅提供类型标识。
- 声明为 `final`，不可被继承。
- 命名为 `Mistral`（与 Bridge 命名空间同名），代表 Mistral AI 平台的对话模型系列（包括 Codestral、Devstral、Pixtral 等不同功能的模型，均使用此同一类型）。

## 关联关系

- 继承 `Platform\Model`，获得模型名称和能力管理。
- 被 `Llm\ModelClient` 和 `Llm\ResultConverter` 用于 `supports()` 类型检查。
- 在 `ModelCatalog` 中，所有对话/代码/视觉/音频模型（除 `mistral-embed` 外）均使用此类。
- 与同命名空间的 `Embeddings` 类共同构成 Mistral Bridge 的模型类型体系。
