# Bedrock/Nova/Nova.php

## 概述

`Nova` 是 Amazon Nova 系列模型的标识类，继承自平台基类 `Model`，本身不添加任何额外逻辑，仅用于类型识别。

## 设计特点

- **标记类（Marker Class）模式**：空类体，仅通过类型继承提供标识语义。所有 Nova 专用的规范化器和客户端均通过 `$model instanceof Nova` 进行分发判断。
- 声明为 `final`，不可被继承（与 `LlamaModelClient` 等非 `final` 类形成对比）。
- 与 `Bridge/Anthropic/Claude`、`Bridge/Meta/Llama` 的设计模式一致。

## 关联关系

- 继承 `Platform\Model`，获得模型名称（`getName()`）和能力（`supports(Capability)`）管理。
- 被 `NovaModelClient`、`NovaResultConverter` 及全部 Nova Contract 规范化器用于 `supports()` / `supportsModel()` 的类型检查。
- 在 `ModelCatalog` 中，所有 Nova 模型条目的 `class` 字段均指向此类。
