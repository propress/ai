# Bedrock/ModelCatalog.php

## 概述

`ModelCatalog` 是 Bedrock Bridge 的模型能力注册表，继承自 `AbstractModelCatalog`，集中声明所有受支持的 Nova、Claude、Llama 模型及其各自的能力集合（`Capability`）。

## 关键方法分析

### `__construct(array $additionalModels = [])`
合并默认模型列表与用户自定义追加的模型列表。每个模型条目包含：
- `class`：对应的模型类（`Nova::class`、`Claude::class` 或 `Llama::class`）。
- `capabilities`：该模型支持的能力枚举列表。

## 已注册模型统计

| 模型系列 | 数量 | 典型能力 |
|---------|------|---------|
| Nova（Amazon）| 4 个（micro/lite/pro/premier） | `INPUT_MESSAGES`、`OUTPUT_TEXT`，部分支持 `TOOL_CALLING`、`OUTPUT_STRUCTURED`、`INPUT_IMAGE` |
| Claude（Anthropic）| 16+ 个 | `INPUT_MESSAGES`、`INPUT_IMAGE`、`OUTPUT_TEXT`、`OUTPUT_STREAMING`、`TOOL_CALLING`，新版本还支持 `OUTPUT_STRUCTURED` |
| Llama（Meta）| 12 个 | 仅 `INPUT_MESSAGES` + `OUTPUT_TEXT` |

## 设计模式

- **注册表模式**：通过数组集中管理模型元数据，而非分散到各模型类中。
- **开放/封闭原则**：通过 `$additionalModels` 构造函数参数支持扩展，无需修改现有代码。

## 关联关系

- 继承 `Platform\ModelCatalog\AbstractModelCatalog`，将模型数组存储在 `$this->models` 属性中。
- 被 `PlatformFactory::create()` 作为默认目录参数使用。
- 引用了三个不同 Bridge 命名空间的模型类：`Bridge/Bedrock/Nova/Nova`、`Bridge/Anthropic/Claude`、`Bridge/Meta/Llama`。
